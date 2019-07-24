---
title: Securely Building Container Images on Kubernetes
author:
  name: "Kurt Madel"
date: 2019-07-24T06:05:15-04:00
showDate: true
tags: ["Kubernetes","CI","CD","DinD","security","kaniko","containers"]
photo: "/img/building-container-images-with-kubernetes/secure-containers.jpg"
photoCaption: "Square Tower House, Ancient Pueblo Dwelling, Mesa Verde National Park, CO<br>Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 37.1mm ƒ/5.6 1/160"
canonicalUrl: https://kurtmadel.com/posts/native-kubernetes-continuous-delivery/building-container-images-with-kubernetes/
draft: false
---
*Originally published on [kurtmadel.com](https://kurtmadel.com/posts/native-kubernetes-continuous-delivery/building-container-images-with-kubernetes/)*

Back in 2013, [before Kubernetes was a thing](https://kubernetes.io/blog/2015/04/borg-predecessor-to-kubernetes/), Docker was making Linux containers (LXC) much more accessible and use of Docker based containers took off (and [Docker quickly dropped LXC as the default execution engine for their own container runtime](https://blog.docker.com/2014/03/docker-0-9-introducing-execution-drivers-and-libcontainer/)). At the same time continuous integration (CI) was rapidly maturing as a best practice and a necessity for efficient software delivery. The use of Docker containers with CI was quickly adopted as the best way to manage CI tools - compilers, testing tools, security scans, etc. But it was new and there weren't a lot of best practices defined - it was more like 'go figure it out'. And early on one very important aspect of using containers for CI/CD was using containers to build container images and pushing those images to container registries - but again, this was all very new, and a lot of people didn't really know what they were doing and there wasn't a *Building Container Images for Dummies*.

Fast forward a couple of years to September of 2015 when Jérôme Petazzoni published an article entitled ["Using Docker-in-Docker for your CI or testing environment? Think twice."](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/) The article basically describes how using [Docker-in-Docker (DinD)](https://github.com/jpetazzo/dind) is bad choice for a CI/CD workload for a number of different reasons - it is definitely an article still worth a read. He promoted the concept of mounting the Docker socket of the host machine running the Docker daemon and using the host Docker daemon to execute Docker commands in your CI/CD jobs. Mounting the `/var/run/docker.sock` file as a volume in a Docker container allows you to accomplish similar functionality to DinD, but without some of the drawbacks of DinD -  to include layer caching and requiring running DinD containers with [`--privileged` mode](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/) enabled. In Part 5 of [this series on Native Kubernetes Continuous Delivery](/series/native-kubernetes-continuous-delivery/) we will explore why it is no longer a best practice to use either of these two approaches for building and pushing container images as part of your Native Kubernetes Continuous Delivery pipelines. We will look at this from two different perspectives: security and performance. Finally, we will take a look at an alternative approach with security and performance in mind.

## Container Images vs Docker Images
Before we dive into securely building and pushing container images on Kubernetes I wanted to share some thoughts on container terminology. I typically refer to an image that you run as a container in a Kubernetes Pod as a **Container Image** instead of a **Docker Image**. Back in 2015, Docker was kind enough to [donate the Docker image format](https://blog.docker.com/2017/07/oci-release-of-v1-0-runtime-and-image-format-specifications/) to the then newly established [Open Container Initiative (OCI)](https://www.opencontainers.org/) - in addition to the container [Image Specification](https://github.com/opencontainers/image-spec/blob/master/spec.md), the OCI also maintains an open [Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/spec.md) for container execution. That makes Docker *no-longer-required* for running containers and pulling container images - and, as you will see later in this post, even building and pushing container images.

## What's Wrong with Docker-in-Docker (DinD)
Again, to understand the drawbacks of using DinD to build and push container images I recommend that you read [Jérôme Petazzoni's article](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/). But here is a quick summary:

* Security - The DinD container must run with `--privileged` flag enabled, resulting in undesirable attack vectors against the underlying Docker host.
* Performance - Layer caching is not shared across builds when using ephemeral DinD containers. All layers of all images must be pulled every time a new DinD container is used, resulting in much slower container image builds.

## What's Wrong with Mounting the Docker Socket

So we have established that using DinD for CI/CD, especially for building and pushing container images, is a bad idea. But for Kubernetes CD you should also think twice if you are mounting the Docker socket for CI/CD - to include building and pushing container images. 

### Performance

Let’s put security aside for a moment - it turns out that mounting the Docker socket has significant issues in a Kubernetes based CD environment. A Kubernetes cluster is made up of one or more worker nodes, and it is on these worker nodes where Kubernetes schedules and runs [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/). When you mount the Docker socket to a Pod you are mounting the `/var/run/docker.sock` file into every container that makes up your Pod. When those containers run Docker commands against that socket they are actually being executed directly by the Docker daemon running as **root** on the worker node where the Pod was scheduled. The Kubernetes scheduler has no way to track that these other containers are running - they aren’t managed by Kubernetes, rather they are managed by the Docker daemon running on the node where the Pod gets scheduled. This may result in serious Kubernetes scheduling issues, especially on busy CD clusters. And one of the main reasons to use Kubernetes in the first place is because of its robust orchestration and scheduling capabilities of containers, so why would you want to circumvent that for your CI/CD? 

Another performance issue is container image layer caching. If you depend on the built-in caching provided by the Docker daemon then you may end up on different K8s nodes for different builds of the same container image or other container images that share layers - thus negating the caching provided by the Docker daemon on a specific K8s worker node.

Additionally, any layers that are stored on the host and any logs generated by the containers running via the Docker socket will not be automatically cleaned up for you.

### Security
In simplistic terms, increased security goes hand in hand with reducing the attack surface or attack vectors. It is no different with CD on Kubernetes. If the Docker socket is exposed to a CD job then a curious/malicious developer can modify the job to run Docker commands as build steps, potentially becoming `root` on the node where that job lands. If they gain access to the underlying host as `root`, there are reasonably straightforward methods to escalate privileges and gain access to the entire Kubernetes cluster. Many cluster operators don't have the proper monitoring in place to properly detect this kind of activity and separate it from legitimate CD job runs, so the exploitation is likely to go unnoticed for a while.

**DinD** has always [required that the `--privileged` flag be enabled for the container running DinD](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/) - and this has always been considered insecure. But mounting the Docker socket has never been any more secure, and has relied on the use of dedicated Docker daemon instances to isolate CI/CD workloads from other container workloads - like production applications for example. 

While this is high on the **bad** scale for a single-purpose cluster just doing CI/CD, it is extremely high on the **bad** scale for a K8s cluster with multiple workloads that should have isolation - the isolation you should expect from containers running on K8s. For example, your **Production** containers may be just a namespace away from the namespace where your CD job is running.

If you go down this path, the net result is *anyone who can modify a CD job has a way to become root for the entire cluster*.

## Block the Use of DinD and the Docker Socket for K8s CD Pods
For some, the drawbacks of mounting the Docker socket for CD on Kubernetes are significant enough that they [recommended going back to DinD](https://applatix.com/case-docker-docker-kubernetes-part-2/). But we have already dismissed both **DinD** and mounting the Docker socket as acceptable approaches for CD on Kubernetes. Before we look at an alternative we will explore K8s features that allow you to block the use of DinD and the mounting of the Docker socket for all containers running on your K8s cluster (or part of your cluster).

By using Kubernetes for dynamic ephemeral CD executors you can run each of your CD steps directly in containers built specifically and singularly for the purpose of executing that step or step(s), all within a [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) managed by Kubernetes. In addition to orchestrating the scheduling and running of these CD Pods - K8s also allows managing other aspects of these Pods' lifecycles to include security sensitive aspects of the pod specification that enable fine-grained authorization of pod creation and updates. A native K8s solution for managing pod security is the [Pod Security Policy Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy). A Pod Security Policy (PSP) may be configured in such a way that a container running in a Pod using a correctly configured policy won't be scheduled if it is configured to mount the Docker socket and won't be allowed to run as a `--privileged` container - disabling DinD.

### Use Pod Security Policies to Block DinD and Mounting the Docker Socket
The Pod Security Policy Admission Controller is a [critical feature for enhancing the security of your K8s cluster](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/#6-use-linux-security-features-and-podsecuritypolicies). The Pod Security Policy Admission Controller allows you to specify Pod Security Policies that limit what containers are allowed to do - if a container in a Pod is configured to do something that is not allowed by the Pod Security Policy then K8s will not schedule the Pod.

>[From the Kubernetes official docs](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#what-is-a-pod-security-policy): *A Pod Security Policy is a cluster-level resource that controls security sensitive aspects of the pod specification. The PodSecurityPolicy objects define a set of conditions that a pod must run with in order to be accepted into the system, as well as defaults for the related fields*

Let's look at some specific settings that will greatly reduce the *attack* surface thus mitigating risk: 

- `privileged`: set to `false` will disallow the use of DinD.
- `runAsUser`: set this to `MustRunAsNonRoot` so containers can't run as the `ROOT` user.

>NOTE: You will need to allow `USER root` to actually do anything meaningful with Kaniko to build and push container image, so you will most likely need to set `runAsUser` to `RunAsAny`. The goal with a Kaniko PSP is to reduce other available attack vectors.

- `allowPrivilegeEscalation`: disable privilege escalation so that no child process of a container can gain more privileges than its parent.
- `volumes`: Don't allow mounting host directories/files as volumes by specifying [specific volume types](https://kubernetes.io/docs/concepts/storage/volumes/) and not allowing the `hostPath` volume for any CD containers. This will disable the ability to mount the Docker socket.
- PSP `annotations`: confine all Pod containers to the `runtime/default` **seccomp** profile via the `seccomp.security.alpha.kubernetes.io/defaultProfileName` annotation and don't set the `seccomp.security.alpha.kubernetes.io/allowedProfileNames` so the default cannot be changed.

Here is an example of a restrictive **PSP** that won't allow **mounting the Docker socket** and won't allow **DinD**:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: cd-restricted
  annotations:
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # Required to prevent escalations to root.
  allowPrivilegeEscalation: false
  # This is redundant with non-root + disallow privilege escalation,
  # but we can provide it for defense in depth.
  requiredDropCapabilities:
    - ALL
  # Allow core volume types. But more specifically, don't allow mounting host volumes to include the Docker socket - '/var/run/docker.sock'
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    # Assume that persistentVolumes set up by the cluster admin are safe to use.
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    # Don't allow containers to run as ROOT
    rule: 'MustRunAsNonRoot'
  seLinux:
    # This policy assumes the nodes are using AppArmor rather than SELinux.
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

>**NOTE**: At the time this post was published [GKE has beta support for Pod Security Polices](https://cloud.google.com/kubernetes-engine/docs/how-to/pod-security-policies), [AKS introduced preview support for Pod Security Policies in April](https://docs.microsoft.com/en-us/azure/aks/use-pod-security-policies) and [EKS added Pod Security Polices as a default feature with their support for K8s 1.13 (also new 1.12 EKS clusters will include support for Pod Security Policies)](https://docs.aws.amazon.com/eks/latest/userguide/pod-security-policy.html).

### Don't Run Docker
Another compelling way to not allow DinD or mounting the Docker Socket is to not use Docker as the container runtime for your K8s cluster. 
[**containerd**](https://containerd.io/) is an implementation of the OCI image runtime mentioned above  ([also donated by Docker to the CNCF](https://blog.docker.com/2017/03/docker-donates-containerd-to-cncf/)) and is (at the time of this post) [supported by the Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs/concepts/using-containerd) - *but no other major cloud providers*. By using the OCI run spec provided by **containerd**, you don't actually need Docker - and you will actually see better performance for your K8s containers with **containerd** because Docker actually uses **containerd** under the covers - so to speak - resulting in an extra daemon and unnecessary communication overhead. One interesting aspect of **containerd** is that it is only a runtime for containers - it does not support building container images. But a big plus of **containerd**, besides better performance, is that it makes it impossible to use **DinD** or mount the Docker socket - thus providing a more secure container runtime. But how do you build container images without Docker?

## Building Container Images without Docker
Or more specifically, building container images without the Docker **daemon** - no **DinD**, no Docker socket. But we still want to leverage K8s managed containers for CD, to include building and pushing container images. How do we do that?

### Enter Kaniko
Kaniko is a tool that is capable of building and pushing container images without the Docker daemon, but it does have one major drawback from a security perspective: to build anything useful/easily you must run as the `root` `USER` in the Kaniko container. 

>[**From Kaniko's docs**](https://github.com/GoogleContainerTools/kaniko#security): If you have a minimal base image (SCRATCH or similar) that doesn't require permissions to unpack, and your Dockerfile doesn't execute any commands as the root user, you can run Kaniko without root permissions. It should be noted that Docker runs as root by default, so you still require (in a sense) privileges to use Kaniko.

>You may be able to achieve the same default seccomp profile that Docker uses in your Pod by setting seccomp profiles with annotations on a PodSecurityPolicy to create or update security policies on your cluster. 

As we already mentioned above, running as `root` is an attack vector that many consider to be an unacceptable security hole - but the use of Pod Security Policies will reduce the attack surface of the Kaniko container running as part of a K8s Pod and provides greater security than the Docker based approaches we have already dismissed.

#### Basic Configuration for Kaniko with Pod Security Policies
The following configuration assumes that you have a K8 cluster up and running, and that you have access to `kubectl` to add and modify K8s resources.

1. **Container registry**: I recommend having a staging or sandbox container registry that Kaniko pushes images to, and a production registry that Kaniko does not have access to push to. A separately managed and secured CD job should be used to promote container images from staging to production once required tests/scans/policies have been successfully run against that container image. But of course the Kaniko container image itself (and any other container images used by CD jobs) should always be pulled from the production container registry.
2. `PodSecurityPolicy`: Once the PodSecurityPolicy admission controller is enabled you will need at least 2 PodSecurityPolicies (acutally you will want to defined and apply all of your Pod Security Policies before enabling the admission controller, not doing so will prevent any pods from being created in the cluster):
   1. The most restrictive policy possible for executing a build and push with Kaniko - the only change to the PSP above is to change `runAsUser` to `RunAsAny` to allow using the ROOT user in Dockerfiles to be built by Kaniko. 
   2. A [privileged policy](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/privileged-psp.yaml) that is equivalent to not using the Pod Security admission controller for Pods that use it.
3. `Namespace`, `Service Account`, `Role`, `RoleBinding`: a PodSecurityPolicy is applied to a K8s `Role` that is then bound to a `ServiceAccount` via a `RoleBinding`. I recommend creating a `ServiceAccount` bound to a `Role` with a restrictive PodSecurityPolicy specifically for Kaniko/CD jobs.


#### Kaniko with GCP, GKE and GCR
Kaniko is a Google sponsored project, so naturally there is good support for using Kaniko with a GKE cluster. I [defer to the Kaniko instructions for GKE + GCR](https://github.com/GoogleContainerTools/kaniko#running-kaniko-in-a-kubernetes-cluster).

[Using Pod Security Policies with GKE.](https://cloud.google.com/kubernetes-engine/docs/how-to/pod-security-policies)

#### Kaniko with AWS, EKS and ECR
The Kaniko instructions tell you to create a Kubernetes secret with your `~/.aws/credentials` to push container images to the ECR, but most organizations don't allow you to use AWS credentials this way. Here are [Kaniko's instructions for pushing to AWS ECR](https://github.com/GoogleContainerTools/kaniko#pushing-to-amazon-ecr).

Another approach for securely using Kaniko with EKS and ECR is to use worker node IAM roles with an instance group dedicated to Kaniko/CD.

[Using Pod Security Policies with EKS.](https://aws.amazon.com/blogs/opensource/using-pod-security-policies-amazon-eks-clusters/)

#### Kaniko with Azure, AKS and ACR
At the time of this post, Kaniko does not have official support for the Azure Container Registry (ACR) - but that doesn't mean it isn't possible. There is an [open issue](https://github.com/GoogleContainerTools/kaniko/issues/425) in the Kaniko GitHub repository that includes some tips on pushing container images to the ACR from Kaniko.

[Using Pod Security Policies with AKS.](https://docs.microsoft.com/en-us/azure/aks/use-pod-security-policies)

#### Kaniko the Easy Way
Jenkins X allows you to [enable Kaniko as the default way to build and push container images](https://jenkins-x.io/getting-started/create-cluster/#the-jx-create-cluster-gke-process) for all of your Jenkins X CD jobs and will be automatically configured to push to the default container registry of the cloud where you install Jenkins X and Kaniko caching is automatically set up for you - resulting in fast, secure container image builds that are pushed to your default Jenkins X container registry.

**Important:** Jenkins X does not have OOTB support for Pod Security Policies as tracked by [this GitHub issue](https://github.com/jenkins-x/jx/issues/1074). In my next post we will take a look at using Pod Security Policies with Jenkins X - but not just for Kaniko, because once you enable Pod Security Policy every K8s `Role`/`ClusterRole` has to have a Pod Security Policy associated to it.

#### Drawbacks for Kaniko

- Requires running the Kaniko container as `ROOT` to execute most container builds
- Doesn't work with all `Dockerfiles` but keeps improving
- Is slightly more complicated to setup than the good old `docker build`

## Other Security Vectors You Should Cover
Enforce use of specific container registries - for example don’t allow pulling from or pushing to a public container registry like DockerHub. You should maintain your own container images and container registries - to include having at least two different container registries - one for CD container images (along with intermediate application container images) and another container registry for production approved application container images. I would actually take this a step further and have a third container registry that is a sandbox of sorts that allows CD specific container images and intermediate application container images that haven't been approved for more secure environments - like production - but are allowed to run in less secure test environments.

Scan your container images before you make them available for use. Here is [a blog post of using Anchore with a Jenkins Pipeline](https://cb-technologists.github.io/posts/cloudbees-cross-team-and-dev-sec-ops/) - super simple to set-up so there really is no reason not to scan your container images and use the scan as a gate for promotion to more secure container registries.

## Other Solutions
Other solutions that allow you to build and push container images using a Kubernetes Pod based container and that don't rely on the Docker daemon:

- [img](https://github.com/genuinetools/img): **img** is only at version 0.5.7 (May 3, 2019 release), but this project is promising. **img** required [upstream patches](https://github.com/genuinetools/img#upstream-patches) to different projects that should eventually make it the most secure (and fast) way to build and push container images from a K8s Pod container.
- [jib](https://github.com/GoogleContainerTools/jib) - Another Google project, **jib** supports building and pushing container images without the Docker daemon.
  - **jib** only supports building container images for Java applications with Maven and Gradle support.

If you aren't already, start building your container images with Kaniko on Kubernetes - with Pod Security Policies. And again, if you want to get Kaniko up and running quickly and easily then you should checkout Jenkins X. Jenkins X will automatically set up Kaniko, along with the necessary configuration to push container images to the Docker registry of your choice (GCR, ECR, Docker Hub, etc). You can go from a Dockerfile in GitHub to container image in your registry in minutes with Jenkins X.

{{< load-photoswipe >}}