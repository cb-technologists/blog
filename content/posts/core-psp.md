---
title: Using Kubernetes Pod Security Policies with CloudBees Core
series: ["Secure CloudBees Core on Kubernetes"]
part: 1
author:
  name: "Kurt Madel"
date: 2019-08-05T06:05:15-04:00
showDate: true
tags: ["Kubernetes","CI","CD","Core v2","security","Pod Security Policies"]
photo: "/img/building-container-images-with-kubernetes/secure-containers.jpg"
photoCaption: "Square Tower House, Ancient Pueblo Dwelling, Mesa Verde National Park, CO<br>Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 37.1mm ƒ/5.6 1/160"
draft: true
---
## What are Pod Security Policies?
Although [Kubernetes Pod Security Polices](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) are still a **beta** feature of Kubernetes they are an important security feature that should not be overlooked. Pod Security Policies (PSPs) are built-in Kubernetes resources that allow you to enforce security related properties of every container in your cluster. If a container in a pod does not meet the criteria for an applicable PSP then it will not be scheduled to run.

## OOTB and Documented Security Best Practices for CloudBees Core v2 on Kubernetes
There are [numerous articles](https://rancher.com/blog/2019/2019-01-17-101-more-kubernetes-security-best-practices/) [on security best practices](https://www.twistlock.com/2019/06/06/5-kubernetes-security-best-practices/) for Kubernetes (to include [this one published on the CNCF blog site](https://www.cncf.io/blog/2019/01/14/9-kubernetes-security-best-practices-everyone-must-follow/)). Many of these articles include similar best practices and most, if not all, apply to running Core v2 on Kubernetes. Some of these best practices are inherent in CloudBees' documented install of Core v2 on Kubernetes, while others are CloudBees documented best practices and are easy next steps after your initial installation. 

Before we take a look at the best practices that aren't necessarily covered by the CloudBees reference architectures and best practices I will provide a quick overview of what is already there OTTB and highlight some CloudBees documentation that speaks to other security best practices.

### Enable Role-Based Access Control (RBAC)
Although you can certainly install Core v2 on Kubernetes with RBAC enabled - the CloudBees install for Core v2 comes with RBAC configured.

### Use Namespaces to Establish Security Boundaries & Separate Sensitive Workloads
CloudBees recommends that you create a `namespace` specifically for Core v2 as part of the install. Additionally, CloudBees recommends establishing boundaries between your CloudBees Jenkins masters and agent workloads.

### Create and Define Cluster Network Policies
Although CloudBees doesn't go as far as to specify Kubernetes Network Policies, CloudBees does recommend using them.

### Run a Cluster-wide Pod Security Policy
This is one component that is not a currently documented as part of the installation guides or best practice documentation for Core v2 and will be the focus of the rest of this post.

## Why should you use Pod Security Policies?
From the Kubernetes documentation on Pod Security Policies, "Pod security policy control is implemented as an optional (**but recommended**) [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)." If you read any number of posts on security best practices for Kubernetes, pretty much all of them will mentions PSPs.

A CD platform, like CloudBees Core v2 on Kubernetes, is typically a  multi-tenant service where security is of the utmost importance. In addition to multi-tenancy, when running CD workloads on a platform like Kubernetes there are typically other workloads deployed and if any workload does not have proper security configured it can impact all of the workloads running on the cluster.

The combination of PSPs with Kubernetes RBAC, namespaces and workload specific node pools allows for the granular security you need to ensure there are adequate safeguards in place to greatly reduce the risk of unintentional (and intentional) stuff that breaks your cluster.
Work in tandem with targeted node pools, namespaces and service accounts.
You want to give as much flexibility to your CI/CD users as possible but provide some guard rails so they don't impact themselves or others by doing something stupid.

Block the Docker socket or disable DinD or don't allow running as `root` in a container.

"protect your cluster from accidental or malicious access"

## How do you use Pod Security Policies with CloudBees Core v2?
As mentioned above, Pod Security Polices are optional so they are not enabled by default on most Kubernetes distributions - to include GCP GKE, AWS EKS and Azure AKS. PSPs can be created without enabling the PodSecurityPolicy admission controller and applied to a `clusterrole` or a `role`. And this is very important, because **once you enable the admission controller any `pod` that does not have a PSP applied to it will not get scheduled**.

CloudBees defines two service accounts for the Core v2 install on Kubernetes, one for provisioning Managed/Team Masters and the other for scheduling dynamic ephemeral agent pods.

We will define two different PSPs - one for non-CloudBees components and the other for CloudBees components and the dynamic ephemeral agent pool.

Default Pod Security Policies created when enabling `pod-security-policy` on a GKE cluster:
```shell
AME                           PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
gce.event-exporter             false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            hostPath,secret
gce.fluentd-gcp                false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            configMap,hostPath,secret
gce.persistent-volume-binder   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            nfs,secret
gce.privileged                 true    *      RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
gce.unprivileged-addon         false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            emptyDir,configMap,secret
```

>NOTE: The default Pod Security Policies created automatically cannot be modified - Google will automatically change them back.