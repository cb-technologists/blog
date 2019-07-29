---
author:
  name: "Kurt Madel"
title: "Is Your Microservice Turning Brown?"
date: 2019-07-29T05:50:46-04:00
showDate: true
photo: "/img/microservice-turning-brown/brown-tree-glacier.png"
photoCaption: "Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 10.4 ƒ/5.6 1/400"
draft: true
tags: ["microservice","faas","nanoservice","kubernetes"]
---
## The Rise of Micro-services
Micro-service architecture has been one of the main areas of focus for [Greenfield](https://en.wikipedia.org/wiki/Greenfield_project) application delivery in the last half-decade+. Developing micro-service based APIs has coincided with the rise of containers and ephemeral cloud infrsatructure, and has required a new way to think about deployment and how applications discover each other.

But micro-services don't take full advantage of containerization and ephemeral cloud infrastructure. Although loosely coupled, a typical *micros-ervice* is always running - meaning they are always incurring infrastructure expenses, even when they are not being utilized. 

In a world where cloud infrastructure can be ephemeral, and where you pay for what you use when you use it - having constantly running services is a waste of money. Sure cost is important, but another very important aspect of delivering successful applications is making sure your customers have what they want when they want it. How can you accomplish both - the most efficient costs and expedient software delivery to meet customers' expectations.

## Smaller, Faster, Cheaper
The concept of nanoservices is evolving alongside the containerized delivery of application running on an evolving infrastructure of ephemeral, low cost cloud infrastructure. 

Kubernetes allows loose couping at the cloud vendor level, but it isn't easy. All of the major cloud platforms provide services and features that attempt to lock you into their platform. And I get it - I work for a software company and we want to keep customers as well.

It's just smaller than a microservice, right? Not exactly, a nanoservice is truly an architecture for Cloud Native applications and Native Kuberenetes CD allows you to easily build, test, and deploy them.

Nanoservices are Cloud Native.

Nanoservices are serverless.

Nanoservices are Functions-as-a-Service (FaaS). All the major cloud vendors have their own FaaS offerings - AWS Lambda, Azure , GCP 

One endpoint, one command.

Very low start-up time.
Very low latency between nanoservices and other services that utilize them.

The benefits of smaller, more flexible units now outweigh the costs.

"So far we have chosen not to use the serverless platforms offered by cloud providers, such as AWS’s Lambda, because they can’t guarantee such a low execution and cross-communication latency." - https://medium.com/bbc-design-engineering/powering-bbc-online-with-nanoservices-727840ba015b

## Gateway vs Mesh
The rise of microservices called for the need to communicate between them and provide the necessarry internal or external access to the services.
The containerization of microservices has made this even more complicated as containers are deployed to a cluster and there is no gurantee where they will end up.

API Gateways were first on the scene - but they had a lot of short comings when it came to intra-service communication. In most cases, a network call from one local service to another would have to go out and come back in through the API Gateway, resulting in increased latency that translated to slower response times for appliation consumers.

This becomes even more complicated in a multi-cloud or hybrid cloud deployment model. And although true multicloud is still a bit of a promise, not a reality - it is coming and with it a much more complex network topology for microservices to navigate.
Managed Istio across GCP and on-prem.

## Hybrid-Cloud or Multi-Cloud
Clear up some definitions.

Cloud agnostic, vendor lock-in, 

On-prem, in the cloud, cloud native....

## Enter Anthos
There has been a a lot of news about Anthos.

## What's Next
GKE on AWS and Azure - YES! There doesn't appear to be a timeline on this.