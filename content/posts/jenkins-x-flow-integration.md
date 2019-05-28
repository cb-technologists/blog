---
author:
  name: "Josh Hendrick"
title: "Extending Jenkins X for Traditional Deployments with CloudBees Flow"
date: 2019-05-28T10:58:46-04:00
showDate: true
draft: true
tags: ["jenkins","jenkins x","cloudbees flow", "tekon", "extending jenkins x"]
---
[Jenkins X](https://jenkins-x.io) is quickly becoming the de facto standard for high performing teams wanting to do CI/CD in a highly scalable and fault tolerant environment. For those who haven’t gotten the opportunity to try out Jenkins X, it allows teams to run CI/CD workloads natively in a Kubernetes environment while taking advantage of modern operating patterns like GitOps and serverless architectures. For teams wanting to modernize their continuous integration and continuous deployment capabilities, Jenkins X is the go to solution.

In today’s heterogenous technology environment, most organizations tend to have a mix of modern cloud native architectures as well as more traditional workloads which get deployed either on-prem or within the cloud. In the latter case, a combination of Jenkins X (performing CI steps) and CloudBees Flow (handling the deployment) can add a huge amount of flexibility and power to a Continuous Delivery process.  Jenkins X can be easily extended to accommodate any type of workload required - it can be a full end to end CI/CD tool for building, deploying, and running applications all within a Kubernetes cluster, or it can handle CI while offloading other release and deployment tasks to another solution.  In this blog post we’re going to cover how Jenkins X can be extended to offload release/deployment tasks to [CloudBees Flow](https://www.cloudbees.com/cloudbees-acquires-electric-cloud).

# Extending Jenkins X
For the purposes of this blog, we’re going to be focusing on the Jenkins X serverless pipeline execution engine with Tekton (See https://jenkins-x.io/architecture/jenkins-x-pipelines/). There are two main ways to customize a Jenkins X pipeline in order to integrate with CloudBees Flow.  The first and simplest would be to modify the jenkins-x.yml (more information on Jenkins X pipelines: https://jenkins-x.io/architecture/jenkins-x-pipelines/#differences-to-jenkins-pipelines and the jenkins-x.yml file) pipeline file in the source code repo for the project we’re going to build.  The other way is to extend the [Jenkins X build packs](https://jenkins-x.io/architecture/build-packs/) and modify the build pack for the language/build tool you want to use.  Both will work, but by forking the build packs you can get reuse across multiple projects using the build pack you extend. In this example, we’ll walk through how to extend the Jenkins X build packs.

## Creating our Cluster and Installing Jenkins X
To start, we’ll fork the Jenkins X Kubernetes build packs into our own repository: https://github.com/jenkins-x-buildpacks/jenkins-x-kubernetes.  Later we'll be extending the maven build pack to support a REST API call into CloudBees Flow.

Then it’s time to create a Kubernetes cluster on GKE using Jenkins X and [Tekton](https://github.com/tektoncd/pipeline): 

```bash
jx create cluster gke --tekton --no-tiller
```

Fill out the options.  For example:

{{< image src="/img/jenkins-x-flow-integration/jx-create-cluster.jpg" alt="Jenkins X create cluster output" >}}

Create an ingress controller if one doesn’t exist and setup the domain or use the default *.nip.io address if you don’t have one.  Go through the prompts and then configure your GitHub credentials.  Create an API token using the URL provided if you don’t have one:

{{< image src="/img/jenkins-x-flow-integration/jx-github.jpg" alt="Jenkins X setup GitHub credentials" >}}

In the setup we’re going to choose the Kubernetes workloads option and later modify the kubernetes workload build packs to include the CloudBees Flow specific steps:

{{< image src="/img/jenkins-x-flow-integration/jx-kubernetes-workload.jpg" alt="Jenkins X Kubernetes Workload" >}}

## Editing the Build Packs
I edited the Jenkins X build packs in VS Code and installed the YAML Language extension (https://jenkins-x.io/architecture/jenkins-x-pipelines/#editing-in-vs-code) for validation as recommended by the Jenkins X team.

This example is going to focus on a sample spring boot application using Maven.  To start I modified the maven build pack in my forked build pack repo (https://github.com/jhendrickCB/jenkins-x-kubernetes/blob/master/packs/maven/pipeline.yaml):

```bash
extends:
 import: classic
 file: maven/pipeline.yaml
pipelines:
 release:
   build:
     steps:
     - sh: jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)
       name: post-build
   promote:
     steps:
     - sh: jx step changelog --version v\$(cat ../../VERSION)
       name: changelog
     - comment: call CloudBees Flow to run a release
       sh: >
         curl -X POST --header "Authorization: Basic $(jx step credential -s flow-token -k token)" --header "Content-Type: application/json" --header "Accept: application/json" -d "{}" "https://ps9.ecloud-kdemo.com/rest/v1.0/pipelines?pipelineName=jhendrick_pipeline&projectName=Training_jhendrick" --insecure
       name: cloudbees-flow-release
```

Compare the original build pack for maven found here: https://github.com/jenkins-x-buildpacks/jenkins-x-kubernetes/tree/master/packs/maven vs. our forked build pack.  We’ve removed all the references to skaffold, watch, and helm since we’re no longer having Jenkin’s X handle the deployment to our Kubernetes cluster.  We’ve also updated the pipeline file to make a call into our CloudBees Flow server:

```bash
curl -X POST --header "Authorization: Basic $(jx step credential -s flow-token -k token)" --header "Content-Type: application/json" --header "Accept: application/json" -d "{}" "https://ps9.ecloud-kdemo.com/rest/v1.0/pipelines?pipelineName=jhendrick_pipeline&projectName=Training_jhendrick" --insecure
```

The above API call into CloudBees Flow tells Flow to run a pipeline called jhendrick_pipeline within a project called Training_jhendrick.

You’ll also notice that we’re using a Jenkins X feature (jx step credential) to get our secret, flow-token, which we created previously so that we can authenticate to the Flow Rest API. Note that there are many other possible ways to call into the CloudBees Flow API’s besides cURL such as the command line tool ectool as well as python or groovy libraries.  Also note that for a production environment we would want to setup the proper certificates rather than using the --insecure parameter.

Next, we need to tell Jenkins X to use our new build pack: 

```bash
jx edit buildpack -u https://github.com/jhendrickCB/jenkins-x-kubernetes -r master -b
```

{{< image src="/img/jenkins-x-flow-integration/jx-edit-buildpack.jpg" alt="Jenkins X Editing the Current Build Pack" >}}

Finally, we’re going to use [cURL](https://curl.haxx.se/) to call CloudBees Flow.  Since we have to authenticate when calling the [Flow REST API](http://docs.electric-cloud.com/eflow_doc/9_0/API/HTML/FlowAPI_Guide_9_0.htm), we’ll create a Kubernetes secret to store our username/password authentication token:

```bash
apiVersion: v1
kind: Secret
metadata:
 name: flow-token
type: Opaque
data:
 token: <Basic Auth Token>
```

Note: In this case, the <Basic Auth Token> will take the form of Username:Password base64 encoded.  Take note that we’ll actually need to base64 encode our Username:Password token twice as it will get base64 decoded automatically when we access it later.

To apply the secret in our Kubernetes cluster, we can save our secret to a file called flow-token-secret.yaml and run the command: 

```bash
kubectl apply -f flow-token-secret.yaml
```

# Creating a Sample Spring Boot Project
To test out our new build pack, we’ll use Jenkins X’s capability to create a quick start project for a Spring Boot microservice:

```bash
jx create spring -d web -d actuator
```

Follow the prompts to create the Spring Boot project and setup the repository on your GitHub account:

{{< image src="/img/jenkins-x-flow-integration/jx-create-quickstart.jpg" alt="Jenkins X Create Quick Start Project" >}}

Once created, the project should build and run automatically.  If everything worked, we should see our Spring Boot project built with Maven, artifacts uploaded automatically to our Nexus repository and then our CloudBees Flow pipeline executed within our CloudBees Flow server.

If for some reason, we made a mistake, the pipeline can be re-run by using: 

```bash
jx start pipeline
```

To debug, build logs can be checked with: 

```bash
jx get build logs 
```

Or more specifically with our project name:

```bash
jx get build logs jhendrickCB/jx-spring-flowdemo/master
```

We can get build activity with: 

```bash
jx get activity -w
```

Or more specifically:

```bash
jx get activity -f jx-spring-flowdemo -w
```

# In Conclusion
In conclusion, in the above example we were able to use Jenkins X to build our application as well as store the built artifacts, and then utilize CloudBees flow to handle execution of our release pipeline.  This allows us to take advantage of the scalability and efficiency of Jenkins X while leveraging the power and control of CloudBees Flow for managing the release. 

For organizations who want to take advantage of modern CI/CD on Jenkins X but are not yet "all in" on Kubernetes and still deploying traditional applications, this provides a very solid approach to achieving Continuous Delivery.
