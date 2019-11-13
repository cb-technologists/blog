---
title: Serverless Preview Environments and GitOps with CloudBees Core and Google Cloud Run
authors:
  - "Kurt Madel"
  - "Logan Donley"
date: 2019-11-13T05:05:15-04:00
showDate: true
tags: ["Kubernetes","containers","Cloud Run","serverless","CaaS","FaaS","CloudBees Core","Anthos","CI","CD","GKE","Workload Identity"]
photo: "/posts/cloud-run-with-core/badlands-clouds.png"
photoCaption: "Badlands National Park, SD<br>Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 13.75mm ƒ/6.3 1/800"
canonicalUrl: 
draft: true
---

[Google Cloud Run](https://cloud.google.com/run/) is a Google Cloud serverless platform for stateless containerized applications that leverage HTTP and event driven workloads. Cloud Run can be fully managed or run on Cloud Run Anthos - either GKE on Google Cloud or on-premises.

CloudBees Core is an enterprise version of Jenkins that provides better scalability, security and availability by running on and leveraging Kubernetes.

In this post we will explore a combination of features and best practices for using CloudBees Core on Kubernetes to deploy serverless preview development environments for GitHub Pull Requests(PR) to Cloud Run, allowing developers to review and test changes for a web application before those changes are merged to the master branch and deployed to production. After the PR is reviewed and merged to the master branch, the web application will be deployed to GKE on Google Cloud running Cloud Run for Anthos. Finally, CloudBees Core [external HTTP endpoints](https://docs.cloudbees.com/docs/cloudbees-core/latest/cloud-admin-guide/external-http-endpoints) for [CloudBees Cross Team Collaboration](https://docs.cloudbees.com/docs/cloudbees-core/latest/cloud-admin-guide/cross-team-collaboration) will be used to automatically clean up the PR Cloud Run preview environment.


## Why Serverless?
Of course serverless doesn't mean there aren't any servers. Rather serverless refers to reducing or completely removing the need to manage infrastructure for applications and making deployment of those applications easier. Cloud Run takes the auto-management of your application deployment to a new level by providing managed autoscaling, redundancy, and TLS. And when your application isn't servicing requests, it is spun down and you pay nothing.

### Why Cloud Run?
There are already a number of articles that compare Cloud Run to other serverless offerings. And just to be clear, Cloud Run is not a Function-as-a-Service (FaaS) offering. Cloud Run is more akin to a Container-as-a-Service (CaaS) and as such has several advantages over FaaS offerings. These advantages include more flexibility, better testability and portability as [outlined by this great post](https://medium.com/google-cloud/cloud-run-and-cloud-function-what-i-use-and-why-12bb5d3798e1) by Guillaume Blaquiere. In that article, Guillaume Blaquiere comes to the conclusion the he would rather use Cloud Run than Google Cloud Functions.

#### A Knative Foundation Provides Portability

Cloud Run is built on top of the open source [Knative project](https://knative.dev/) that describes itself as a "platform to deploy and manage modern serverless workloads." This provides a level of portability that is atypical of most other serverless offerings from other Cloud providers. Here is the resulting Kubernetes Knative YAML manifest from deploying to the managed Cloud Run service:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hugo-cloud-run
  namespace: cloud-run
  selfLink: /apis/serving.knative.dev/v1/namespaces/cloud-run/services/hugo-cloud-run
  uid: 86050f4c-0555-11ea-a0c6-42010a9600e6
  resourceVersion: '82604296'
  generation: 1
  creationTimestamp: '2019-11-12T14:05:46Z'
  annotations:
    client.knative.dev/user-image: gcr.io/core-workshop/hugo-cloud-run:master-4
    serving.knative.dev/creator: core-cloud-run@core-workshop.iam.gserviceaccount.com
    serving.knative.dev/lastModifier: core-cloud-run@core-workshop.iam.gserviceaccount.com
spec:
  traffic:
  - percent: 100
    latestRevision: true
  template:
    metadata:
      labels:
        client.knative.dev/nonce: oxxbnmdxld
      initializers: {}
    spec:
      timeoutSeconds: 300
      containers:
      - name: user-container
        image: gcr.io/core-workshop/hugo-cloud-run:master-4
        resources: {}
        readinessProbe:
          successThreshold: 1
status:
  conditions:
  - type: ConfigurationsReady
    status: 'True'
    lastTransitionTime: '2019-11-12T14:05:54Z'
  - type: Ready
    status: 'True'
    lastTransitionTime: '2019-11-12T14:05:56Z'
  - type: RoutesReady
    status: 'True'
    lastTransitionTime: '2019-11-12T14:05:56Z'
  observedGeneration: 1
  traffic:
  - revisionName: hugo-cloud-run-pprqg
    percent: 100
    latestRevision: true
  latestReadyRevisionName: hugo-cloud-run-pprqg
  latestCreatedRevisionName: hugo-cloud-run-pprqg
  address:
    url: http://hugo-cloud-run.cloud-run.svc.cluster.local
  url: http://hugo-cloud-run.cloud-run.knative.***.***
```

The YAML spec for a Cloud Run services are available via the UI of the **Cloud Run > Service Details** console. You may also retrieve it with the following [Google Cloud SDK command](https://cloud.google.com/sdk/gcloud/reference/beta/run/services/describe): 

```bash
gcloud beta run services describe hugo-cloud-run --platform gke \
--cluster core-labs-cb-sa --cluster-location us-east4-b \
--namespace cloud-run --format=yaml
```

Furthermore, the current alpha release of the Google Cloud SDK allows [creating or replacing a Cloud Run service from such a YAML specification](https://cloud.google.com/sdk/gcloud/reference/alpha/run/services/replace).

## Ephemeral Preview Environments for Continuous Delivery with Cloud Run and Core

When developers commit deployable code they want to see it working, especially for web based applications. [Preview environments](https://jenkins-x.io/developing/preview/) for GitHub Pull Requests is a developer friendly feature that [Jenkins X](https://jenkins-x.io/) has provided for some time now but preview environments aren't limited to Jenkins X. Cloud Run provides an excellent platform for creating light-weight ephemeral preview environments for stateless containers that leverage HTTP workloads - and even better, when your application isn't receiving requests your service is scaled down to zero and you pay nothing. So even if that PR sits there for a few days, you only pay for the time your application is being used and that may only be a handful of minutes over several days. 

CloudBees Core provides the perfect balance of flexibility and operational consistency for Pipelines to orchestrate preview environments and GitOps with GitHub and Cloud Run. By combining team specific Jenkins Masters with Pipeline templates and external event notifications we are able to create a complex orchestration that is easy for developers to use - so they can concentrate on their code.

### CloudBees Pipeline Template Catalogs

[CloudBees Pipeline Template Catalogs](https://docs.cloudbees.com/docs/admin-resources/latest/pipeline-templates-user-guide/setting-up-a-pipeline-template-catalog), paired with [Pipeline Shared Libraries](https://jenkins.io/doc/book/pipeline/shared-libraries/), provide an easily managed and scalable solution to support the seamless deployments of hundreds of applications to Cloud Run while employing best practices around security, compliance, performance and agent management for Jenkins Pipelines. All the developer has to do is fill in a few template parameters and they have an instant [Multi-branch Pipeline](https://jenkins.io/doc/book/pipeline/multibranch/#creating-a-multibranch-pipeline) that provides end to end deployment from PR preview environments to production deployments for the master branch. And better yet, once their templated Jenkins Pipeline is created they never have to leave GitHub since the ***Hugo Pipeline*** template will update the PR with a link to a preview environment for the updated blog site and a merge to the master branch will automatically trigger a production deployment.

The [***Hugo Pipeline*** template](https://github.com/cloudbees-days/pipeline-template-catalog/tree/master/templates/hugo) will:

1. [Be parameterized](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/hugo/template.yaml) to allow deploying to the fully managed Cloud Run, Cloud Run for Anthos on Google Cloud (GKE) or Cloud Run for Anthos for GKE on-prem - and a different deployment target can be selected for PRs vs master branch deployments.
2. [Use Hugo to generate the static website](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/hugo/Jenkinsfile#L40).
3. [Build a container image](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/hugo/Jenkinsfile#L47) using [img](https://github.com/genuinetools/img) and push to GCR with [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).
4. [Use the Google Cloud SDK with GKE Workload Identity to deploy the container as a Cloud Run service](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/hugo/Jenkinsfile#L55) - a temporary preview environment for PRs and a production Cloud Run deployment for the master branch.
5. [For GitHub Pull Requests (PR), use the GitHub API to add a comment to the PR with a link to the running Cloud Run Service](https://github.com/cloudbees-days/pipeline-library/blob/master/vars/cloudRunDeploy.groovy#L26-L35).
6. [For merged/closed PRs, use the Google Cloud SDK with GKE Workload Identity to delete the PR associated Cloud Run deployment](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/hugo/Jenkinsfile#L99-L121).

#### Pipeline Shared Libraries

A Pipeline Shared Library provides reusable global variables used in the catalog template that are similar in use to built-in Pipeline steps. The shared library will also provide Kubernetes Pod specs (Jenkins agent templates) for ephemeral containerized agents.  The structure and the pertinent files of the [shared library used with the ***Hugo Pipeline*** template](https://github.com/cloudbees-days/pipeline-library) are described below:

```
+- vars
|   +- cloudRunDeploy.groovy             # uses Google Cloud SDK to deploy container images to Cloud Run, also updates PRs with link to Cloud Run service
|   +- cloudRunDelete.groovy             # uses Google Cloud SDK to delete Cloud Run services
|   +- containerBuildPushGeneric.groovy  # uses img to build and push container images to a container registry
+- resources                             # resource files 
|   +- podtemplates
|       +- containerBuildPush.yml        # provides img with Google Cloud SDK for building and pusing container images, used by containerBuildPushGeneric.groovy
|       +- cloud-run.yml                 # Provides Google Cloud SDK, used by both cloudRunDeploy.groovy and cloudRunDelete.groovy 
|       +- hugo
|           +- pod.yml                   # provides Hugo image for generating static content, used directly by the Huge Pipeline template Jenkinsfile
```

#### Security

By leveraging [Workload Identity for GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) we are able to securely provide GCP IAM permissions without exporting service account keys (keys that don't expire for 10 years unless manually rotated). You will note that neither the [`cloudRunDeploy.groovy`](https://github.com/cloudbees-days/pipeline-library/blob/master/vars/cloudRunDeploy.groovy) nor the [`cloudRunDelete.groovy`](https://github.com/cloudbees-days/pipeline-library/blob/master/vars/cloudRunDelete.groovy) shared library scripts have any explicit Google Cloud authentication steps. That is because the Google Cloud SDK provides seamless integration with GKE Workload Identity and automatically authenticates when accessing Google Cloud APIs with a Kubernetes `ServiceAccount` that is bound to an IAM Service Account with Workload Identity. To set this up we:

1. Created a GCP IAM Service Account with the most limited set of permissions for pushing and pulling GCR container images and deploying, describing and deleting Cloud Run services.
2. Created a Cloud Run specific Kubernetes Namespace and Kubernetes `ServiceAccount` in our CloudBees Core GKE cluster.
   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: cloud-run
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: cloud-run-sa
     namespace: cloud-run
   ```
3. Bound the IAM Service Account to a Kubernetes Service Account.
   ```
   gcloud iam service-accounts add-iam-policy-binding \
     --role roles/iam.workloadIdentityUser \
     --member "serviceAccount:core-workshop.svc.id.goog[cloud-run/cloud-run-sa]" \
     core-cloud-run@core-workshop.iam.gserviceaccount.com
   ```
4. Created a Jenkins Kubernetes Cloud configured to connect to the CloudBees Core GKE cluster with the IAM bound Kubernetes `ServiceAccount` `cloud-run-sa`.
   ```yaml
     clouds:
     - kubernetes:
         connectTimeout: 5
         containerCapStr: "10"
         credentialsId: "k8s-cloud-run-sa"
         defaultsProviderTemplate: "default-jnlp"
         maxRequestsPerHostStr: "32"
         name: "kubernetes"
         namespace: "cloud-run"
   ```
   The `k8s-cloud-run-sa`  `credentialsId` refers to a Jenkins Secret Text credential with the value being the `ServiceAccount` token of the `cloud-run-sa` Kubernetes `ServiceAccount` and only the Team Master that is configured to use this Jenkins credential will be able to provision Kubernetes agent `Pods` with the `cloud-run-sa` `ServiceAccount` thus limiting access to deploy to Cloud Run to the team with access to this Team Master.
   *from https://github.com/kypseli/demo-mm-jcasc/blob/cloud-run/jcasc.yml*
5. Created a Jenkins Kubernetes Pod Template to run the `google/cloud-sdk:252.0.0-slim` container image.
   ```yaml
   kind: Pod
   metadata:
     name: cloud-run-pod
   spec:
     containers:
     - name: gcp-sdk
       image: google/cloud-sdk:252.0.0-slim
       command:
       - cat
       tty: true
       volumeMounts:
         - name: gcp-logs
           mountPath: /.config/gcloud/logs
     volumes:
     - name: gcp-logs
       emptyDir: {}
   ```
   *from https://github.com/cloudbees-days/pipeline-library/blob/master/resources/podtemplates/cloud-run.yml*
6. Use the Google Cloud SDK from within the a Jenkins Pipeline, in this case from a shared library script with Workload Identity taking care of authenticating with the `core-cloud-run@core-workshop.iam.gserviceaccount.com` IAM service account that has permissions to deploy to Cloud Run:
   ```groovy
   def call(Map config) {
     def podYaml = libraryResource 'podtemplates/cloud-run.yml'
     def label = "cloudrun-${UUID.randomUUID().toString()}"
     def CLOUD_RUN_URL
     podTemplate(name: 'cloud-run-pod', label: label, yaml: podYaml, nodeSelector: 'workload=general') {
       node(label) {
         container(name: 'gcp-sdk') {
          sh "gcloud beta run deploy ${config.serviceName} --image ${config.image} --platform gke --cluster ${config.clusterName} --cluster-location ${config.region} --namespace ${config.namespace}"
         }
       }
     }
   }
   ```
   *from https://github.com/cloudbees-days/pipeline-library/blob/master/vars/cloudRunDeploy.groovy*
7. 


With this approach we have no long-lived GCP IAM Service Account key file, no mounting Kubernetes `Secrets`, and no Jenkins Credentials. The actual GCP service account token that is finally created for authentication is short lived and non-persistent.

*Check out this blog post for more details on [using Workload Identity with CloudBees Core](https://technologists.dev/posts/best-practices-for-cloudbees-core-jenkins-on-kubernetes/securely-using-cloud-services-from-jenkins-kubernetes-agents/).*

We are also using [img]() along with a very restrictive [Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) to provide a more secure Kubernetes CI/CD environment for container image builds.

*Checkout out this blog post for details on [using img for securely building container images on Kubernetes with Jenkins](https://technologists.dev/posts/building-images-with-img/).*

#### Preview Environment Clean-up with CloudBees Cross Team Collaboration

One useful feature of CloudBees Cross Team Collaboration is support for external notifications. Notification Webhook HTTP Endpoints for GitHub webhooks allow us to create a GitHub webhook for the Hugo blog repository that uses that CloudBees Core notification endpoint as its *Payload URL* and is only triggered on PR events. We can then set-up an event trigger for `closed` PRs for the Hugo blog repository and add that as a trigger for the ***Hugo Pipeline*** template: 

```groovy
  triggers {
    eventTrigger jmespathQuery("action=='closed' && repository.full_name=='${repoOwner}/${repo}'")
  }
```

Then add a `Stage` to the ***Hugo Pipeline*** template configured with a conditional `when` clause on a that will only be executed when the `eventTrigger` conditions are met - a closed PR on the blog repository and only for the `master` branch (as we can't rely on other branches not being deleted). Then use the Jenkins CLI to get the Cross Team Collaboration payload and just jq to extract the PR number from the GitHub webhook payload and pass it, along with other Cloud Run specific parameters, to the `cloudRunDelete.groovy` shared library pseudo step. Here is the entire `Stage`:

```groovy
    stage('PR Delete') {
      agent {
        kubernetes {
          label 'default-jnlp'
        }
      }
      when {
        beforeAgent true
        allOf {
          branch 'master'
          triggeredBy 'EventTriggerCause' 
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'cli-username-token', usernameVariable: 'JENKINS_CLI_USR', passwordVariable: 'JENKINS_CLI_PSW')]) {
          script {
            prNumber = sh (script: "curl -u $JENKINS_CLI_USR:$JENKINS_CLI_PSW --silent ${BUILD_URL}api/json | jq -r '.actions[0].causes[0].event.number' | tr -d '\n'", 
                returnStdout: true)
          }
        }
        cloudRunDelete(serviceName: "${projectName}-pr-${prNumber}", deployType: "${deployTypePR}", region: "${gcpRegionPR}", clusterName: "${clusterNamePR}", namespace: "${namespacePR}")
      }
    }
```

## Summary

CloudBees Core Pipeline Templates provide the flexibility for incorporating best practices around security, compliance, performance and agent management for Jenkins Pipelines. And combining CloudBees Core with Cloud Run provides a streamlined approach for providing easily consumable developer focused deployment environments for CI/CD Pipelines. 

