---
title: Securely Using Cloud Services from Jenkins Kubernetes Agents
series: ["Best Practices for CloudBees Core (Jenkins) on Kubernetes"]
part: 2
authors:
  - "Kurt Madel"
date: 2019-10-20T09:05:15-04:00
showDate: true
tags: ["Kubernetes","CI","CD","Core v2","security","Workload Identity","IAM Roles for Service Accounts","Jenkins","IAM","Cloud","CloudBees Core","GKE","EKS","AWS","GCP"]
photo: "/posts/best-practices-for-cloudbees-core-jenkins-on-kubernetes/bryce-canyon-wall-street.jpg"
photoCaption: "Wall Street, Bryce Canyon National Park, Utah<br>Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 250 37.1mm ƒ/4.9 1/100"
draft: false
---

In the second part of this series on [best practices for Jenkins (and CloudBees Core) on Kubernetes](/series/best-practices-for-cloudbees-core-jenkins-on-kubernetes/) we will continue to look at security. In this post we will look at how to reduce security risk of using cloud services from Jenkins Kubernetes agents, similar to how [the previous post in this series](/posts/best-practices-for-cloudbees-core-jenkins-on-kubernetes/core-psp/) showed how Kuberenetes Pod Security Policies can be used with Jenkins Kubernetes agents to limit the security risk of Jenkins agent containers.

## The Problem
I have already established in several other posts [why Kubernetes is an excellent platform for CD](https://kurtmadel.com/posts/native-kubernetes-continuous-delivery/native-k8s-cd/). However, accessing cloud services from Kubernetes CD jobs that require Identity Access Management (IAM) permissions for AWS, Azure and Google Cloud Platform (GCP), a typical step in many CD pipelines, presents a number of security challenges and usually falls short of [the principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege). Existing approaches for accessing cloud services requiring IAM permissions from Kubernetes based Jenkins agents have numerous security implications and complexities. Furthermore, the most commonly proposed solutions do not meet many organizations' security requirements. In addition to the security implications, some of these approaches aren't native cloud provider solutions and have to be self managed, and also have performance and reliabitliy issues.

There is a better way for providing **least privilege** IAM permissions to Jenkins Kubernetes agents - at least for the Amazon Elastic Kubernetes Service (EKS) and Google Kubernetes Engine (GKE). But before we look at a *better way*, I am going to review the security implications for some of the other most commonly used solutions.

## The Old Way

#### Create Cloud IAM credentials and store them as Jenkins Credentials or Kubernetes Secrets

First, lets walk throught the steps for setting up a GCP IAM Service Account Key ([a previously recommended approach for GKE](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform)) to use from a Jenkins Pipeline running on a [Kubernetes agent](https://github.com/jenkinsci/kubernetes-plugin#pipeline-support). For this example we are going to deploy a container image (from a public Google Container Registry (GCR)) to [Google Cloud Run](https://cloud.google.com/run/). The IAM Service Account will be created in the same GCP project where CloudBees Core (Jenkins) is running on the Google Kubernetes Engine (GKE).

1. Create an IAM Service Account with the **Cloud Run Admin** permissions to use the gcloud SDK to execute the `gcloud run deploy` command.
2. Export a JSON key file for that IAM Service Account.
3. Create a Kubernetes **Secret** with the contents of that key file in the same Kubernetes Namespace where the Jenkins Kubernetes agent will run.
4. Configure the Jenkins Kubernetes Pod template for the agent to mount the Kubernetes **Secret** for the IAM Service Account key file.
5. Inside the Jenkins Pipline, use the gcloud SDK to authenticate using the key file from the Kubernetes Secret.
6. Execute the gcloud command for deploying to Cloud Run.

*The Jenkins Kubernetes Pod template:*

```yaml
kind: Pod
metadata:
  name: cloud-run
spec:
  - name: gcloud
    image: google/cloud-sdk:252.0.0-slim
    command:
    - cat
    tty: true
    volumeMounts:
      - name: gcp-credential
        mountPath: /home/
    env:
      - name: GOOGLE_CLOUD_KEYFILE_JSON
        value: "/home/gcp-service.json"
  volumes:
    - name: gcp-credential
      secret:
        secretName: gcp-credential
```

*The Jenkins Pipeline:*

```groovy
pipeline {
  agent {
    kubernetes {
      label 'cloud-run'
      yamlFile 'pod.yml'
    }
  }
  stages {
    stage('Cloud Run Deploy') {
      steps {
        container('gcloud'){
          sh '''
            gcloud auth activate-service-account --key-file=$GOOGLE_CLOUD_KEYFILE_JSON
            gcloud beta run deploy bee-cd --image gcr.io/core-workshop/bee-cd:65 --allow-unauthenticated --platform managed --region us-east1
            echo "$(cat $GOOGLE_CLOUD_KEYFILE_JSON)" //Don't do this!
          '''
        }
      }
    }
  }
}
```

Some of the issues with this approach include:

1. These type of credentials are long-lived.
   - GCP IAM Service Account keys are valid for 10 years
   - AWS credentials (Access Key ID and Secret Access Key) - require custom rotation policies, but that is typically set to 90 days
2. These types of credentials are valid no matter where they are used, whether it is from a Jenkins agent running in EKS or GKE, or from the shell of a personal computer.
3. Storing these credentials as Jenkins Credentials or Kubernetes Secrets is inherently insecure. 
   - It is relatively straightforward to print out Jenkins Credentials or Kubernetes Secrets to Jenkins build logs in plain text.
   - Unless extra security precautions are taken, Kubernetes Secrets are typically stored as base64 encoded strings accessible by all Pods that run in that Namespace.
4. Many organizations won’t allow the use of these type of credentials in Jenkins, and for good reason.
5. The management overhead of inventory and rotation makes this a less than ideal method for authenticating.

### Associate an IAM object with a cloud instance and/or instance group (node pool)

1. All Kubernetes Pods created on the node/node pool with an instance profile will have access to the same set of cloud IAM permissions - regardless of the Kubernetes Namespace these Pods run in.
2. The principle of least privilege makes this method of authenticating less than ideal.

### Third-party solutions
For EKS: [kube2iam](https://github.com/jtblin/kube2iam) and [kiam](https://github.com/uswitch/kiam)

1. It isn't a cloud provider solution so good luck with support.
2. You have to install and manage it.
3. Performance issues. Kiam was specifically [created because of critical security and performance issues with kube2iam](https://www.bluematador.com/blog/iam-access-in-kubernetes-kube2iam-vs-kiam).
4. Require running a Kubernetes services that has the ability to provide all permissions you need across all Jenkins Pipeline jobs.

For GKE: [k8s-gke-service-account-assigner](https://github.com/imduffy15/k8s-gke-service-account-assigner)

1. Basically a rewrite of kube2iam for GKE with all the same issues listed above.


## A Better Way

Kubernetes v1.11 introduced [Service Account Token Volume Projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) and that feature became beta in v1.12. This allows projecting a temporary Kubernetes Service Account Token into a Pod and allows specifying the audience and validity duration. Furthermore, this projected Service Account token becomes invalid once the pod is deleted. 

AWS and GCP both created new offerings around this Kubernetes feature for their respective managed Kubernetes platforms. AWS created [IAM roles for service accounts](https://docs.aws.amazon.com/en_pv/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html) for EKS and Google Cloud created [Workload Identity](https://cloud.google.com/blog/products/containers-kubernetes/introducing-workload-identity-better-authentication-for-your-gke-applications) for GKE. Both offerings have a similar architecture that allow binding native cloud IAM permissions (via AWS IAM Roles and GCP IAM Service Accounts) to a specific Kubernetes Service Account in a specific Namespace.

These **bound** permissions have several advantages over the approaches mentioned above:

1. No file or passwords to store anywhere - no Jenkins Credentials, no Kubernetes Secrets.
2. Kubernetes Pods in a given Namespace and created with a specific Kubernetes ServiceAccount only have the cloud IAM permissions you want them to have and come closer than any other solution to achieving the principle of least privilege.
3. They are token based and the generated tokens can only be used from the Kubernetes Namespace and Service Account they are bound.
4. The tokens are short-lived and are destroyed when the Pod using it is destroyed.
5. The tokens are never actually exposed to the Jenkins Pipeline as they are integrated with the cloud provider SDKs for automatic authentication and authorization. No extra authentication step is necessary in your Jenkins Pipeline.


### Using with OSS Jenkins and CloudBees Core (Enterprise Jenkins)

These IAM to Kubernetes Service Account binding solutions can be used with OSS Jenkins, but they are even more effective when combined with CloudBees Core on Kubernetes. That is because [CloudBees Core provides dynamic provisiong and easy management of many team specific Jenkins Masters that we refer to as Managed Masters](https://docs.cloudbees.com/docs/cloudbees-core/latest/cloud-admin-guide/managing-masters) and [each Managed Master can be easily provisioned into its own Kubernertes Namespace](https://docs.cloudbees.com/docs/cloudbees-core/latest/cloud-admin-guide/managing). This allows you to utilize a default standard Kubernetes Cloud across all Masters that should have the same Kubernetes and IAM permissions, but also to easily manage Jenkins Master specific Kubernetes Cloud configurations for individual teams that need additional IAM permissions that you don't want all teams to have. 

>NOTE: If you do want to have multiple teams on one Jenkins Master you can create multiple Jenkins Kubernetes Cloud configurations - each in its own Kubernetes Namespace - and then leverage the Kubernetes plugin capability to [restrict pipeline support to authorized folders](https://github.com/jenkinsci/kubernetes-plugin#restricting-what-jobs-can-use-your-configured-cloud). You will then have to apply proper RBAC configuration so that only the users you want have access to configure folders to use one or more protected Jenkins Kubernetes clouds, and you will have to create the jobs that need to use the more permissive Kubernetes cloud configuration in those folders.

The default configuration for the [Jenkins Kubernetes Cloud plugin](https://github.com/jenkinsci/kubernetes-plugin) uses the same Namespace and Kubernetes Service Account as the Jenkins Master it is configured for - when the Jenkins Master is also running on Kubernetes. This is typically the case with Core on Kubernetes and also for OSS Jenkins. To fully leverage binding IAM permissions to Kubernetes Service Accounts we must revisit how we set-up and use the Jenkins Kubernetes Clouds for agents.

#### Create a unique Kubernetes Namespace and ServiceAccount for each Managed Master and Kubernetes Cloud
Placing each of your Jenkins Masters into their own Kubernetes Namespaces provides and extra layer of security that isn't just limited to binding IAM credentials - it also protects Kubernetes Secrets from other Jenkins Masters and this will provide a more secure integration for managing Jenkins credentials with JCasC and Kubernetes Secrets. Creating a unique Kubernetes Cloud per Jenkins Master can be [managed more easily with JCasC](https://github.com/kypseli/workshop-mm-jcasc/blob/ops/jcasc.yml), as we will manage the Kubernetes cloud configuration and the Jenkins credential used to connect to Kubernetes for the Master specific cloud. The Jenkins credential used for the Kubernetes Cloud configuration will be the Kubernetes Service Account token stored in Jenkins as a Secret Text credential - stored at the Master level (not CloudBees Core Operations Center), so only that Jenkins Master has access to it. The Kubernetes Service Account token should be managed as a Kubernetes Secret in the same Namespace as the Managed Master is created so it can be dynamically injected into the JCasC configuration for that Master. No other Core Managed Masters will have access to this Kubernetes Secret.

>NOTE: This approach depends on managing what Kuberentes Namespaces are used by Managed Masters. You must not allow untrusted users to have access to configure Managed Masters on your Core Operations Center, otherwise they could use a Namespace they shouldn't have access to - that is why [I prefer to do it as code](https://github.com/kypseli/demo-mm-jcasc/blob/cloud-run/groovy/createManagedMaster.groovy).

#### Example: Using Workload Identity with a Core Managed Master to Deploy a Container Image to Google Cloud Run

1. [Enable Workload Identity for your GKE cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#enable_workload_identity_on_an_existing_cluster) - this only has to be done once per cluster.
2. Create an IAM Service Account with the **Cloud Run Admin** permissions that will allow us to use the [gcloud SDK](https://cloud.google.com/sdk/) to execute the `gcloud run deploy` command.
3. Create a new Kubernetes Namespace and Service Account that will be unique to the Core Managed Master. Note the [`automountServiceAccountToken`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server) is set to `false` - this will require that you configure the Jenkins Kubernetes cloud with a **credential** for the same Kubernetes Service Account that is used to create the Managed Master as it will no longer have the Service Account token automatically mounted to its Pod.
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
      automountServiceAccountToken: false
      ```
4. Bind the GCP IAM Service Account (`core-cloud-run@core-workshop.iam.gserviceaccount.com` in the example below) to the GKE Namespace and Service Account (`cloud-run` and `cloud-run-sa` in the example below):

      ```shell
      gcloud iam service-accounts add-iam-policy-binding \
        --role roles/iam.workloadIdentityUser \
        --member "serviceAccount:core-workshop.svc.id.goog[cloud-run/cloud-run-sa]" \
        core-cloud-run@core-workshop.iam.gserviceaccount.com
      ```
5. [Create a Managed Master in the Master specific Kubernetes Namespace](https://docs.cloudbees.com/docs/cloudbees-core/latest/cloud-admin-guide/managing) - here is [an example of automating this with code](https://github.com/kypseli/demo-mm-jcasc/tree/cloud-run). Here is an example of a Managed Master Kubernetes yaml configuration that specifies a unique Kubernetes Service Account - note the `serviceAccount` value matches the Kuberenes Service Account we created above:

      ```yaml
      ---
      kind: StatefulSet
      spec:
        template:
          metadata:
            annotations:
                cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
          spec:
            containers:
            - name: jenkins
              env:
                # With the help of SECRETS environment variable
                # we point Jenkins Configuration as Code plugin the location of the secrets
                - name: SECRETS
                  value: /var/jenkins_home/mm-secrets
                - name: CASC_JENKINS_CONFIG
                  value: https://raw.githubusercontent.com/kypseli/demo-mm-jcasc/cloud-run/jcasc.yml
              volumeMounts:
              - name: mm-secrets
                mountPath: /var/jenkins_home/mm-secrets
                readOnly: true
            volumes:
            - name: mm-secrets
              secret:
                secretName: mm-secrets
            nodeSelector:
              type: master
            serviceAccount: cloud-run-sa
            serviceAccountName: cloud-run-sa
            securityContext:
              runAsUser: 1000
              fsGroup: 1000 
      ```

6. Create [a Master specific Jenkins Kubernetes cloud configuration](https://github.com/kypseli/demo-mm-jcasc/blob/cloud-run/jcasc.yml#L8) that uses the Master specific Namespace and Kubernetes Service Account. To ensure that we are using the `cloud-run-sa` Kubernetes Service Account for all Pod based agents we [explicitly set this on a Kubernetes cloud Pod template](https://github.com/kypseli/demo-mm-jcasc/blob/cloud-run/jcasc.yml#L36) that is [configured as the `defaultsProviderTemplate`](https://github.com/kypseli/demo-mm-jcasc/blob/cloud-run/jcasc.yml#L12). 
7. Update the `cloud-run-sa` Kubernetes Service Account with the `iam.gke.io/gcp-service-account` annotation:

      ```shell
      kubectl annotate serviceaccount \
        --namespace cloud-run \
        cloud-run-sa \
        iam.gke.io/gcp-service-account=core-cloud-run@core-workshop.iam.gserviceaccount.com
      ```

8. Create a yaml file based Pod template for the GCP gcloud SDK ([source on GitHub](https://github.com/kypseli/core-cloud-run-example/blob/master/pod.yml)):

      ```yaml
      kind: Pod
      metadata:
        name: cloud-run-pod
        namespace: cloud-run
      spec:
        serviceAccountName: cloud-run-sa
        containers:
        - name: gcp-sdk
          image: google/cloud-sdk:252.0.0-slim
          command:
          - cat
          tty: true
      ```

>NOTE: If you look at the example on GitHub you will notice that it also specifies the `runAsUser` and mounts an `emptyDir` volume at the path `/.config/gcloud/logs` - this is because we have enabled Pod Security Policies on the GKE cluster we are using for this example and we don't allow Pods to run as the `root` (0) user. You can read more about [using Pod Security Policies with CloudBees Core Jenkins in the previous post of this series](/posts/best-practices-for-cloudbees-core-jenkins-on-kubernetes/core-psp/).

1. Create a Jenkins Declarative Pipeline that uses the above Pod template and executes the gcloud Cloud Run deploy command:

      ```groovy
      pipeline {
        agent {
          kubernetes {
            label 'cloud-run'
            yamlFile 'pod.yml'
          }
        }
        stages {
          stage('Cloud Run Deploy') {
            steps {
              container('gcp-sdk'){
                sh 'gcloud beta run deploy bee-cd --image gcr.io/core-workshop/bee-cd:65 --allow-unauthenticated --platform managed --region us-east1'
              }
            }
          }
        }
      }
      ```

>NOTE: There is no authentication step needed as the glcoud SDK automatically authenticates with the token provided by  Workload Identity. The AWS SDK also doesn't require an explicit assume role step or authentication step.

One security limitation of the Jenkins Kubernetes plugin is that you can define Pod templates as standalone yaml files to include specifying whatever Kubernetes Namespace and Service Account that you want (this is also a very nice feature for managing your agent Pod template configuration as code). But if we were to create a Pod template specifying the `cloud-run` Namespaced and `cloud-run-sa` Service Account it will only work from a Master with a Jenkins Kubernetes cloud that is configured with the Namespace and Service Account token that has permissions to create Pods in the `cloud-run` Namespace. Running it from unauthorized Jenkins Kubernetes clouds will result in the following error:

```shell
io.fabric8.kubernetes.client.KubernetesClientException: 
Failure executing: POST at: https://10.11.240.1/api/v1/namespaces/cloud-run/pods. 
Message: pods is forbidden: User "system:serviceaccount:core-demo:jenkins" cannot create resource "pods" in API group "" in the namespace "cloud-run".
```

## A Better Way, But Not Perfect

I know this seems like a lot of steps, but once you have IAM permissions bound to a Kubernetes Service Account set up you can easily update the GCP IAM Service Account with additional permissions, and all the Pods launched with the Master specfic Namespace/Service Account will instantly have access to those permissions. And for all of the different solutions for providing cloud IAM permissions to Jenkins Kubernetes based agents - these new IAM binding solutions from AWS and GCP come closest to achieving the principle of least privilege.

