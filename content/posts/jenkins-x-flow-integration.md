---
authors:
  - "Josh Hendrick"
title: "Extending Jenkins X for Traditional Deployments with CloudBees Flow"
date: 2019-05-29T12:47:46-04:00
showDate: true
draft: false
tags: ["jenkins","jenkins x","cloudbees flow", "tekon", "extending jenkins x"]
---
[Jenkins X](https://jenkins-x.io) is quickly becoming the de facto standard for high performing teams wanting to do CI/CD in a highly scalable and fault tolerant environment. For those who haven’t gotten the opportunity to try out Jenkins X, it allows teams to run CI/CD workloads natively in a Kubernetes environment while taking advantage of modern operating patterns like GitOps and serverless architectures. For teams wanting to modernize their continuous integration and continuous deployment capabilities, Jenkins X is the go to solution.

In today’s heterogenous technology environment, most organizations tend to have a mix of modern cloud native architectures as well as more traditional workloads which get deployed either on-prem or within the cloud. In the latter case, a combination of Jenkins X (performing CI steps) and CloudBees Flow (handling the deployment) can add a huge amount of flexibility and power to a Continuous Delivery process.  The combination of Jenkins X and CloudBees Flow also brings improved visibility and tracability across the application landscape. 

Jenkins X can be easily extended to accommodate any type of workload required - it can be a full end to end CI/CD tool for building, deploying, and running applications all within a Kubernetes cluster, or it can handle CI while offloading other release and deployment tasks to another solution.  In this blog post we’re going to cover how Jenkins X can be extended to offload release/deployment tasks to [CloudBees Flow](https://www.cloudbees.com/cloudbees-acquires-electric-cloud).  We will accomplish this by extending the maven Jenkins X build pack in order to call the CloudBees Flow REST API as part of the Jenkins X pipeline execution.

# Extending Jenkins X
For the purposes of this blog, we’re going to be focusing on the Jenkins X serverless pipeline execution engine with Tekton (See https://jenkins-x.io/architecture/jenkins-x-pipelines/). There are two main ways to customize a Jenkins X pipeline in order to integrate with CloudBees Flow.  The first and simplest would be to modify the jenkins-x.yml (more information on Jenkins X pipelines: https://jenkins-x.io/architecture/jenkins-x-pipelines/#differences-to-jenkins-pipelines and the jenkins-x.yml file) pipeline file in the source code repo for the project we’re going to build.  The other way is to extend the [Jenkins X build packs](https://jenkins-x.io/architecture/build-packs/) and modify the build pack for the language/build tool you want to use.  Both will work, but by forking the build packs you can get reuse across multiple projects using the build pack you extend. In this example, we’ll walk through how to extend the Jenkins X build packs.

## Creating our Cluster and Installing Jenkins X
To start, we’ll fork the Jenkins X Kubernetes build packs into our own repository: https://github.com/jenkins-x-buildpacks/jenkins-x-kubernetes.  Later we'll be extending the maven build pack to support a REST API call into CloudBees Flow.

Now it’s time to create a Kubernetes cluster on GKE using Jenkins X and [Tekton](https://github.com/tektoncd/pipeline).  In this case, we're starting by creating a cluster from scratch, but Jenkins X can also be installed into an existing Kubernetes cluster if you already have one available by using the `jx install` command: 

```bash
jx create cluster gke --tekton --no-tiller
```

Fill out the options.  For example:

```shell
$  jx create cluster gke
Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=....


? Google Cloud Project: jhendrick-ckcd
Updated property [core/project].
Lets ensure we have container and compute enabled on your project
No apis need to be enable as they are already enabled: container compute
No cluster name provided so using a generated one: crownprong
? What type of cluster would you like to create Zonal
? Google Cloud Zone: us-west1-a
? Google Cloud Machine Type: n1-standard-4
? Minimum number of Nodes (per zone) 3
? Maximum number of Nodes 5
? Would you like use preemptible VMs? No
? Would you like to access Google Cloud Storage / Google Container Registry? No
Creating cluster...
Initialising cluster ...
? Select Jenkins installation type: Serverless Jenkins X Pipelines with Tekton
Setting the dev namespace to: jx
Namespace jx created 
```

Create an ingress controller if one doesn’t exist and setup the domain or use the default *.nip.io address if you don’t have one.  Go through the prompts and then configure your GitHub credentials.  Create an API token using the URL provided if you don’t have one:

```shell
If you don't have a wildcard DNS setup then setup a DNS (A) record and point it at: 35.197.85.1 then use the DNS domain in the next input...
? Domain 35.197.85.1.nip.io
nginx ingress controller installed and configured
? Would you like to enable Long Term Storage? A bucket for provider gke will be created No
Lets set up a Git user name and API token to be able to perform CI/CD

Creating a local Git user for GitHub server
? GitHub user name: jhendrick
To be able to create a repository on GitHub we need an API Token
Please click this URL https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,delete_repo

Then COPY the token and enter in into the form below:

? API Token: ****************************************
Select the CI/CD pipelines Git server and user
? Do you wish to use GitHub as the pipelines Git server: Yes
Creating a pipelines Git user for GitHub server
To be able to create a repository on GitHub we need an API Token
Please click this URL https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,delete_repo

Then COPY the token and enter in into the form below:

? API Token: ****************************************
Setting the pipelines Git server https://github.com and user name jhendrick.
Saving the Git authentication configuration
```

In the setup we’re going to choose the Kubernetes workloads option and later modify the kubernetes workload build packs to include the CloudBees Flow specific steps:

```shell
? Pick default workload build pack: [Use arrows to move, space to select, type to filter]
> Kubernetes Workloads: Automated CI+CD with GitOps Promotion
Library Workloads: CI+Release but no CD
```

## Editing the Build Packs
You can use your favorite IDE but in this case, we'll modify the Jenkins X build packs in VS Code with the YAML Language extension installed (https://jenkins-x.io/architecture/jenkins-x-pipelines/#editing-in-vs-code) for validation as recommended by the Jenkins X team.

This example is going to focus on a sample Spring Boot application using Maven.  To start we'll modified the maven build pack in our forked build pack repo (https://github.com/jhendrickCB/jenkins-x-kubernetes/blob/master/packs/maven/pipeline.yaml):

```yaml
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
         curl -X POST --header "Authorization: Basic $(jx step credential -s flow-token -k token)" --header "Content-Type: application/json" --header "Accept: application/json" -d "{}" "https://ps9.ecloud-kdemo.com/rest/v1.0/pipelines?pipelineName=my_pipeline&projectName=my_project" --insecure
       name: cloudbees-flow-release
```

Compare the original build pack for maven found here: https://github.com/jenkins-x-buildpacks/jenkins-x-kubernetes/tree/master/packs/maven vs. our forked build pack.  We’ve removed all the references to skaffold, watch, and helm since we’re no longer having Jenkin’s X handle the deployment to our Kubernetes cluster.  We’ve also updated the pipeline file to make an API call into our CloudBees Flow server using [cURL](https://curl.haxx.se/):

```bash
curl -X POST --header "Authorization: Basic $(jx step credential -s flow-token -k token)" --header "Content-Type: application/json" --header "Accept: application/json" -d "{}" "https://ps9.ecloud-kdemo.com/rest/v1.0/pipelines?pipelineName=my_pipeline&projectName=my_project" --insecure
```

The above API call into CloudBees Flow tells Flow to run a pipeline called `my_pipeline` within a project called `my_project`.

You’ll also notice that we’re using a Jenkins X feature (`jx step credential`) to get our secret, `flow-token`, which we created previously so that we can authenticate to the Flow Rest API. Note that there are many other possible ways to call into the CloudBees Flow API’s besides cURL such as the command line tool [ectool](http://docs.electric-cloud.com/eflow_doc/9_0/API/HTML/FlowAPI_Guide_9_0.htm#EFlow_api/usingAPI.htm?Highlight=ectool) as well as [perl](http://docs.electric-cloud.com/eflow_doc/9_0/API/HTML/FlowAPI_Guide_9_0.htm#EFlow_api/usingAPI.htm%3FTocPath%3DUsing%2520the%25C2%25A0ElectricFlow%2520Perl%2520API%7C_____0) or [groovy libraries](http://docs.electric-cloud.com/eflow_doc/9_0/API/HTML/FlowAPI_Guide_9_0.htm#EFlow_api/UsingGroovy.htm%3FTocPath%3D_____11).  Also note that for a production environment we would want to setup the proper certificates rather than using the `--insecure parameter`.

Next, we need to tell Jenkins X to use our new build pack: 

```shell
$ jx edit buildpack -u https://github.com/jhendrickCB/jenkins-x-kubernetes -r master -b

Setting the team build pack to  repo: https://github.com/jhendrickCB/jenkins-x-kubernetes ref: master
```

Since we have to authenticate when calling the [Flow REST API](http://docs.electric-cloud.com/eflow_doc/9_0/API/HTML/FlowAPI_Guide_9_0.htm), we’ll create a Kubernetes secret to store our username/password basic authentication token:

```yaml
apiVersion: v1
kind: Secret
metadata:
 name: flow-token
type: Opaque
data:
 token: <Basic Auth Token>
```

Note: In this case, the `<Basic Auth Token>` will take the form of `username:password` base64 encoded.  Take note that we’ll actually need to base64 encode our username:password token twice as it will get base64 decoded automatically when we access it later.

To apply the secret in our Kubernetes cluster, we can save our secret to a file called `flow-token-secret.yaml` and run the command: 

```bash
kubectl apply -f flow-token-secret.yaml
```

# Creating a Sample Spring Boot Project
To test out our new build pack, we’ll use Jenkins X’s capability to create a quick start project for a Spring Boot microservice:

```bash
jx create spring -d web -d actuator
```

Follow the prompts to create the Spring Boot project and setup the repository on your GitHub account:

```shell
$ jx create spring -d web -d actuator
Using Git provider GitHub at https://github.com
? Do you wish to use jhendrick as the Git user name? Yes


About to create repository  on server https://github.com with user jhendrick
? Which organisation do you want to use? jhendrickCB
? Enter the new repository name:  jx-spring-flowdemo


Creating repository jhendrickCB/jx-spring-flowdemo
? Language: java
? Group: com.example
Created Spring Boot project at /Users/jhendrick/Cloudbees/jx-spring-flowdemo
The directory /Users/jhendrick/Cloudbees/jx-spring-flowdemo is not yet using git
? Would you like to initialise git now? Yes
? Commit message:  Initial import

Git repository created
selected pack: /Users/jhendrick/.jx/draft/packs/github.com/jhendrickCB/jenkins-x-kubernetes/packs/maven

replacing placeholders in directory /Users/jhendrick/Cloudbees/cloudbees-days/kops-cluster/jx-spring-flowdemo
app name: jx-spring-flowdemo, git server: github.com, org: jhendrickcb, Docker registry org: jhendrickcb
skipping directory "/Users/jhendrick/Cloudbees/jx-spring-flowdemo/.git"
skipping ignored file "/Users/jhendrick/Cloudbees/jx-spring-flowdemo/HELP.md"
Pushed Git repository to https://github.com/jhendrickCB/jx-spring-flowdemo

Creating GitHub webhook for jhendrickCB/jx-spring-flowdemo for url http://hook.jx.35.197.85.1.nip.io/hook

Watch pipeline activity via:    jx get activity -f jx-spring-flowdemo -w
Browse the pipeline log via:    jx get build logs jhendrickCB/jx-spring-flowdemo/master
Open the Jenkins console via    jx console
You can list the pipelines via: jx get pipelines
When the pipeline is complete:  jx get applications

For more help on available commands see: https://jenkins-x.io/developing/browsing/

Note that your first pipeline may take a few minutes to start while the necessary images get downloaded!
```

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
In the above example we were able to use Jenkins X to build our application as well as store the built artifacts, and then utilize CloudBees flow to handle execution of our release pipeline.  This allows us to take advantage of the scalability and efficiency of Jenkins X while leveraging the power and control of CloudBees Flow for managing the release. 

For organizations who want to take advantage of modern CI/CD on Jenkins X but are not yet "all in" on Kubernetes and still deploying traditional applications, this provides a very solid approach to achieving Continuous Delivery.
