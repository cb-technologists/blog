---
authors:
  - "David Cañadillas"
title: "CI/CD High Availability: Misunderstanding the concept"
date: 2019-10-28T01:51:10+01:00
showDate: true
draft: false
tags: ["jenkins","jenkins x", "CI/CD pipelines","High Availability"]
---

High availability scenarios are key configurations for all mission critical software platforms and solutions at every company.

But in my experience from every technology conversation I've found out that High Availability (HA from now on) is one of those terms that is usually misunderstood when talking about CI/CD. The reason? Because HA can be a subjective topic depending on who you are talking to (IT, Development, Operations, end users, etc.)

I am writing this post to clear things out about the "myth" of HA in CI/CD and what it means from a solution architecture point of view. So, don't expect a post with a lot code or configuration examples. It is about architecture concepts, business needs and its technical alignment to help finding out best practices for *"highly available"* CI/CD pipelines. 


## What is HA and why is relative?

Even though [the definition at Wikipedia](https://en.wikipedia.org/wiki/High_availability) is not a true "official" source, I like how it defines HA:

> *High availability (HA) is a characteristic of a system, which aims to ensure an agreed level of operational performance, usually uptime, for a higher than normal period.*

The words *agreed level of operational performance* show some very important topics to clarify:

* *Operational performance* is defined by the use case and the business. Not from the technical components or the software being used.
* The software architecture and its technical definition depends on the *level agreement* of that operational performance.

Let's use an example. If we think about the service that [Amazon retail](https://amazon.com) users are expecting, any purchase transaction using the web portal cannot be interrupted in a matter of seconds. So, web services, front end applications, databases, middleware, payment channels  and other services or microservices, etc. cannot interrupt or block that user transaction. That means that every server needs to be up and running all the time, or at least to recover at the same point in a sub-second basis. 

This is just because the business requires that any user transaction and interaction with the application cannot be interrupted at all, considering that an interruption is something blocked during 3 seconds or more (users don't like to wait more than 3 seconds to refresh a web page to the next step). And if that doesn't happen, I - as a user - will be doing the transaction from a different web retailer or competitor. Business then requires a very restrictive HA scenario.

I am not going to enter in the differences of *"Active-Active"* or *"Active-Passive"* HA scenarios. But in the previous example a software architecture candidate usually needs to be based on an "Active-Active" replication method, where every software transaction is supported or backed-up automatically from an already running service that can re-take immediately the current step in the transaction, with no data-loss, or acting as a performant load balancer solution.

But not every use case or business requires the same level of HA. In other terms, the word *High* in the *HA* term is relative. A critical service interruption can be seconds, minutes or even hours depending on the context, use case or business.

## What about HA and CI/CD?

Continuous Integration and Continuous Delivery (CI/CD) is based on the concept of continuously develop, build, integrate, test, deliver and release the software, with the end goal of releasing high quality applications for demanding users. So, how should we define a highly available platform or solution to be able to do that with no *"service interruption"*?

Let's see what a *service interruption* means in this case.

### Industrial manufacturing, the example to look at

I think that software development automation is taking all the learnings from the industrial manufacturing processes and its *Continuous Delivery* systems where final products where tend to be manufactured on demand, saving a lot of unnecessary costs and making production lines a lot more agile with very high quality end user products (concepts like [JIT](https://en.wikipedia.org/wiki/Just-in-time_manufacturing), [Kanban](https://en.wikipedia.org/wiki/Kanban), [ConWIP](https://en.wikipedia.org/wiki/CONWIP), [Lean](https://www.sciencedirect.com/science/article/pii/S1877705814034092)... that are usually known for software development were born at the Industrial Manufacturing evolution during the 20th Century).

This is exactly what CI/CD is doing for software development. An [assembly line](https://en.wikipedia.org/wiki/Assembly_line) is the exact same concept for industrial manufacturing than a [CI/CD pipeline](https://dzone.com/articles/learn-how-to-setup-a-cicd-pipeline-from-scratch) for software development. Just an automated sequence of stages (can be parallel or not) that execute different steps to deliver high quality products in a frequent manner.

So, what about HA in manufacturing assembly pipelines? The first approach in the 1940's was to reduce the service interruption in a production line just by converting internal machine components to external (we could compare this as a decoupling method for manufacturing, similar to decouple monoliths to distributed components), so to repair a machine in the *pipeline* would impact much less in the loss of service for the entire assembly line. ([Toyota started to apply this](http://artoflean.com/index.php/2010/02/15/set-up-reduction-in-toyota/) in the 1950's for machine setup, reducing times from hours to minutes, with a huge impact in the case or restarting any *pipeline* stage in a loss of service).

But let's focus on how the industry faces this nowadays. Any manufacturing industry is able to assure production of hundreds or thousands of components every day from automated assembly lines just by applying different design methods in the *pipeline*. So, regarding this highly available processes, they apply things like:

* [**Cellular manufacturing**](https://en.wikipedia.org/wiki/Cellular_manufacturing). Just by a right decoupled architecture design of the pipeline and its flexibility, we can improve the flow of the execution and a fast recovery from a loss of service.
* [**Line buffers**](https://www.manufacturing.net/operations/article/13195207/buffers-and-merging-assembly-lines-balanced-or-unbalanced). By applying buffers in the assembly or production line, the cadence of production is not interrupted by a temporary loss of a workstation in the line. They also can improve the performance of the *pipeline execution*

![buffered_lines](/img/cicd-ha/buffered_line.png)

* [**Assembly line balancing**](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.2382&rep=rep1&type=pdf). Decoupling, replicating and parallelizing steps executions for assembly lines can reduce the recovery of the entire line for a highly available service.

Regarding the *"agreed level of operational performance"* from our previous HA definition, in manufacturing production lines there is no need to replicate the entire manufacturing platform or production lines to have a highly available service. This means that the lines can afford a stoppage of a couple of minutes with no impact on the cadence of production. From a business and production perspective this is a ***highly available service***.

One of the reasons of not working with replicated *active-active* running production lines as HA strategy is because is not cost effective for the service interruption recovery requirements. In other words, we get the same final results using easy scalable production lines that can recover fast from a failure, without the need of expensive *active-active* infrastructures.

### Resilient vs Highly Available

Once said that, one of the most important features about a *Continuous Delivery* system (note that I am talking in general, not only software) to be highly available, is **resilience**. Understood as the capacity of our system components to recover on time, producing a complete stable, durable and efficient execution that doesn't impact the overall service (e.g. my deployment or release frequency and quality of final artifacts are the same, with or without short downtimes. Just because my platform and systems are *resilient* to failures).

Let's back then to the question:

> ***What platform configuration do I need for a highly available service?***

To answer this question, it's very important to think about **resilience first**, because having a resilient platform and architecture is usually enough to deliver a highly available service, being able to recover on time to not impact the final service. But in some scenarios where a couple of minutes of service blockage means loss of millions of dollars (like Amazon retail previous example), resilient solutions are not enough. In this case then you probably need to work on very restrictive HA replication infrastructures. The cost of replication complexity need to be worth it.

## So CI/CD HA is just Resilience

Coming back to main topic... **YES**, it is about resilient solutions. HA in CI/CD is not about looking for an *active-active* load balancing platform that is able to orchestrate CI/CD pipelines. It is about assuring enough *availability* of my pipelines executions to deliver my software on time and covering all expectations (deployment frequency, failure recovery, lead time, quality standards, etc.).

What does resilient architecture mean in CI/CD pipeline orchestration? I think that following solution architecture concepts are a must in order to offer a resilient *highly available* solution:

* Orchestration layer does not have the same needs that execution layer. Pipeline orchestrators and pipeline executors have different scalability  requirements, and because of that **decoupling execution from orchestration is a must**.
* **Pipeline execution downtime recovery needs to be fast** and from the same execution point. Executors infrastructure can fail, even if they are replicated, so it is more important to recover from the last successful stage or status than moving the execution to another live agent executor that could restart the execution from the beginning
* **Decouple also the orchestration layer**. Smaller and more masters (orchestrators) are going to be more scalable and resilient to any overall failure. If one master fails it is not impacting any other orchestrator and its pipelines. So platform availability improves with scalability.
* Rely on a **native self-healing, resilient and scalable infrastructure**. Event thought that replication is not needed, we need to recover fast from any hardware or service downtime.
* **Decouple also pipeline execution**. Using different agent executors during pipeline execution improves availability, just the same way that *assembly lines* in the industry do.

It can be pretty cool if I can access my jobs or builds all the time because my masters are all the time up and running, but it is not that cool if they are not producing the same results in terms of building and delivery software because my pipelines are not executing and recovering on time. Again... resilience first.

## Scalability and Kubernetes as HA best practice

If we are looking for HA in any CI/CD solution architecture we should then think about a **scalable, decoupled and flexible solution** that runs on a **resilient platform**. Today, this terms gives us to think about containerized solutions orchestrated by [Kubernetes](https://kubernetes.io/).

### A traditional Jenkins example

[Jenkins](https://jenkins.io) is one of the most used solutions to orchestrate CI/CD pipelines. Its internal architecture is more than 10 years old, so we might think not to be the most advanced solution. But, the truth is that it has been evolving with these resilient best practices in mind (master-agent layers, pipeline decoupling and flexibility, containerized deployment, etc.), so if you deploy and design your platform and pipelines nicely you can get a true resilient CI/CD experience, meaning a CI/CD HA platform.

But let's see the following picture:

![Jenkins HA](/img/cicd-ha/CICD_HA_Archs.png)

In a traditional deployment - like the diagram on the left - with Jenkins masters (orchestrators) and Jenkins agents (executors), we should at least work on an *Active-Passive* replication on masters to recover on time to provide a resilient platform with HA capabilities. But, if we also decouple the masters with smaller ones and deploy them natively in Kubernetes - diagram in the right -, the experience can be much more resilient without any replication method. A small master, that is running in a pod, can automatically be recovered by Kubernetes if it fails, restarting its service in a couple of minutes in the worse case. The pipeline could still be running by the way. Also, the rest of the masters are not impacted and agents are just ephemeral, only running and restarted when needed. So, service loss is not perceived.

In a *Jenkins enterprise experience* this is a must, and that is why for example, [CloudBees Core](https://docs.cloudbees.com/docs/cloudbees-core/latest/) (usually known as *Jenkins Enterprise*) provides out of the box these capabilities with master management features and Kubernetes scalability.

### The Cloud Native evolution (a.k.a. Jenkins X)

But if we go purely Kubernetes native, and we provide an architecture where every piece of the pipeline orchestration and execution is just a state declaration object that can be recovered automatically when something goes down, then the HA and resilient experience is event better.

 [Jenkins X](https://jenkins-x.io) takes these concepts and all the previous mentioned best practices of scalability and resilience. Everything is about Kubernetes native objects definitions (based on [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)), which are resilient by nature from Kubernetes. No need to think about decoupled pipelines, infrastructure replication, or recovering scenarios. It is already out of the box from its solution architecture design. If you deploy your Kubernetes platform with some [Disaster Recovery](https://en.wikipedia.org/wiki/Disaster_recovery) configurations like Cloud cross-region availability, autoscaling pools and backup features (I think [Velero](https://velero.io/) is an interesting solution for K8s backup), it is practically impossible to completely loss the service for your CI/CD pipelines (except the case the World is ending...).

As an example, in the image below we can see that a pipeline execution on a default Jenkins X deployment recovered just in 30 seconds to continue running after a [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) failure simulation. Nothing needed to be done in terms of deployment or pipeline definition. It is just its native behavior.

![Jenkins X pod recovery](/img/cicd-ha/jenkins-x_podrestart.png)

## The message and conclusion

I've received in the past several redundant requests from organizations like following ones:

* *"We need to replicate and load balance our masters to have HA"*
* *"My masters need to be up and running all the time as Active-Active to have a continuously running CI/CD service"*
* *"How can I write to my shared storage at the same time with replicated masters?"* (another active-active HA conversation)

And many more similar.

Funny thing is that lots of cases were not prioritizing scalability, pipeline and master decoupling, or containerized orchestrated deployments. And then you realize that most of these requests come from IT requirements definitions in terms of HA general software architectures, no matter the nature of the service. 

Most of the people asking about HA is always referring to an *Active-Active* replication scenario and usually thinking of infrastructure, not the real business issue when comes to CI/CD. 

So it is very important to clarify that HA for CI/CD is not a matter of infrastructure replication. It is just a matter of resilient automated pipelines execution where usually just a couple of minutes of loss of service at some specific point is not even an issue... but you need to recover quickly and be stable on time.

I cannot imagine a car manufacturer like BMW or Toyota replicating their entire manufacturing plants to support loss of service for HA. True that is not comparable in costs to run a couple more CPUs, but the same decoupling, flexibility and scalability best practices apply to get the same HA experience as CI/CD.

Let's change our mindset. If we provide a scalable and distributed solution that deploys natively on a resilient platform, we don't even need to think about HA.