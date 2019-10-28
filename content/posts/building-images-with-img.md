---
title: "Unprivileged Container Image Builds with img and Jenkins on Kubernetes"
authors:
  - "Matt Elgin"
date: 2019-10-28T00:00:00-00:00
showDate: true
tags: ["kubernetes","img","docker","jenkins","containers","google kubernetes engine","workload identity"]
draft: true
---

## Why use `img` for container builds?
As more organizations turn to containers and Kubernetes to manage their CI/CD workloads, numerous strategies have emerged to handle the actual building of container images within these containerized environments. However, each of these approaches have not been without their security drawbacks (see [Kurt Madel](/authors/kurt-madel/)'s recent post on ["Securely Building Container Images on Kubernetes"](/posts/build-continaer-images/) for a rundown of these approaches and their security implications.)

The current choice for many teams is Google's [kaniko](https://github.com/GoogleContainerTools/kaniko) project. This is a good option for many organizations (especially when combined with Kurt's recommendations for `PodSecurityPolicies`). However, it's worth noting that while `kaniko` itself does not require running as root, it will [require that privilege for most significant container building](https://github.com/GoogleContainerTools/kaniko#security). While this is an acceptable caveat for some organizations, it can be a deal-breaker for others.

Fortunately, efforts have been underway to create an unprivileged, non-root container builder. Jessie Frazelle introduced [`img`](https://github.com/genuinetools/img), one such tool, in her blog post ["Building Container Images Securely on Kubernetes"](https://blog.jessfraz.com/post/building-container-images-securely-on-kubernetes/). I'll leave the details of design approach and usage to her post and the GitHub project page, but there are a few points I'll highlight here:

1. `img` is intended to run as a non-root user such as UID 1000.
2. `img` can be run without requiring the `--privileged` Docker flag or the equivalent `privileged: true` security context in Kubernetes.
3. Syntax for building, pushing, and pulling images, among other actions, largely mirror Docker's - for example, `img build -t hello-world .` and `img push hello-world` are the commands to build and push an image called `hello-world`.

In this post, we'll dive into using `img` as a `Pod` within a Google Kubernetes Engine (GKE) cluster, integrating it with Google Cloud services, and running it from a Jenkins Pipeline to automate our build and push workflow. We'll also discuss some of the security implications of `img` as they relate to running in Kubernetes.

## Running `img` in Kubernetes
Jessie's blog post includes a sample YAML manifest file (with a related [Docker container](https://r.j3ss.co/repo/img/tags)) for deploying `img` as a Kubernetes Pod, which we'll use as a starting point. However, we also want to leverage [Google Cloud's Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) to seamlessly integrate with Google Container Registry, so we'll create a custom Docker image that includes both `img` and `gcloud`. (For a deeper dive into using Workload Identity to securely tie Kubernetes Service Accounts to cloud IAM permissions, see [part 2 of Kurt's series on best practices for Jenkins in Kubernetes](/posts/best-practices-for-cloudbees-core-jenkins-on-kubernetes/securely-using-cloud-services-from-jenkins-kubernetes-agents/).)

Here's our `Dockerfile` for this combined image:
```Dockerfile
FROM gcr.io/cloud-builders/gcloud-slim:latest AS gcloud

FROM r.j3ss.co/img:v0.5.7 AS img
USER root
# install Python - gcloud dependency
RUN apk add python
# copy google-cloud-sdk to img image
COPY --from=gcloud /builder/google-cloud-sdk /home/user/google-cloud-sdk

USER user

ENV PATH "$PATH:/home/user/google-cloud-sdk/bin/"
```

This image will allow us to access both `img` and `gcloud` commands from the same container, which will allow us to securely authenticate to our registry.

Next, before creating our `Pod`, we need to set up our Google Service Account to support Workload Identity. We'll create the Google Service Account (`img-gcr`, in this example), add the Storage Admin role (to allow pushing & pulling to our GCR repository), and bind that Google Service Account to the corresponding Kubernetes `ServiceAccount`. 
```bash
# create GSA (if not already created)
gcloud iam service-accounts create img-gcr

# grant Storage Admin permissions to GSA
gcloud projects add-iam-policy-binding melgin \
  --member serviceAccount:img-gcr@melgin.iam.gserviceaccount.com \
  --role roles/storage.admin

# create namespace
kubectl create namespace img

# create KSA and annotate
kubectl create serviceaccount --namespace img img

kubectl annotate serviceaccount --namespace img img \
  iam.gke.io/gcp-service-account=img-gcr@melgin.iam.gserviceaccount.com

# bind GSA to KSA
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:melgin.svc.id.goog[img/img]" \
  img-gcr@melgin.iam.gserviceaccount.com
```

With our service accounts configured both in Google Cloud and our Kubernetes cluster, we can now create our `PodSecurityPolicy`:
```yaml
# based on https://github.com/kypseli/cb-core-oc-workshop/blob/master/k8s/cb-core-psp.yml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: img
  namespace: img
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'unconfined,runtime/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'unconfined,runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  allowPrivilegeEscalation: true
  allowedProcMountTypes:
  - Unmasked
  - Default
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  hostPID: false
  hostIPC: false
  hostNetwork: false
  privileged: false
  runAsUser:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - 'emptyDir'
  - 'secret'
  - 'downwardAPI'
  - 'configMap'
  - 'persistentVolumeClaim'
  - 'projected'
```

There are a few details worth calling out in this policy:
1. We use the `runAsUser` specification to ensure our `img` Pod runs as a non-zero/non-root user.
2. `privileged` is set to `false`. However, it's worth noting that `allowPrivilegeEscalation` must be set to true for `img` to execute commands properly.
3. For `img` to run properly, both `seccomp` and `AppArmor` profiles must be set to `unconfined`, which is [required by `runc`](https://github.com/genuinetools/img#running-with-docker).
4. We include `allowedProcMountTypes` with both `Unmasked` and `Default` as accepted values - more on this in a minute.

Next, we're going to attach our `img` `ServiceAccount` to a corresponding `Role` with a `RoleBinding`. This will allow our `img` `Pod` to use the custom `PodSecurityPolicy` we just created.
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: img
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - img
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: img
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: img
subjects:
- kind: ServiceAccount
  name: img
```

Finally, here is our `Pod` definition:
```yaml
# based on https://blog.jessfraz.com/post/building-container-images-securely-on-kubernetes/
apiVersion: v1
kind: Pod
metadata:
  name: img
  labels:
    run: img
  annotations:
    container.apparmor.security.beta.kubernetes.io/img: unconfined
    container.seccomp.security.alpha.kubernetes.io/img: unconfined
spec:
  securityContext:
    runAsUser: 1000
  serviceAccountName: img
  containers:
  - name: img
    image: gcr.io/melgin/gcloud-img:0.1.1
    imagePullPolicy: Always
    command:
    - cat
    tty: true
    # securityContext:
    #   procMount: Unmasked
    volumeMounts:
    - name: docker-config
      mountPath: /.docker/
    - name: gcloud-config
      mountPath: /.config/gcloud
    - name: cache-volume
      mountPath: /tmp/
  volumes:
  - name: docker-config
    emptyDir: {}
  - name: gcloud-config
    emptyDir: {}
  - name: cache-volume
    emptyDir: {}
  restartPolicy: Never
  
```

Again, a few items to focus on here:
1. As noted above, we include annotations that set our `seccomp` and `AppArmor` to `unconfined`. 
2. Our `Pod` `securityContext` specifies running as user 1000.
3. On the container level, we include a `securityContext` that sets `procMount: Unmasked`. However, you'll notice that we currently have these two lines commented out.

### What's going on with `procMount: Unmasked`?
As mentioned in the [`img` GitHub README](https://github.com/genuinetools/img#running-with-kubernetes), setting `securityContext.procMount` to `Unmasked` is no longer required to run `img`, but it does enable PID namespace isolation. Essentially, this prevents child processes from executing `kill -2` against the parent `img` process. This `procMount` option was introduced by a series of patches (ex. [this pull request](https://github.com/kubernetes/kubernetes/pull/64283)) against Kubernetes and other projects.

So why is that option commented out in our `Pod` above? The answer lies in current Kubernetes support, both for the `securityContext.procMount` option and the `allowedProcMountTypes` specification in our `PodSecurityPolicy`. While `procMount` has been [enabled by default since Kubernetes 1.12](https://github.com/genuinetools/img#running-with-kubernetes), `allowedProcMountTypes` is [still an Alpha feature](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-gates-for-alpha-or-beta-features).

This leaves us with a few options to handle the `procMount` specification, depending on the nature of our Kubernetes cluster:

1. If running a cluster that allows admin access to the Kubernetes master, we can selectively enable the Feature Gate for `allowedProcMountTypes` in our `PodSecurityPolicy`. Because GKE doesn't allow this access, this isn't an option in this example.
2. We can [create an Alpha GKE cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/alpha-clusters) which will enable the field in question, among many others. Because of the  functional limitations of Alpha clusters, this is only an option for sandbox workloads.
3. Because the required field is `PodSecurityPolicy`-specific, we could disable these policies across the entire cluster. However, this goes against security best practices.
4. Finally, we could simply omit the `securityContext.procMount: Unmasked` option in our `Pod` definition, which disables PID namespace isolation. We'll use this option for the remainder of the blog post.

## Using `img` in a Jenkins Pipeline
With our Kubernetes and Google Cloud resources now configured, we'll now look at running our `img` `Pod` from a Jenkins Pipeline. 

To execute this Jenkins job, we'll set up a [Multibranch Pipeline job](https://jenkins.io/doc/book/pipeline/multibranch/) that pulls from our [`img-pipeline` GitHub repository](https://github.com/mdelgin/img-pipeline). Here is the `Jenkinsfile` we'll use:
```groovy
pipeline {
  agent {
    kubernetes {
      label "img"
      yamlFile 'img-resources/imgPod.yaml'
    }
  }
  stages {
    stage('Build and Push') {
      steps {
        container('img') {
          sh """
            img build -t gcr.io/melgin/img-hello-world ./hello-world
            gcloud auth configure-docker --quiet
            img push gcr.io/melgin/img-hello-world
          """
        }
      }
    }
  }
}
```

In this Pipeline script, we load our agent `Pod` definition from `imgPod.yaml`, described above. Our actual steps are fairly straightforward and consist of three main actions within our `sh` step:

1. We use `img build` to build our `Dockerfile` located in the `hello-world` subdirectory. This is a simple example that copies a "Hello World" script into the [Docker scratch image](https://hub.docker.com/_/scratch).
2. We use `gcloud auth configure-docker` to [set up `gcloud` as a Docker credential helper](https://cloud.google.com/container-registry/docs/advanced-authentication#gcloud_as_a_docker_credential_helper), allowing us to authenticate to GCR through Workload Identity.
3. Finally, we use `img push` to push our newly built image to our GCR repository.

## Final Thoughts

We've now successfully used `img` as a non-privileged, non-root Kubernetes `Pod` agent to build and push a container image from a Jenkins Pipeline. Before wrapping up, let's summarize some of the security implications of running `img` in this manner.

First, `img` is running as a non-root user 1000 within a non-privileged container. This is an improvement from container building approaches that require root access (like `kaniko`) or using the `--privileged` flag (like Docker-in-Docker).

However, there are security settings at the `PodSecurityPolicy`-level that require consideration, like setting `AppArmor` & `seccomp` policies to `unconfined` and allowing privilege escalation. While the default behavior in Kubernetes is to [set `seccomp` to `unconfined`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#seccomp) and to [allow privilege escalation](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#privilege-escalation), more security-conscious organizations might disallow these settings by default in their `PodSecurityPolicies`.

Finally, without `procMount` set to `Unmasked`, the risk of child processes killing the parent `img` process remains. However, this risk is somewhat contained when `img` is run in an ephemeral container like it is as a Jenkins pod agent.
> *Note*: this concern will be effectively resolved if or when `allowedProcMountTypes` is promoted from its current Alpha status.

For organizations particularly sensitive to avoiding running root or privileged container workloads, `img` is a great candidate for building containers in Kubernetes.
