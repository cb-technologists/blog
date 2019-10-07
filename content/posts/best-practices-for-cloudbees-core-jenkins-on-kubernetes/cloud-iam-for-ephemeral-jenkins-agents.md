---
title: Securely Using Cloud IAM with Jenkins Kubernetes Agents
series: ["Best Practices for CloudBees Core (Jenkins) on Kubernetes"]
part: 1
authors:
  - "Kurt Madel"
date: 2019-10-07T05:05:15-04:00
showDate: true
tags: ["Kubernetes","CI","CD","Core v2","security","IAM","Jenkins","agents"]
photo: "/posts/best-practices-for-cloudbees-core-jenkins-on-kubernetes/bank.jpg"
photoCaption: "Wells Fargo Bank, Market Street, San Francisco, CA<br>Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 10.4mm ƒ/2.2 1/100"
draft: false
---
## Cloud IAM Permissions for Kubernetes Pods
As of a few months ago, there were a few ways to leverage cloud IAM (AWS IAM, GCP IAM, Azure ) from within a Kubernetes Pod. 

1. Applying a cloud IAM object (AWS IAM role, GCP IAM service account, Azure prinicipal) ot the underlying node where the Kubernetes `pod` would run. The biggest drawback of this approach is that all `pods` that run on that node would have access to those IAM permissions regardless of what Kubernetes `namespace` they were in and what Kubernetes RBAC configuration you are using.
2. Creating cloud credentials that can be mounted into Kubernetes `pods`. This may be an AWS credential or a GCP IAM servicer account key managed as a Kubernetes `secret` and mounted as a volume into a Kubernetes `pod`. One of the issues with this approach is that these type of credentials are much longer-lived than those in the first approach. In the case of a GCP IAM service account key it is valid for 10 years - so that would be very bad if an unwated user got access to that key file.

## There is a better way
In the past few months AWS and GCP have introduced a new more secure way of providing IAM permissions to a specific Kubernetes `namespace` and `serviceaccount` for EKS and GKE respectively. The underlying approach is very similar:

- with AWS EKS you bind an IAM role to a specifc Kubernetes `ServiceAccount` in a specific Kubernetes `Namespace`
- with GCP GKE you bind an IAM Service Account to a specifc Kubernetes `ServiceAccount` in a specific Kubernetes `Namespace`

Both of these new approaches for managing cloud IAM permissions in Kubernetes `pods` leverage some new features added to Kubernetes 1.12: [ServiceAccountTokenVolumeProjection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) and 

Projected service account tokens are valid OIDC JWTs (JSON Web Tokens) for `pods`.

In both cases the respective client library for interacting with AWs (AWS CLI)and GCP (gloud SDK) will automatically authorize against the short lived tokens that are dynamically mounted for all Kubernetes `pods` that are launched by the bound Kubernetes `ServiceAccount` and `namespace`.

## Limiting Your Jenkins Kuberentes Blast Radius with Master and/or Folder Specific Kubberentes Cloud Configurations
A common use case for a Jenkins Pipeline is to interact with cloud services. For example, 

One K8s cloud config per Jenkins masters.
One K8s cloud config per Jenkins folder that is protected with some form of RBAC.

Create a unique Kubernetes `namespaced` and `serviceaccount` for each Jenkins Kubernetes cloud config.
Configure each unique Jenkins Kubernetes cloud configuration with the `secret` token for the corresponsind `SecurityAccount`.

