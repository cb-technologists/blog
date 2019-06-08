---
author:
  name: "Kurt Madel"
title: "CloudBees' Cross Team Collaboration for Asynchronous DevSecOps"
date: 2019-06-08T05:50:46-04:00
showDate: true
photo: "/img/cloudbees-cross-team-dev-sec-ops/locks.png"
photoCaption: "Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 10.4mm ƒ/1.8 1/250"
draft: true
tags: ["jenkins","plugins","containers","CasC","DevSecOps","security","anchore","container scanning"]
---
## What is Cross Team Collaboration?
CloudBees' Cross Team Collaboration provides the ability to publish an event from a Jenkins job that triggers any other Jenkins job on the same master or different masters that are listening for that event. It is basically a light-weight [**PubSub**](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) for CloudBees Core Masters connected to [CloudBees Operations Center](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/operating/#managing-operation-center). Jenkins has had the ability to [trigger other jobs](https://jenkins.io/doc/pipeline/steps/pipeline-build-step/) for quite a while now (and [with CloudBees this is even easy to do across Masters](https://support.cloudbees.com/hc/en-us/articles/226408088-Trigger-jobs-across-masters)), but it always required that the upstream job be aware of the downstream job(s) to be triggered. The Cross Team Collaboration feature provides a loosely coupled link between upstream and downstream Jenkins jobs - so that any job that is interested in a certain event, for whatever reason, can subscribe to that event and get triggered whenever that event is published.


There are a few good CloudBees' blog posts and CloudBees' documentation on CloudBees' Cross Team Collaboration: 

- [Cross Team Collaboration (Part 1)](https://www.cloudbees.com/blog/cross-team-collaboration-part-1)
- [Cross Team Collaboration (Part 2)](https://www.cloudbees.com/blog/cross-team-collaboration-part-2)
- [Cross Team Collaboration documentation](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/)

## DevSecOps
[DevSecOps](https://tech.gsa.gov/guides/understanding_differences_agile_devsecops/) - the idea of shifting security left in your Continuous Delivery pipelines - is becoming a vital component of successful CD. DevSecOps is all about speeding up software delivery, while maintaining, or even improving, the level of security for delivered application code. However, even though you should be shifting automated security left - you still don't want it to impede developers trying to deliver software more quickly. CloudBees' Cross Team Collaboration feature is a perfect capability for automating security while at the same time getting out of the way of developers - improving the security and quality of your software delivery while minimizing the impact on delivery speed.

## Use Case: Asynchronously Scan Container Images for Vulnerabilities and Compliance
As containers become a more and more ubiquitous method for delivering your applications, ensuring that your container images don't have security vulnerabilities and/or organization specific security compliance issues is an important aspect of CD for containerized application delivery. However, scanning containers images isn't the fastest process in the world and you don't want to unnecessarily slow down developers trying to get stuff done. You also may not want to depend on individual development teams to configure and manage important securitys steps in their delivery pipelines.

Cross Team Collaboration enables you to publish an event from a [Pipeline Shared Library](https://jenkins.io/doc/book/pipeline/shared-libraries/) for securely building container images and then asynchronously triggering *not-so-quick* security related jobs listening for events, making it very easy to provide security as part of the CD pipelines for an entire organization.

### Cross Team Collaboration Events
There are basically two types of Cross Team Collaboration events:

1. Simple Event: `publishEvent simpleEvent("${dockerReg}/helloworld-nodejs:${repoName}-${BUILD_NUMBER}")`
2. JSON Event: `publishEvent event:jsonEvent("{'eventType':'containerImagePush', 'image':'${dockerReg}/helloworld-nodejs:${repoName}-${BUILD_NUMBER}'}"), verbose: true`

For this example we will be using the more verbose JSON event. The problem with the **Simple Event** approach is that the triggered job would have to subscribe to a single `string` value and in this case a specific container image. But what we really want is to run an Anchore scan for all container images being pushed to our DEV container registry. The **JSON Event** approach allows us to subscribe to a more generic event, `containerImagePush`, while passing the exact container image being pushed as an additional JSON value for the key `image`.  But to use this approach the triggered job(s) must retrieve the value of the `image` key from the event payload.

### Capturing the Cross Team Collaboration Event Payload
Groovy code vs a `curl` REST call against the [Jenkins REST API](https://wiki.jenkins.io/display/JENKINS/Remote+access+API) to get the JSON event payload:

- You could get the event JSON with the following: `currentBuild.getBuildCauses()[0].event.toString()`. But that will run on the Jenkins Master, not the Jenkins agent and will impact performance when you are scanning hundreds or even thousands of container images.
- Using `sh` step with a `curl` call against the Jenkins REST API with a [Jenkins API token](https://jenkins.io/blog/2018/07/02/new-api-token-system/) to get the JSON representation of the current build, and then piping the JSON response to [**jq**](https://stedolan.github.io/jq/) to get the value for the `image` key from the event payload in a Jenkins Pipeline triggered by the `EventTriggerCause`: `curl -u 'beedemo-admin':$TOKEN --silent ${BUILD_URL}/api/json| jq '.actions[0].causes[0].event.image'`. `BUILD_URL` is one of many [Pipeline global variables](https://jenkins.io/doc/book/pipeline/getting-started/#global-variable-reference) available to all Jenkins Pipeline jobs. The advantages of this approach are:
  - The `sh` step will run on the agent, not the Jenkins Master, allowing you to scale across as many agents as needed for your container scans with very little impact on the performance of the Jenkins Master.
  - Using lightweight shell scripts provide easier testing and more portability of your CD pipelines to other platforms.

### Anchore Inline Scan
Earlier this year, [Anchore](https://anchore.com/) provided some new tools and scripts to make it easier to execute Anchore scans without constantly running an Anchore Engine. The [Anchore **inline scan**](https://anchore.com/inline-scanning-with-anchore-engine/) provides the same analysis/vulnerability/policy evaluation and reporting as a statically managed Anchore engine and is used in this example to highlight how easy and fast you can add container security scanning to your own CD pipelines. However, a better long-term approach would be to stand-up your own centralized, managed and stable Anchore engine to use across all of you dev teams. The advantages of a static, always running Anchore Engine include:

- Faster scans: since you don't have to wait for the Anchore engine to start-up for each job.
- Reduced infrastructure costs: if you only do a few scans a day then this is less of an advantage as you will have a constant infrastructure cost for the static Anchore engine. But if you are doing 100s of scan per day then you will defintely realize savings with this approach.
- More secure: as we will see in the **inline scan** example below, the Anchore `inline_scan` script requires access to a Docker daemon. And in this example we are using the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin) to provide dynamic and ephemeral agent pods for the Anchore inline scan job. A quick and dirty approach - that has a number of security implications - for providing a K8s pod agent access to the Docker daemon is to mount the Docker socket as a `volume` on the pod.

*Anchore inline scan Pod* - `dockerClientPod.yml`
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker-client
    image: gcr.io/technologists/docker-client:0.0.3
    command: ['cat']
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
```

Even though there is an [Anchore plugin for Jenkins](https://plugins.jenkins.io/anchore-container-scanner), there is no reason to install another plugin when you can accomplish the same thing with a very simple `sh` step. As mentioned in my [last post here on the Technologists site](./jenkins-plugins-good-bad-ugly/) - using fewer Jenkins plugins is a **good** thing.

```groovy
container('docker-client'){
  sh "curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 \
  | bash -s -- -f -b ./.anchore_policy.json -p ${containerImage}"
}
```

Again, the only thing required to run the scan above is a Docker daemon. So you could just as easily run that command on your laptop running Docker as on a Jenkins agent that has access to a Docker daemon.

### Putting It All Together
*CloudBees' Pipeline Template Catalog, Pipeline Shared Library, and Cross Team Collaboration*

By combining the new [CloudBees' Pipeline Template Catalogs](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/pipeline/#_setting_up_a_pipeline_template_catalog) with a Pipeline Shared Library and CloudBees' Cross Team Collaboration we are able to provide a robust DevSecOps application delivery that is super easy for development teams to adopt quickly.

First we have the Pipeline Shared Library for building our container images with [Kaniko](https://github.com/GoogleContainerTools/kaniko):

*pipeline shared library* - `kanikoBuildPush.groovy`
```groovy
def call(String imageName, String imageTag = env.BUILD_NUMBER, String gcpProject = "core-workshop", String target = ".", String dockerFile="Dockerfile", Closure body) {
  def dockerReg = "gcr.io/${gcpProject}"
  imageName = "helloworld-nodejs"
  def label = "kaniko-${UUID.randomUUID().toString()}"
  def podYaml = libraryResource 'podtemplates/dockerBuildPush.yml'
  podTemplate(name: 'kaniko', label: label, yaml: podYaml, inheritFrom: 'default-jnlp', nodeSelector: 'type=agent') {
    node(label) {
      body()
      imageNameTag()
      gitShortCommit()
      def repoName = env.IMAGE_REPO.toLowerCase()
      container(name: 'kaniko', shell: '/busybox/sh') {
        withEnv(['PATH+EXTRA=/busybox:/kaniko']) {
          sh """#!/busybox/sh
            /kaniko/executor -f ${pwd()}/${dockerFile} -c ${pwd()} --build-arg context=${repoName} --build-arg buildNumber=${BUILD_NUMBER} --build-arg shortCommit=${env.SHORT_COMMIT} --build-arg commitAuthor=${env.COMMIT_AUTHOR} -d ${dockerReg}/helloworld-nodejs:${repoName}-${BUILD_NUMBER}
          """
        }
      }
      publishEvent event:jsonEvent("{'eventType':'containerImagePush', 'image':'${dockerReg}/helloworld-nodejs:${repoName}-${BUILD_NUMBER}'}"), verbose: true
    }
  }
}
```

Note the `publishEvent` step at the end - after the container image has been successfully built and pushed to our **dev** container registry it will **publish** the `containerImagePush` event. 

*The JSON output for the `publishEvent` step - note the `image` key value is the container image just built and pushed by Kaniko:*
```json
{
  "eventType": "containerImagePush",
  "image": "gcr.io/core-workshop/helloworld-nodejs:beeops-cb-days-7",
  "source":     {
      "type": "JenkinsTeamBuild",
      "buildInfo":         {
          "build": 7,
          "job": "template-jobs/beedemo-admin-helloworld-nodejs/master",
          "jenkinsUrl": "https://********/teams-sec/",
          "instanceId": "d37a81cc1906b6fe684f253a8a07834c",
          "team": "sec"
      }
  }
}
```

The `kanikoBuildPush` shared library is then consumed by a [Pipeline Template Catalog](https://github.com/cloudbees-days/pipeline-template-catalog) template. In this case a [template for Node.js applications](https://github.com/cloudbees-days/pipeline-template-catalog/tree/master/templates/nodejs-app):

[*Pipeline Template*](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/nodejs-app/Jenkinsfile) - **Build and Push Image** `stage`
```groovy
    stage('Build and Push Image') {
      when {
        beforeAgent true
        branch 'master'
      }
      steps {  
        echo "${repoOwner}"
        kanikoBuildPush(env.IMAGE_NAME, env.IMAGE_TAG, "${gcpProject}") {
          checkout scm
        }
      }
      post {
        success {
          slackSend message: "${JOB_NAME} pipeline job is awaiting approval at: ${RUN_DISPLAY_URL}"
        }
      }
    }
```
Again, if the `kanikoBuildPush` library step is successful it will publish a `containerImagePush` event.

Finally, we set-up a job on our **Security** Jenkins Master to listen for the `containerImagePush` event:

[**anchore-scan** `Jenkinsfile`](https://github.com/cloudbees-days/anchore-scan/blob/master/Jenkinsfile)
```groovy
def containerImage
pipeline {
  agent none

  triggers {
      eventTrigger jmespathQuery("eventType=='containerImagePush'")
  }
  
  stages {
    stage('Anchore Scan') {
      agent {
        kubernetes {
          label 'docker-client'
          yamlFile 'dockerClientPod.yml'
        }
      }
      when { 
        triggeredBy 'EventTriggerCause' 
        beforeAgent true
      }
      environment {
        TOKEN = credentials('beedemo-admin-api-key')
      }
      steps {
        script {
          containerImage = sh(script: """
             curl -u 'beedemo-admin':$TOKEN --silent ${BUILD_URL}/api/json| jq '.actions[0].causes[0].event.image'
          """, returnStdout: true)
        }
        echo containerImage
        container('docker-client'){
          sh "curl -s https://ci-tools.anchore.io/inline_scan-v0.3.3 | bash -s -- -f -b ./.anchore_policy.json -p ${containerImage}"
        }
      }
    }
  }
}
```

Note the `eventTrigger` step uses `jmespathQuery` to listen for the `containerImagePush` `eventType`. Also note the `triggerdBy` condition `EventTriggerCause` in the [`when` directive](https://jenkins.io/doc/book/pipeline/syntax/#when) - this will result in the `Anchore Scan` stage only running (and the provisioning of a K8s pod based agent) if this job is triggered by a Cross Team Collaboration event.

If the newly built container image doesn't pass all of the policies specified in the [`.anchore_policy.json`](https://github.com/cloudbees-days/anchore-scan/blob/master/.anchore_policy.json) file then the job will fail.

Here is an example Anchore report for a failed `anchore-scan` job:

```console
Image Digest: sha256:e03d86b75d38d1d18035b58e9e43088c9d0d5dd6e49f2c507d949937174f3465
Full Tag: anchore-engine:5000/helloworld-nodejs:beeops-cb-days-5
Image ID: 0b22d7798cd24465252335d602059fea88128244b623bc4af20926eeec8f9b4c
Status: fail
Last Eval: 2019-06-07T12:56:03Z
Policy ID: custom-anchore-policy-nodejs
Final Action: stop
Final Action Reason: policy_evaluation

Gate              Trigger               Detail                                                                                     Status        
dockerfile        effective_user        User root found as effective user, which is explicity not allowed list                     stop          
dockerfile        instruction           Dockerfile directive 'HEALTHCHECK' not found, matching condition 'not_exists' check        warn          

Image Digest: sha256:e03d86b75d38d1d18035b58e9e43088c9d0d5dd6e49f2c507d949937174f3465
Full Tag: anchore-engine:5000/helloworld-nodejs:beeops-cb-days-5
Status: fail
Last Eval: 2019-06-07T12:56:04Z
Policy ID: custom-anchore-policy-nodejs
```

As you can see from the above output the scan failed because of the `effective_user` trigger - [the official `node` container image we are using from DockerHub runs as `root`](https://github.com/nodejs/docker-node/blob/master/10/alpine/Dockerfile) and [this is a very bad security practice](https://snyk.io/blog/10-docker-image-security-best-practices/) as it allows **container breakouts** where the container user is able to escape the container namespace and interact with other processes on the host.

### Some Improvements

- One improvement would be to run this without mounting the Docker socket in the [`docker-client` container](https://github.com/cloudbees-days/anchore-scan/blob/declarative/dockerClientPod.yml). The Anchore inline-scan script runs a number of Docker commands that requires a Docker daemon - but this is not good security.
- Another improvement would be to extend the `anchore-scan` job to push the container image to a **Prod** container registry on success and notify interested dev teams that their image is now available for production deployments.

### CasC for Cross Team Collaboration Configuration for your CloudBees Core v2 Masters
In order for all of this to work you have to turn on Cross Team Collaboration for all of your Core v2 Masters that you want to publish and subscribe to events. I am a big proponent of CasC for everything so here is an [`init.groovy.d`](https://wiki.jenkins.io/display/JENKINS/Post-initialization+script) script to set-up CasC to automatically enable Cross Team Collaboration notifications for your CloudBees Core v2 Masters on start-up:

[*cb-core-mm-workshop/quickstart/init_61_notification_api.groovy*](https://github.com/kypseli/cb-core-mm-workshop/blob/master/quickstart/init_61_notification_api.groovy):
```groovy
import jenkins.model.Jenkins
import hudson.ExtensionList

import com.cloudbees.jenkins.plugins.notification.api.NotificationConfiguration
import com.cloudbees.jenkins.plugins.notification.spi.Router
import com.cloudbees.opscenter.plugins.notification.OperationsCenterRouter

jenkins = Jenkins.getInstance()

NotificationConfiguration config = ExtensionList.lookupSingleton(NotificationConfiguration.class);
Router r = new OperationsCenterRouter();
        config.setRouter(r);
        config.setEnabled(true);
        config.onLoaded();
```

I'm also a big fan of the Jenkins Config-as-Code plugin. However, currently, the CloudBees' plugins for Cross Team Collaboration do not yet support [JCasC](https://github.com/jenkinsci/configuration-as-code-plugin) (but support for JCasC is coming soon).


## Add DevSecOps to Your CD with CloudBees Now
So there is really no execuse NOT to add asynchronous container security scans to your container image CD pipelines when it is as easy as this with CloudBees Core v2, Cross Team Collaboration and the Anchore **inline scan** - when it is as easy as this!