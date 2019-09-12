---
authors:
  - "Matt Elgin"
title: "Self-Updating Jenkins: GitOps for Jenkins Configuration"
date: 2019-07-03T17:00:00-04:00
showDate: true
draft: false
tags: ["jenkins","cloudbees jenkins distribution","gitops","kubernetes","kaniko","docker","cert-manager","google cloud platform"]
---

In this blog post, we'll walk through creating a self-updating instance of the [CloudBees Jenkins Distribution](https://www.cloudbees.com/products/cloudbees-jenkins-distribution), with all configuration stored as code in a GitHub repository.

We'll deploy the CJD master as a `StatefulSet` in a Kubernetes cluster, configure the master using the [Jenkins Configuration as Code plugin](https://github.com/jenkinsci/configuration-as-code-plugin), and set up a TLS certificate through [cert-manager](https://github.com/jetstack/cert-manager). Finally, we'll seed a Pipeline job that updates the master upon commit to the [Git repository](https://github.com/cb-technologists/cjd-casc) that contains the configuration - enabling GitOps for Jenkins itself.

| UPD (Sep 12, 2019): Jenkins Configuration as Code plugin is now supported in [CloudBees Jenkins Distribution](https://www.cloudbees.com/products/cloudbees-jenkins-distribution) and [CloudBees Jenkins Support](https://www.cloudbees.com/products/cloudbees-jenkins-support). See [Administering CJD: Configuration as Code](https://go.cloudbees.com/docs/cloudbees-jenkins-distribution/distro-admin-guide/configuration-as-code/) for usage guidelines and quick start. You can also find an official demo [here](https://github.com/cloudbees-oss/cjd-jcasc-demo). For information about other CloudBees products, please see [this page](https://support.cloudbees.com/hc/en-us/articles/360031191471-State-of-Jenkins-Configuration-as-Code-JCasC-support-in-CloudBees-products). |
| --- |

## Deploying CloudBees Jenkins Distribution in Kubernetes
First, we'll need to deploy a Jenkins instance into a Kubernetes cluster. In this case, we'll use [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) to deploy a containerized version of CJD. To provision a cluster, we'll follow the Google Cloud documentation [here](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster). (**Note:** this blog post assumes prior installation of and basic familiarity with using `kubectl` to interact with a Kubernetes cluster.)

Once the cluster has been provisioned and `kubectl` has been configured, we'll create a dedicated `namespace` for our CJD resources and update our `kubectl config` to use it by default:

```bash
kubectl create namespace cjd
kubectl config set-context $(kubectl config current-context) --namespace cjd
```

We'll also need to ensure an ingress controller is deployed within the cluster. For this post, we'll assume the use of the [NGINX ingress controller](https://kubernetes.github.io/ingress-nginx/). Following the [Installation Guide](https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md), we'll manually deploy using a few `kubectl` commands:

```bash
# grant cluster-admin to user
kubectl create clusterrolebinding cluster-admin-binding \ --clusterrole cluster-admin \ --user $(gcloud config get-value account)
# deploy nginx ingress controller resources
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml
```

Next, let's look at the manifest file that will deploy the necessary resources for CJD using the [cjd.yaml](https://github.com/cb-technologists/cjd-casc/blob/master/cjd.yaml) manifest file.

First, we create a `ServiceAccount`, a `Role` with the necessary permissions to manage agents and perform the required update actions, and a `RoleBinding` to connect the two.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cjd

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cjd
rules:
- apiGroups: [""]
  resources: ["pods","configmaps","services","serviceaccounts"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles","rolebindings"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["create","delete","get","list","patch","update","watch"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: cjd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cjd
subjects:
- kind: ServiceAccount
  name: cjd
```

Next, we create a `Service` that exposes ports for access to the CJD web interface and for master-agent communication:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: cjd
spec:
  selector:
    app: cjd
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: agent
    port: 50000
    protocol: TCP
    targetPort: 50000
  type: ClusterIP
```

Next, we set up an `Ingress` to allow access to our CJD instance from outside of the cluster. We'll examine this in more detail in a later section where we walk through the setup of `cert-manager`.

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cjd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    certmanager.k8s.io/issuer: "letsencrypt-prod" # add after cert-manager deploy
    certmanager.k8s.io/acme-challenge-type: http01 # add after cert-manager deploy
spec:
  tls: # cert-manager
  - hosts: # cert-manager
    - cjd.cloudbees.elgin.io # cert-manager
    secretName: cjd-tls # cert-manager
  rules:
  - host: cjd.cloudbees.elgin.io
    http:
      paths:
      - path: /
        backend:
          serviceName: cjd
          servicePort: 80
```

We'll also need to make sure that we create a DNS A Record through our hosting provider that maps our `host` URL to the `EXTERNAL-IP` of our ingress controller. We can get that IP after deploying our NGINX ingress controller by running:

```bash
kubectl get svc -n ingress-nginx
```

Finally, we provision the `StatefulSet` that controls the CJD Pod and `PersistentVolumeClaim`. The container image we use here is a custom image inheriting from the [official CJD Docker image](https://hub.docker.com/r/cloudbees/cloudbees-jenkins-distribution/). We'll examine the `Dockerfile` for this image in the next section, when we detail the configuration.

Additionally, you'll notice the creation of a few `secretRef` environment variables, as well as the setting of the `CASC_JENKINS_CONFIG` environment variable and the mounting of a `jenkins-casc` `ConfigMap` - these again will be expanded upon in the configuration section.

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cjd
spec:
  selector:
   matchLabels:
     app: cjd
  serviceName: "cjd"
  template:
    metadata:
      labels:
        app: cjd
    spec:
      containers:
      - name: cjd
        image: gcr.io/melgin/cjd-casc:d176f38b289d0437a2503c83af473f57b25a4d26
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        - containerPort: 50000
        env:
        - name: CASC_JENKINS_CONFIG
          value: /var/jenkins_config/jenkins-casc.yaml
        envFrom:
          - secretRef:
              name: github
          - secretRef:
              name: url
          - secretRef:
              name: github-oauth
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home/
        - name: jenkins-casc
          mountPath: /var/jenkins_config/
      securityContext:
        fsGroup: 1000
      serviceAccountName: cjd
      volumes:
      - name: jenkins-casc
        configMap:
          name: jenkins-casc
  volumeClaimTemplates:
  - metadata:
      name: jenkins-home
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

## Configuring with Jenkins Configuration-as-Code Plugin

With the YAML for the required CJD Kubernetes resources laid out, we'll now go into the code handling the configuration of the master. While detailing the `StatefulSet` above, we mentioned that a custom Docker image is used for the CJD container. The `Dockerfile` for this image can be found below:

```Dockerfile
FROM cloudbees/cloudbees-jenkins-distribution:2.164.3.2

LABEL maintainer "melgin@cloudbees.com"

ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

USER root

RUN echo 2.0 > /usr/share/jenkins/ref/jenkins.install.UpgradeWizard.state

ENV TZ="/usr/share/zoneinfo/America/New_York"

ENV JENKINS_UC https://jenkins-updates.cloudbees.com
# add environment variable to point to configuration file
ENV CASC_JENKINS_CONFIG /usr/jenkins_config/jenkins-casc.yaml

# Install plugins
ADD https://raw.githubusercontent.com/jenkinsci/docker/master/install-plugins.sh /usr/local/bin/install-plugins.sh
RUN chmod 755 /usr/local/bin/install-plugins.sh
ADD https://raw.githubusercontent.com/jenkinsci/docker/master/jenkins-support /usr/local/bin/jenkins-support
RUN chmod 755 /usr/local/bin/jenkins-support
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN bash /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER jenkins
```

In this `Dockerfile`, we add custom configuration to the official CJD Docker image. We first set the `JENKINS_UC` environment variable to use the CloudBees update center, as well as the `CASC_JENKINS_CONFIG` variable to point to the location we'll mount our configuration file. Finally, we leverage the [Jenkins Docker `install-plugins.sh` script](https://github.com/jenkinsci/docker#preinstalling-plugins) to install a list of plugins from our `plugins.txt` file. These plugins include:

```txt
configuration-as-code:1.20
job-dsl:1.74
kubernetes:1.14.9
kubernetes-credentials:0.4.0
credentials:2.2.0
workflow-multibranch:2.20
github-branch-source:2.4.5
workflow-aggregator:2.5
blueocean:1.10.2
github-oauth:0.32
```

This will handle the initial installation of the plugins we need, including resolving any dependencies.

Next, we'll need to use the Configuration as Code plugin to handle the configuration of the master itself. To do so, we'll mount the configuration YAML as a `ConfigMap` that our CJD `StatefulSet` will use. Here's what our `jenkinsCasc.yaml` file looks like:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-casc
data:
  jenkins-casc.yaml: |
    jenkins:
      agentProtocols:
      - "Diagnostic-Ping"
      - "JNLP4-connect"
      - "Ping"
      crumbIssuer:
        standard:
          excludeClientIPFromCrumb: false
      securityRealm:
        github:
          githubWebUri: "https://github.com"
          githubApiUri: "https://api.github.com"
          clientID: "${CLIENT_ID}"
          clientSecret: "${CLIENT_SECRET}"
          oauthScopes: "read:org,user:email"
      systemMessage: "CJD in Kubernetes configured as code!"
      clouds:
      - kubernetes:
          name: kubernetes
          jenkinsUrl: http://cjd
          containerCapStr: 100
      authorizationStrategy:
        loggedInUsersCanDoAnything:
          allowAnonymousRead: false
    credentials:
      system:
        domainCredentials:
          - credentials:
            - usernamePassword:
                scope: GLOBAL
                id: "github"
                description: "GitHub API token"
                username: ${username}
                password: ${token}
    jobs:
    - script: >
        multibranchPipelineJob('cjd-casc') {
          branchSources {
            github {
              scanCredentialsId('github')
              repoOwner('cb-technologists')
              repository('cjd-casc')
            }
          }
          orphanedItemStrategy {
            discardOldItems {
              numToKeep(5)
            }
          }
        }
    security:
      remotingCLI:
        enabled: false
    unclassified:
      location:
        adminAddress: "address not configured yet <nobody@nowhere>"
        url: "https://cjd.cloudbees.elgin.io/"
```

This config file sets up a handful of basic Jenkins settings like allowed agent protocols, security settings, and an example system message.

Three config items in particular are worth additional exploration. First, the security realm is set to use a GitHub organization for authentication (see [the Jenkins GitHub OAuth Plugin page](https://wiki.jenkins.io/display/JENKINS/GitHub+OAuth+Plugin) for details on setting up a GitHub OAuth application). To avoid hardcoding our Client ID and Client Secret in our GitHub repository, we take advantage of Kubernetes `Secrets`.

Recall from our `StatefulSet` above that we load a few environment variables from `Secrets`. These include our GitHub OAuth application ID & secret, as well as the username and API token used by our Pipeline job to communicate with our repository.

To create these, we use the following `kubectl` commands (replacing the placeholder variables with the actual credentials):

```bash
kubectl create secret generic github-oauth --from-literal=CLIENT_ID=${CLIENT_ID} --from-literal=CLIENT_SECRET=${CLIENT_SECRET}

kubectl create secret generic github --from-literal=username=${USERNAME} --from-literal=token=${TOKEN}
```

The second config item to note is the creation of a simple Kubernetes cloud that our master will use for provisioning pod template agents using the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin).

The third and final detail to call out is the `jobs` section, which uses the [Job DSL plugin](https://github.com/jenkinsci/job-dsl-plugin) to seed a Multibranch Pipeline job. The Jenkinsfile for this Pipeline is stored in the same GitHub repository as the rest of our config files. We'll detail the contents of this Pipeline script in a later section.

To apply this configuration, we apply the `ConfigMap` manifest file to our cluster:

```bash
kubectl apply -f jenkinsCasc.yaml
```
With our `ConfigMap` and related `Secrets` created, we can now apply the manifest file from the previous section to deploy the remainder of the CJD resources:

```bash
kubectl apply -f cjd.yaml
```

## Securing with cert-manager

At this point, our CJD instance is not accessible through HTTPS. To remedy this and enhance the security of our environment, we'll be using [`cert-manager`](https://docs.cert-manager.io/en/latest/), a Kubernetes tool used to automate the management of certificates within a cluster. In this case, we'll use it to manage our TLS certificate issuance from [Let's Encrypt](https://letsencrypt.org/).

Our setup process for `cert-manager` loosely follows their [Quick-Start guide](https://github.com/jetstack/cert-manager/blob/master/docs/tutorials/acme/quick-start/index.rst). Because we've already configured an ingress controller with a corresponding DNS entry along with deploying the CJD resources, we can [ensure Helm](https://github.com/jetstack/cert-manager/blob/master/docs/tutorials/acme/quick-start/index.rst#step-0---install-helm-client) [& Tiller](https://github.com/jetstack/cert-manager/blob/master/docs/tutorials/acme/quick-start/index.rst#step-1---installer-tiller) are installed on the cluster, then skip to the [step of actually deploying `cert-manager`](https://github.com/jetstack/cert-manager/blob/master/docs/tutorials/acme/quick-start/index.rst#step-5---deploy-cert-manager).

Once `cert-manager` has been deployed in its new `namespace`, we'll next need to deploy the `Issuer` to our `cjd` `namespace`.

>**Note**: on initial setup of `cert-manager`, it's probably prudent to heed the Quick-Start's recommendation to create a staging `Issuer` first to minimize the risk of being rate limited by Let's Encrypt. For brevity, we'll only walk through the production `Issuer` creation here.

Using the provided [example `Issuer` manifest file](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.8/docs/tutorials/acme/quick-start/example/production-issuer.yaml), we'll swap in our actual email address before creating the resource in our `cjd` `namespace`:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: melgin@cloudbees.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    http01: {}
```

```bash
kubectl apply -f production-issuer.yaml
```

Once created, this `Issuer` relies on annotations on our `Ingress` to manage the TLS certificate creation. Recall that we briefly discussed the `Ingress` manifest in a previous section:

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cjd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    certmanager.k8s.io/issuer: "letsencrypt-prod" # add after cert-manager deploy
    certmanager.k8s.io/acme-challenge-type: http01 # add after cert-manager deploy
spec:
  tls: # cert-manager
  - hosts: # cert-manager
    - cjd.cloudbees.elgin.io # cert-manager
    secretName: cjd-tls # cert-manager
  rules:
  - host: cjd.cloudbees.elgin.io
    http:
      paths:
      - path: /
        backend:
          serviceName: cjd
          servicePort: 80
```

The lines with comments referencing `cert-manager` are required for the TLS certificate to be successfully issued. These include specifying the `Issuer`, the challenge type, as well as the hostname and `secretName`.

You can confirm that the certificate has been successfully issued by running `kubectl get certificate` and verifying that `READY` is `True` for our `cjd-tls` certificate. Once this process has been completed, CJD should now be accessible via HTTPS.

## Writing the update Pipeline job

With CJD now running in our cluster and accessible via HTTPS, we'll next take a look at the Pipeline script that will handle the update of the master. At a high-level, we need our Pipeline to accomplish two major tasks: 

1. build and push our Docker image whenever a change is pushed to the GitHub repository, and
2. update our Kubernetes resources with the newly built Docker image and any additional changes.

We represent these two procedures as stages within our Pipeline script. 

For the first stage, we will use [kaniko](https://github.com/GoogleContainerTools/kaniko) to build and push our Docker image to [Google Container Registry](https://cloud.google.com/container-registry/). Because we'll be using different agents for each stage, we'll start the Pipeline with `agent none`. Within the first stage, we define our agent using YAML, which specifies the [Google-provided kaniko image](https://gcr.io/kaniko-project/executor:debug) as the container we will use. 

To use kaniko, we'll first need to [follow this kaniko documentation](https://github.com/GoogleContainerTools/kaniko#kubernetes-secret) to create a Google Cloud service account with appropriate permissions and download the related JSON key. Assuming we've renamed the key `kaniko-secret.json`, we can [follow this procedure from Heptio](http://docs.heptio.com/content/private-registries/pr-gcr.html) to create another Kubernetes `Secret` to allow for authentication to Google Container Registry (again replacing the placeholder email with the real service account email address):

```bash
kubectl create secret docker-registry gcr-secret \
    --docker-server=https://gcr.io \
    --docker-username=_json_key \
    --docker-email=${SERVICE_ACCOUNT@PROJECT.iam.gserviceaccount.com} \
    --docker-password="$(cat kaniko-secret.json)"
```

Within the `step` block, we are accomplishing two main things:
1. In the default `jnlp` container, we store the specific Git commit ID that triggered the build as an environment variable
2. In the `kaniko` container, we build and push our latest Docker image, tagging it with the commit ID we just stored.

```groovy
pipeline {
  agent none
  stages {
    stage('Build and push with kaniko') {
      agent {
        kubernetes {
          label "kaniko-${UUID.randomUUID().toString()}"
          yaml """
kind: Pod
metadata:
  name: kaniko
spec:
  serviceAccountName: cjd
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug-v0.10.0
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: jenkins-docker-cfg
        mountPath: /kaniko/.docker
  volumes:
  - name: jenkins-docker-cfg
    projected:
      sources:
      - secret:
          name: gcr-secret
          items:
            - key: .dockerconfigjson
              path: config.json
"""
        }
      }
      environment {
        PATH = "/busybox:/kaniko:$PATH"
      }
      steps {
        container('jnlp') {
          script {
              env.COMMIT_ID = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
          }
        }
        container(name: 'kaniko', shell: '/busybox/sh') {
          sh """#!/busybox/sh
                /kaniko/executor --context `pwd` --destination gcr.io/melgin/cjd-casc:${env.COMMIT_ID} --cache=true
          """
        }
      }
    }
```

In the subsequent stage, we now apply changes to our CJD configuration to the resources running in our Kubernetes cluster.

First, we use a `when` directive to ensure we only run this stage when the Pipeline is running off of the *master* branch. We then use the [Google-provided kubectl image](https://gcr.io/cloud-builders/kubectl) for our stage agent pod template. Within this container, we apply changes to our `jenkins-casc` `ConfigMap`, the resources specified in `cjd.yaml`, and finally set the image for our CJD `StatefulSet` to the latest one we've just pushed to Google Container Registry:

```groovy
    stage('Update CJD') {
      when {
        beforeAgent true
        branch 'master'
      }
      agent {
        kubernetes {
          label "kubectl-${UUID.randomUUID().toString()}"
          yaml """
kind: Pod
metadata:
  name: kubectl
spec:
  serviceAccountName: cjd
  containers:
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl@sha256:50de93675e6a9e121aad953658b537d01464cba0e4a3c648dbfc89241bb2085e
    imagePullPolicy: Always
    command:
    - cat
    tty: true
"""
        }
      }
      steps {
        container('kubectl') {
          sh """
            kubectl apply -f jenkinsCasc.yaml
            kubectl apply -f cjd.yaml
            kubectl set image statefulset cjd cjd=gcr.io/melgin/cjd-casc:${env.COMMIT_ID}
          """
        }
      }
    }
  }
}
```

To ensure the `cjd-casc` Pipeline job is triggered automatically upon each commit or pull request, we need to ensure a webhook is setup within the GitHub repository following [this process](https://support.cloudbees.com/hc/en-us/articles/224543927-GitHub-Integration-Webhooks).

With this in place, we now have all of our Jenkins configuration stored as code in our GitHub repository, including the process for updating the configuration. Whenever a change is pushed to the repository, those changes will automatically be applied to our Jenkins master.

![successful run of cjd-casc Pipeline](/img/cjd-casc/cjd-casc-pipeline.png)

## Further enhancements

This approach moves us much closer to the practice of GitOps for our Jenkins configuration. However, there are certainly areas for enhancement going forward. A few immediate examples that come to mind include:

- Non-master branch Pipeline runs could deploy the CJD resources & config to a staging `namespace`. This would allow for the vetting of changes in a non-production environment before merging to master - a workflow critical for use in any scenario supporting mission-critical workloads.
- Some level of smoke testing should be introduced for either/both of the non-prod/prod `namespaces` as a third Pipeline stage. This could range from a simple `curl` command to check the Jenkins system message in order to verify Jenkins is up and running, all the way to more complex cases that verify the latest configuration has been appropriately applied.
- `post` blocks could be introduced for notification to the appropriate Slack channel, email list, etc., that a Jenkins update has commenced/succeeded/failed.
- Right now, the Docker image is rebuilt on every Pipeline run - even if no changes have been committed to the `Dockerfile` or related files. While caching is in place, it would be even more efficient to check for changes to those specific files, then selectively skip or run the `Build and push with kaniko` stage (though this does somewhat complicate the tagging of the Docker image each time a commit triggers a build).
