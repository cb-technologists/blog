---
title: Using Kubernetes Pod Security Policies with CloudBees Core
series: ["CloudBees Core on Kubernetes Best Practices"]
part: 1
author:
  name: "Kurt Madel"
date: 2019-09-04T05:05:15-04:00
showDate: true
tags: ["Kubernetes","CI","CD","Core v2","security","Pod Security Policies"]
photo: "/posts/cloudbees-core-on-kubernetes-best-practices/bank.jpg"
photoCaption: "Wells Fargo Bank, Market Street, San Francisco, CA<br>Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 10.4mm ƒ/2.2 1/100"
draft: false
---
## What are Pod Security Policies?
Although [Kubernetes Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) are still a **beta** feature of Kubernetes they are an important security feature that should not be overlooked. Pod Security Policies (PSPs) are built-in Kubernetes resources that allow you to enforce security related properties of every container in your cluster. If a container in a pod does not meet the criteria for an applicable PSP then it will not be scheduled to run.

## Best Practices for CloudBees Core v2 on Kubernetes
There are [numerous articles](https://rancher.com/blog/2019/2019-01-17-101-more-kubernetes-security-best-practices/) [on security best practices](https://www.twistlock.com/2019/06/06/5-kubernetes-security-best-practices/) for Kubernetes (to include [this one published on the CNCF blog site](https://www.cncf.io/blog/2019/01/14/9-kubernetes-security-best-practices-everyone-must-follow/)). Many of these articles include similar best practices and most, if not all, apply to running Core v2 on Kubernetes. Some of these best practices are inherent in CloudBees' documented install of Core v2 on Kubernetes, while others are documented best practices and are recommended next steps after your initial Core v2 installation. 

Before we take a look at the best practices that aren't necessarily covered by the CloudBees reference architectures and best practice documentation, I will provide a quick overview of what is already available with an OOTB Core v2 install and highlight some CloudBees documentation that speaks to other best practices for running Core v2 on Kubernetes more securely.

### Enable Role-Based Access Control (RBAC)
Although you can certainly install Core v2 on Kubernetes without RBAC enabled - the CloudBees install for Core v2 comes with RBAC pre-configured. Running Kubernetes with RBAC enabled is typically the default (it is for all the major cloud providers) and is always a recommended security setting.

### Use Namespaces to Establish Security Boundaries & Separate Sensitive Workloads
CloudBees recommends that you create a `namespace` specifically for Core v2 as part of the install. CloudBees also recommends establishing boundaries between your CloudBees Jenkins masters and agent workloads by [setting up distinct node pools](https://go.cloudbees.com/docs/cloudbees-core/cloud-install-guide/gke-install/#_distinct_node_pools) using [taints and tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) and [assigning pods to specific node pools with node selectors](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/).

### Create and Define Cluster Network Policies
Although CloudBees doesn't provide specific Kubernetes Network Policies, CloudBees does recommend using them and [provides documentation for setting up a private and encrypted network for AWS EKS](https://go.cloudbees.com/docs/cloudbees-core/cloud-reference-architecture/ra-for-eks/#_setting_up_a_private_and_encrypted_network).

### Run a Cluster-wide Pod Security Policy
At the time of this post, this is one component that is not documented as part of the CloudBees installation guides for Core v2 on Kubernetes and will be the focus of the rest of this post.

## [Why should you use Pod Security Policies?](/posts/build-continaer-images/)
From the Kubernetes documentation on Pod Security Policies (PSPs): "Pod security policy control is implemented as an optional (**but recommended**) [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)." If you read any number of posts on security best practices for Kubernetes, pretty much all of them will mentions PSPs.

A CD platform, like CloudBees Core v2 on Kubernetes, is typically a multi-tenant service where security is of the utmost importance. In addition to multi-tenancy, when running CD workloads on a platform like Kubernetes there are typically other workloads deployed and if any workload does not have proper security configured it can impact all of the workloads running on the cluster.

The combination of PSPs with Kubernetes RBAC, namespaces and workload specific node pools allows for the granular security you need to ensure there are adequate safeguards in place to greatly reduce the risk of unintentional (and intentional) actions that breaks your cluster. PSPs provide additional safeguards along with targeted node pools, namespaces and service accounts. This allows for the flexibility needed by CI/CD users while providing adequate guard rails so they don't negatively impact CD workloads or other important Kubernetes workloads by doing something stupid - accidental or otherwise.

## Using Pod Security Policies with CloudBees Core v2
As mentioned above, Pod Security Polices are an optional Kubernetes feature (and still beta) so they are not enabled by default on most Kubernetes distributions - to include GCP GKE, and Azure AKS. PSPs can be created and applied to a `ClusterRole` or a `Role` resource definition without enabling the PodSecurityPolicy admission controller. This is very important, because **once you enable the PodSecurityPolicy admission controller any `pod` that does not have a PSP applied to it will not get scheduled**.

>NOTE: PSPs are enabled by default on AWS EKS 1.13 and above, but with a very permissive PSP that is the same as running EKS without PSPs.

We will define two PSPs for our Core v2 cluster:

- A very restrictive PSP used for all CloudBees components, additional Kubernetes services being leveraged with Core v2 and the *majority* of dynamic ephemeral Kubernetes based agents used by our Core v2 cluster:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: cb-restricted
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  # prevents container from manipulating the network stack, accessing devices on the host and prevents ability to run DinD
  privileged: false
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  runAsUser:
    rule: 'MustRunAs'
    ranges:
      # Don't allow containers to run as ROOT
      - min: 1
        max: 65535
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  # Allow core volume types. But more specifically, don't allow mounting host volumes to include the Docker socket - '/var/run/docker.sock'
  volumes:
  - 'emptyDir'
  - 'secret'
  - 'downwardAPI'
  - 'configMap'
  # persistentVolumes are required for CJOC and Managed Master StatefulSets
  - 'persistentVolumeClaim'
  - 'projected'
  hostPID: false
  hostIPC: false
  hostNetwork: false
  # Ensures that no child process of a container can gain more privileges than its parent
  allowPrivilegeEscalation: false
```

Once the primary Core v2 PSP (`cb-restricted` in this case) has been created you must update the `Roles` to use it. CloudBees defines two Kubernetes `Roles` for the Core v2 install on Kubernetes, `cjoc-master-management` bound to the `cjoc` `ServiceAccount` for [provisioning Managed/Team Masters `StatefulSets` from CJOC](https://go.cloudbees.com/docs/cloudbees-core/cloud-reference-architecture/ra-for-gke/#_master_provisioning), and `cjoc-agents` bound to the `jenkins` `ServiceAccount` for scheduling dynamic ephemeral agent pods from Managed/Team Masters. The following Kubernetes configuration snippets show how this is configured:

```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cjoc-master-management
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - cb-restricted
...
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cjoc-agents
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - cb-restricted
...
```

- The second PSP will be almost identical except for `RunAsUser` will be set to `RunAsAny` to allow running as `root` - this is specifically to run Kaniko containers ([read more about building containers as securely as possible with Kaniko](https://kurtmadel.com/posts/native-kubernetes-continuous-delivery/building-container-images-with-kubernetes/)), but there may be some other uses cases that require containers to run as `root`:

```
  runAsUser:
    rule: 'RunAsAny'
```

The cluster used as an example for this post relies on two Kubernetes services for running Core v2: **cert-manager** for TLS and **ingress-nginx** for, well, ingress. If these are installed before you enable PSPs on your cluster then the `pods` associated with them will be stopped if the associated `Roles`/`ClusterRoles` don't have PSPs applied to them. Both services are deployed to their own namespaces so an easy way to ensure that all `ServiceAccounts` associated with those services have a PSP applied is to create a `ClusterRole` with the PSP  and then bind that `ClusterRole` to all `ServiceAccounts` in the applicable `namespace`:

*`ClusterRole` with the cb-restricted PSP applied*
```yaml
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-restricted-clusterrole
rules:
- apiGroups:
  - extensions
  resources:
  - podsecuritypolicies
  resourceNames:
  - cb-restricted
  verbs:
  - use
```

*`RoleBindings` for cert-manager and ingress-nginx `ServiceAccounts`*
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cert-manager-psp-restricted
  namespace: cert-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp-restricted-clusterrole
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingress-nginx-psp-restricted
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp-restricted-clusterrole
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts
```

>NOTE: You can use this command `kubectl get role,clusterrole --all-namespaces` to check your cluster for any other `Roles` or `ClusterRoles` that need to have a PSP applied to them. Remember, any `pod` that is running under a `ServiceAccount` that doesn't have a PSP will be shut down as soon as you enable the Pod Security Policy Admission Controller. For GKE you don't need to apply PSPs to any `Roles` in the `kube-system` `namespace` or any **gce** or **system** `ClusterRoles` as GKE will automatically apply the necessary PSPs.

Now that PSPs are applied to all the necessary `Roles` and `ClusterRoles` you can enable the Pod Security Policy Admission Controller for your GKE cluster:
```shell
gcloud beta container clusters update [CLUSTER_NAME] --zone [CLUSTER_ZONE] --enable-pod-security-policy
```

Next, you should ensure that all `pods` are still running:
```shell
kubectl get pods --all-namespaces
```
If a `pod` that you expect to be running is not, you need to find the `Role`/`ClusterRole` that is used for the `pod`/`deployment`/`service` and apply a PSP to it.


Default Pod Security Policies created when enabling the `pod-security-policy` feature on a GKE cluster:
```shell
NAME                           PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
gce.event-exporter             false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            hostPath,secret
gce.fluentd-gcp                false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            configMap,hostPath,secret
gce.persistent-volume-binder   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            nfs,secret
gce.privileged                 true    *      RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
gce.unprivileged-addon         false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            emptyDir,configMap,secret
```

>NOTE: The default Pod Security Policies created automatically cannot be modified - Google will automatically change them back to those above.

[AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/pod-security-policy.html) and [Azure AKS - Preview](https://docs.microsoft.com/en-us/azure/aks/use-pod-security-policies) also support Pod Security Policies.

## Oh no, My Jenkins Agents Won't Start!
The [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin) (for ephemeral K8s agents) defaults to using a K8s [`emptyDir` volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) type for the Jenkins agent workspace. This causes issues when using a restrictive PSP such at the **cb-restricted** PSP above. Kubernetes defaults to mounting `emptyDir` volumes as `root:root` with permissions set to `750` - as [detailed by this GitHub issue](https://github.com/kubernetes/kubernetes/issues/2630) opened way back in 2014. When using a PSP, with Jenkins K8s agent pods, that doesn't allow containers to run as `root` the containers will not be able to access the default K8s plugin workspace directory. One approach for dealing with this is to set the K8s `securityContext` for `containers` in the `pod` spec. You can do this in the K8s plugin UI via the **Raw yaml for the Pod** field:

![Raw yaml for the Pod](/posts/cloudbees-core-on-kubernetes-best-practices/raw-yaml-for-the-pod.png)

This can also be set in the raw yaml of a `pod` spec that you [load into your Jenkins job from a file](https://github.com/cloudbees-days/pipeline-template-catalog/blob/master/templates/nodejs-app/Jenkinsfile#L2):

*`pod` spec with the `securityContext`*
```yaml
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.10.0-alpine
    command:
    - cat
    tty: true
  - name: testcafe
    image: gcr.io/technologists/testcafe:0.0.2
    command:
    - cat
    tty: true
  securityContext:
    runAsUser: 1000
```