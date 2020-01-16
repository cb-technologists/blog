---
title: Amazon ECS vs EKS for running CloudBees Core/Jenkins
authors:
  - "Logan Donley"
date: 2020-01-13T05:05:15-04:00
showDate: true
tags: ["Kubernetes","aws","EKS","ECS"]
draft: true
---

A question my colleagues and I are often asked is "if I'm already using ECS is there any point in moving to EKS (or Kubernetes in general) for running CloudBees Core?" Both Amazon ECS and EKS are great products, but we have a strong preference towards Kubernetes. We'll take a look at why in this post.

## What are ECS and EKS?

These are both services from AWS which deal with container orchestration. Containers have become a crucial component in the modern tech stack, but as your applications scale, having an orchestrator is a must. 

### ECS

Amazon Elastic Container Service is a platform which will orchestrate your containers in a straightforward manner. You create an application definition and it will ensure that the container is running. 

With ECS there are a couple of ways you can run your containers: on EC2 instances or with [Fargate](https://aws.amazon.com/fargate/), AWS's serverless compute-engine.

With EC2 you operate like you normally would and configure auto-scaling groups, networking, etc. Fargate on the other hand abstracts away the individual server and let you instead request specific cpu and memory for your workloads. The cost when using Fargate will be higher per compute/memory resource but it handles a lot for you.


### EKS

Amazon Elastic Kubernetes Service is AWS's managed Kubernetes platform. Kubernetes has become the community-favored container orchestrator and for good reason. It's incredibly powerful, resilient, and now has a great ecosystem built around it.

With all of the power of Kubernetes comes some complexity on the infrastructure side. Setting up and maintaining your own Kubernetes cluster can be challenging, even for those with lots of experience. That's why whenever possible, we would recommend you use a managed Kubernetes service like EKS, GKE, or AKS since they solve those challenges for you.

In December 2019, AWS started to [support using EKS with Fargate](https://aws.amazon.com/about-aws/whats-new/2019/12/run-serverless-kubernetes-pods-using-amazon-eks-and-aws-fargate/), so that specific differentiator between using ECS and EKS has vanished. Now you can deploy services in Kubernetes without worrying about the underlying infrastructure at all by using the Fargate serverless compute engine with EKS.

Using EKS is in many ways similar to using ECS since whether you use Fargate or EC2, AWS will take care of provisioning the resources for you, letting you instead focus on the applications you will be running. The real differences come down to which container orchestrator you are using. ECS's solution or Kubernetes'.

## Battle of the orchestrators

If at the end of the day an application is going to be running in a container does it really matter which orchestrator you use as long as it works? Especially if one is simpler than the other? I would argue that it does matter. 

Container orchestrators have become the new platform on which you build and deploy applications. While we often talk about avoiding vendor lock-in as a best practice, it happens just as frequently with technologies. For instance, if you are developing for Linux machines your application isn't necessarily portable to a Windows machine. At least not without extra work. This is a form of lock-in.

Investing time and effort into a container platform is locking you in to that platform to some degree. Sure your applications will be portable as they are containers, but the process and lifecycle will be different depending on which platform you use. This isn't to say that the migration between the two is complicated, just that it does require some time and effort.

The focus of this post is about how this relates to CloudBees Core & Jenkins, but first let's do a quick pros/cons of both ECS and EKS. 

### ECS Pros & Cons

| Pros |
| --- |
| + Easy to use |
| + Longstanding integration with Fargate to abstract away the infrastructure |
| + Tight integration with the AWS ecosystem |

| Cons |
| --- |
| - Proprietary tooling |
| - Limited third-party ecosystem |
| - Not portable outside of AWS |


### EKS Pros & Cons

| Pros |
| --- |
| + All the power of Kubernetes |
| + Any Kubernetes work done is portable to any other Kubernetes cluster |
| + Easy to build tooling around |
| + New integration with Fargate to add serverless functionality* |


| Cons |
| --- |
| - Kubernetes can be challenging to learn |
| - Without automation or tooling, the process of deploying a single application is more involved |
| - Can be overkill for small projects |


* It's important to note that with the Fargate integration, there are some limitations. For the time being, Fargate doesn't support stateful applications, so you wouldn't run Jenkins/Core directly on Fargate, but rather use it for agents. There is also a limited number of regions which support Fargate on EKS. More details here: [https://docs.aws.amazon.com/eks/latest/userguide/fargate.html](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html).


## Running CloudBees Core/Jenkins

Up this point, we've taken a look at ECS vs EKS and how Fargate fits into the mix. Now let's look at why we prefer running CloudBees Core and Jenkins on Kubernetes.

### Portability

Kubernetes is an open-standard. When you build your CI/CD system on Kubernetes, it can run on EKS, GKE, AKS, OpenShift, PKS, home-built, etc. without issue (well some vendor flavors like OpenShift have additional resources which aren't portable to standard k8s clusters).

Some organizations are fine with going all-in on a single cloud vendor, but many others try to avoid vendor lock-in as much as possible. Leveraging as many open and portable platforms as possible ensures that the effort you put in while building your system won't be wasted if you switch from AWS to GCP or Azure.

The ECS platform exists only in AWS and is proprietary, so you don't have portability to other vendors. 

This is a big plus for using EKS (or any Kubernetes).


### Community

As of writing this post, Kubernetes is the open standard for container orchestration, in major part because of it's huge and rapid community adoption. Since it is the standard, a large amount of DevOps tooling has been built on top of it. Most CI/CD processes will run through many different tools, so opting for the platform with the greatest marketshare is a safe choice.

Because of this reality, we at CloudBees have built lots of functionality into CloudBees Core specific to Kubernetes which adds a lot of quality of life improvements. The ability to dynamically provision masters and agents is in my opinion a game changer. There are features like hibernating masters which we have recently released as a technical preview which allows for potentially huge cost savings by spinning down masters that haven't been used in a while. Functionality like this is only possible on a consistent platform, and Kubernetes gives us this.


## Wrapping it up

When it comes to running CloudBees Core on a modern platform, Kubernetes is our recommendation. With recent changes to Amazon's EKS to support Fargate, it is now possible to run your agents as serverless workloads, giving you one less thing to manage.

ECS is a nice tool, especially for smaller projects, but it pales in comparison to Kubernetes when it comes to the quality features required by the DevOps community.