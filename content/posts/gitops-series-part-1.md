---
author:
  name: "Logan Donley"
title: "Introduction to GitOps - Part 1"
date: 2019-07-16T15:00:00-04:00
showDate: true
draft: false
tags: ["jenkins","gitops","gitops","kubernetes","cert-manager","google cloud platform"]
---

GitOps is a concept that was first coined by Weaveworks in their [GitOps - Operations by Pull Request](https://www.weave.works/blog/gitops-operations-by-pull-request) post. The idea itself wasn't anything particularly new, people had been doing automated operations with infrastructure-as-code for years. But now that there was a descriptive new name for this concept, the DevOps community has really started to embrace it. Especially with the ever growing prevalence of Kubernetes.

![GitOps Trend](/img/gitops-series/part-1/trend.png)

If you haven't already done so, I'd recommend reading that Weaveworks post since it is always good to understand the origination of a concept. But simply put, GitOps is a way for you to manage your operations from a source code repo. With GitOps you won't be doing any manual steps when you want to change something. On a commit to your master branch, a job will get kicked off and will make the necessary changes.

## Why would you want this?

If you have any experience on an operations team with little or no automation, you no doubt know the frustration of manual configuration and snowflake servers. In this sort of environment it is easy to quickly get overwhelmed. And when things go wrong, they can spin out of control.

As you move to infrastructure-as-code by using configuration management tools you're able to get away from most of that headache. Now that you have code which describes your desired environment state you have an easy way to manage and maintain the environment. When you need to change something, submit a pull request and have someone review it. Once the change is merged, go ahead and run the automation and the change will propogate. Should something disastrous happen to your environment, you can get your environment back up in no time by rerunning the automation.

But you are still missing something if these commits to master don't automatically kick off a job to make the change. You should always want your environment to 100% match the configuration in your repo. If the automation isn't automatically run, you will drift away from the target state. 

I've personally been guilty of putting off running automation due to fear of something breaking, and I know I'm not alone in this. Knowing that merging your pull request is going to trigger a job makes you more careful in your review of pull requests but also gives you confidence in knowing that your environment is always up-to-date.


## Objective of this series

To explore the idea of GitOps I am writing a 3-part series where we'll be building out a fully-functional GitOps process. 

In this first part we will take a look at building out the infrastructure automation piece. This will involve provisioning a Kubernetes cluster, setting up certificates and DNS, and more. From here we will fork into two different directions.

In the second part we will add the automation of our [CloudBees Jenkins Distribution](https://www.cloudbees.com/products/cloudbees-jenkins-distribution). This will include plugin management, configuration, and more.

In the final part we will look at [CloudBees Core](https://www.cloudbees.com/products/cloudbees-core) and some cool stuff we can do with custom Operations Center and Managed Master images.

Overall, while the goal of this series is educational, I hope it is also useful and that the assets are useable. As I write this I am using this automation daily to make my life easier as I play around with new features and try new configurations.

By necessity the resulting assets are based on my configuration and preferences. While I am trying to keep things as generic as possible, some things like my use of GKE might differ from your situation. The ideas and processes should be transferable to your environment.


# Time to get down to business

With that background out of the way, let's dive right in.

## What's the plan?

![Plan of attack](/img/gitops-series/part-1/plan.svg)

1. Pull latest changes - we'll leave this to Jenkins
2. Kubernetes cluster - for this I have decided to use [Terraform](https://www.terraform.io/) to provision/manage the Kubernetes cluster. We'll be using [(GKE) Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) since it is my favorite managed Kubernetes platform
3. Namespace and permissions - will use `kubectl` to handle this
4. Ingress controller - Will use the recommended nginx ingress controller
5. Set DNS record - Will take advantage of the Ansible role to do this easily
6. Cert manager - Will use Kubectl to install
7. Ensure CJD/Core are running - This is where we will fork into the next 2 posts


### Already there is an issue

In order to have a GitOps process that works, we need to have something to actually kick off the jobs. In this case we're going to be using Jenkins, but we don't have it up yet. 

There are a couple of ways we could handle this: 

1. Have a Jenkins server running outside of the Kubernetes cluster (where is the fun in that?)
2. Create a seed script which will run everything the first time and setup the Jenkins server on the cluster we just created

Since I am trying to minimize the number of things we need to manage, we'll be going with #2.


## Creating the repo

In order to keep the separations of concerns pretty straightforward, I've got the structure of the repo looking like this:

```
.
├── ansible <- Our ansible playbook will live here
├── cert-manager <- The cert manager configuration lives here
├── scripts <- All other scripts (including our seed script) live here
└── terraform <- The terraform configuration lives here
```


## Provisioning the Kubernetes cluster

The first obvious step in building out our project is to be able to spin up the Kubernetes cluster, since after all, that is the platform everything will be running on. For this task I've chosen to use Terraform for it's quick and easy way to provision cloud resources in an [idempotent](https://en.wikipedia.org/wiki/Idempotence) fashion. 

Specifically we'll use the Google Kubernetes Engine (GKE) [provisioner](https://www.terraform.io/docs/providers/google/r/container_cluster.html).

If you don't have any experience with Terraform, they have a good [learning site](https://learn.hashicorp.com/terraform/) where you can get started. The scope of what we'll be doing is rather limited, so if you don't have any prior experience don't worry, it should be simple enough to follow and understand.

At a minimum you will want to have Terraform [installed locally](https://learn.hashicorp.com/terraform/getting-started/install).

### Variables file

When using Terraform I like to split out all of the variables into a separate variables file. This makes it easier when making changes to see all settings at once.

Inside of our `terraform/` directory will start by creating a `variables.tf` file. 

In Terraform, a variable looks like this:
```terraform
variable "cluster_name" {
  default = "ld-cluster-1"
}
```

This can then be referenced like this: `"${var.cluster_name}"`

The `terraform/variables.tf` file is going to look like this:

```terraform
variable "project" {
  default = "myproject"
}

variable "region" {
  default = "us-east1-b"
}

variable "cluster_name" {
  default = "my-cluster-name"
}

variable "cluster_zone" {
  default = "us-east1-b"
}

variable "cluster_k8s_version" {
  default = "1.13.6-gke.13"
}

variable "initial_node_count" {
  default = 1
}

variable "autoscaling_min_node_count" {
  default = 1
}

variable "autoscaling_max_node_count" {
  default = 5
}

variable "disk_size_gb" {
  default = 100
}

variable "disk_type" {
  default = "pd-standard"
}

variable "machine_type" {
  default = "n1-standard-2"
}
```

Since this is where we are setting the environment specific variables, go ahead and replace those with your own desired state. You'll most likely want to adjust `project`, `region`, `cluster_zone`, and `cluster_name`. There is also a chance that as you read this the `cluster_k8s_version` I have listed here is no longer available, so you may need to update that.

### Cluster definition file

Now with the variables out of the way, it's time to build out the actual definition of what the cluster is going to look like. This is the stuff that isn't likely to change as much. If you're following along you shouldn't need to make any changes except for one specific spot I'll point out.

We're going to create a `cluster.tf` file.

It's going to look like this: `terraform/cluster.tf`

```terraform
provider "google" {
  project = "${var.project}"
  region  = "${var.region}"
}

# Change this section
terraform {
  backend "gcs" {
    bucket  = "my-unique-bucket"
    prefix  = "terraform/state"
    project = "my-project"
  }
}

resource "google_container_cluster" "cluster" {
  name               = "${var.cluster_name}"
  location           = "${var.cluster_zone}"
  min_master_version = "${var.cluster_k8s_version}"

  addons_config {
    network_policy_config {
      disabled = true
    }

    http_load_balancing {
      disabled = false
    }

    kubernetes_dashboard {
      disabled = false
    }
  }

  node_pool {
    name               = "default-pool"
    initial_node_count = "${var.initial_node_count}"

    management {
      auto_repair = true
    }

    autoscaling {
      min_node_count = "${var.autoscaling_min_node_count}"
      max_node_count = "${var.autoscaling_max_node_count}"
    }

    node_config {
      preemptible  = false
      disk_size_gb = "${var.disk_size_gb}"
      disk_type    = "${var.disk_type}"

      machine_type = "${var.machine_type}"

      oauth_scopes = [
        "https://www.googleapis.com/auth/devstorage.read_only",
        "https://www.googleapis.com/auth/logging.write",
        "https://www.googleapis.com/auth/monitoring",
        "https://www.googleapis.com/auth/service.management.readonly",
        "https://www.googleapis.com/auth/servicecontrol",
        "https://www.googleapis.com/auth/trace.append",
        "https://www.googleapis.com/auth/compute",
        "https://www.googleapis.com/auth/cloud-platform"
      ]

    }
  }
}

output "client_certificate" {
  value     = "${google_container_cluster.cluster.master_auth.0.client_certificate}"
  sensitive = true
}

output "client_key" {
  value     = "${google_container_cluster.cluster.master_auth.0.client_key}"
  sensitive = true
}

output "cluster_ca_certificate" {
  value     = "${google_container_cluster.cluster.master_auth.0.cluster_ca_certificate}"
  sensitive = true
}

output "host" {
  value     = "${google_container_cluster.cluster.endpoint}"
  sensitive = true
}
```

This may look complicated, but really all we are doing is defining the configuration of the cluster we want to provision. As you can see we are taking full use of the variables we listed in the `variables.tf` file.

There is one section you will need to modify, it is the following block:

```terraform
terraform {
  backend "gcs" {
    bucket  = "my-unique-bucket"
    prefix  = "terraform/state"
    project = "my-project"
  }
}
```

By default, when you are using Terraform it stores the state of your environment to the local system. Since we are going to be running this from Jenkins in an ephemeral agent, we don't want this. Instead, this block tells Terraform to store the state to a GCS storage bucket so the state will persist between runs.

If you're following along, you can follow these [instructions](https://cloud.google.com/storage/docs/creating-buckets) to create a GCS bucket here.

### Testing it out

With this configuration all set, we are ready to test it out and see if we can provision a GKE cluster using terraform.

If you `cd terraform` to change to that directory, you can initialize the Terraform project and pull the requisite plugins by running `terraform init`. You can then run `terraform plan` to see the plan that gets generated by Terraform.

If all looks good, you can go ahead and run `terraform apply`. Unless you specify a specific flag, it is going to prompt you whether you want to perform the actions or not.

Go ahead and type `yes` when you're ready, then the provisioning process will begin. This should take a few minutes to complete since it has to spin up and configure quite a few resources.

Once the cluster is up and ready to go, we can move on to the next steps.

## Setting up namespace and permissions

These steps we're performing will be put into a script since automation is our goal, but I'm going to run through them manually the first time so we can understand what is going on.

First we need to connect to the Kubernetes cluster we created. This is easiest done by running (with your specific parameters):

`gcloud container clusters get-credentials MYCLUSTER --zone MYZONE --project MYPROJECT`

You can verify that you're connected by running a kubectl command like `kubectl get nodes`.

### Assigning cluster-admin role

Certain components of our setup will need cluster-admin role access so we can easily set that up by running:

`kubectl create clusterrolebinding cluster-admin-binding  --clusterrole cluster-admin  --user $(gcloud config get-value account)`

### Create the namespaces

Next we will want to create a namespace for Core or CJD to live in. 

```bash
kubectl create namespace core
kubectl label namespace core name=core
kubectl config set-context $(kubectl config current-context) --namespace=core
```

If you're familiar with kubectl you might be aware that we are going to get an error on subsequent runs of the `kubectl create` command since it will already exist. We will need to take care of that as part of the Jenkinsfile.

## Setup the ingress controller

In order to get traffic into an application running in Kubernetes we will need to create ingresses for each application. It turns out that manually doing this is a bit of a pain, so the Kubernetes community created the [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) which will do most of the work for us.

There are several different ways to install this, including a simple helm install, all of which can be found [here](https://kubernetes.github.io/ingress-nginx/deploy/).

To avoid having to manage anything else (i.e. helm), I've opted to just use the yaml file install.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/provider/cloud-generic.yaml
```

An important note about this is that it takes several seconds to provision and attach the public ip address to the service. We will need to handle this in the Jenkinsfile.

## Set DNS Record

Now that we have a public ip address, we can point our domain at it. This is one of those problems that you can tackle 100 different ways. The simplest way is probably to make an api call to your DNS host to update a particular record with the ip address.

I'm going to make it a little more complicated in order to make it easier to switch between different DNS hosts. 

I've setup an [Ansible](https://github.com/ansible/ansible) playbook which takes advantage of the pre-built DNS provider modules.

Here are the [Ansible install instructions](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html). If you've got `pip` on your system you can simply run `pip install ansible`.

I created a file `ansible/dns.yml` which contains the following:

```yaml
- hosts: localhost
  tasks:
  - name: Create a managed zone
    gcp_dns_managed_zone:
      name: "my-zone-name"
      dns_name: "my-domain-name.com"
      description: My playground
      project: my-project
      state: present
      auth_kind: serviceaccount
    register: managed_zone

  - name:  Create an A record to point to the Core instance
    gcp_dns_resource_record_set:
      managed_zone: "{{ managed_zone }}"
      name: "core.my-domain-name.com."
      type: A
      target: 
        - "{{ target_ip }}"
      project: my-project
      auth_kind: serviceaccount
```

What we are doing here is taking advantage of the gcp_dns modules ([`gcp_dns_managed_zone`](https://docs.ansible.com/ansible/latest/modules/gcp_dns_managed_zone_module.html) & [`gcp_dns_resource_record_set`](https://docs.ansible.com/ansible/latest/modules/gcp_dns_resource_record_set_module.html)) to easily set the DNS.

The nice thing about this is should you need to use another DNS host like [Cloudflare](https://www.cloudflare.com/) you can easily transition over using the right [module](https://docs.ansible.com/ansible/latest/modules/cloudflare_dns_module.html#cloudflare-dns-module).

Once that is configured, you can run the playbook with (setting the TARGET_IP according to your cluster's ip):

`ansible-playbook ansible/dns.yml -e target_ip=${TARGET_IP}`

## Setting up Cert Manager

Now we have our ingress controller which has given us a public ip address and we have setup a DNS record to point to the address. We could go ahead and install Core or CJD at this point if we wanted, but we might as well setup SSL certificates to make things more secure.

We're going to use [Let's Encrypt](https://letsencrypt.org/) Certificate Authority in order to generate the certs. To do this in an easy and automated fashion, we'll use [cert-manager](https://github.com/jetstack/cert-manager). 

Installing it is pretty easy, and you can find the most update instructions [here](https://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html).

```bash
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.8.0/cert-manager.yaml
```

The above commands create a namespace for cert-manager and then deploy the cert-manager application in there. 

Next we'll need to create some cert issuers to allow us to actually generate the certs. We'll create two of them in the cert-manager directory.

`cert-manager/staging-issuer.yaml`:
```yaml
   apiVersion: certmanager.k8s.io/v1alpha1
   kind: Issuer
   metadata:
     name: letsencrypt-staging
   spec:
     acme:
       # The ACME server URL
       server: https://acme-staging-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
       email: myemail@example.com # change this
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-staging
       # Enable the HTTP-01 challenge provider
       http01: {}
```

`cert-manager/production-issuer.yaml`:
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
       email: myemail@example.com
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-prod
       # Enable the HTTP-01 challenge provider
       http01: {}
```

The reason we have two of these is because Let's Encrypt has a rate-limiter on how often you can generate certificates. So while you are experimenting with things, it is safer to use the staging issuer. When things are all sorted, you can switch to the production-issuer.

Go ahead and apply these two issuers with:

```bash
kubectl apply -f cert-manager/staging-issuer.yaml
kubectl apply -f cert-manager/production-issuer.yaml
```

Now when we create an ingress for our applications, we can add some metadata to the ingress definition and the certificates will automatically be generated and stored as [K8s Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

## Setting up CloudBees Core or CJD

The final step in this flow is to either setup Core or CJD. We will add this portion, and more in the following posts.

But before concluding this post, let's take a look at how we can put all of these steps into a Jenkinsfile.

## Putting it all together.

Since the whole objective here was to automate this process, it makes sense to use Jenkins to run the process. We can easily achieve GitOps by having every commit to master kick off our pipeline here.

You'll note that we've added a couple of dependencies along the way that we'll need to make sure Jenkins will have access to. Thankfully, since we'll be running on Kubernetes, we can take advantage of the ephemeral, container-based agents. We can define a [pod template](https://jenkins.io/doc/pipeline/steps/kubernetes/) which will describe all of the containers we will need.

In the root directory of my repo, I have created a `pod-template.yml` file:

```yaml
kind: Pod
metadata:
  name: gitops-pod
spec:
  containers:
  - name: terraform
    image: hashicorp/terraform:light
    command:
    - cat
    tty: true
    volumeMounts:
      - name: gcp-credential
        mountPath: /root/
    env:
      - name: GOOGLE_CLOUD_KEYFILE_JSON
        value: "/root/gcp-service.json"
      - name: GCP_SERVICE_ACCOUNT_FILE
        value: "/root/gcp-service.json"
      - name: GOOGLE_APPLICATION_CREDENTIALS
        value: "/root/gcp-service.json"
  - name: ansible
    image: ldonleycb/ansible-ci:new
    command:
    - cat
    tty: true
    volumeMounts:
      - name: gcp-credential
        mountPath: /root/
    env:
      - name: GOOGLE_CLOUD_KEYFILE_JSON
        value: "/root/gcp-service.json"
      - name: GCP_SERVICE_ACCOUNT_FILE
        value: "/root/gcp-service.json"
      - name: GOOGLE_APPLICATION_CREDENTIALS
        value: "/root/gcp-service.json"
      - name: GCP_PROJECT
        value: "my_project"
      - name: GCP_CLUSTER_NAME
        value: "my_cluster_name"
  - name: kubectl
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
      - name: GCP_SERVICE_ACCOUNT_FILE
        value: "/home/gcp-service.json"
      - name: GOOGLE_APPLICATION_CREDENTIALS
        value: "/home/gcp-service.json"
      - name: GCP_PROJECT
        value: "my_project"
      - name: GCP_CLUSTER_NAME
        value: "my_cluster_name"
  volumes:
    - name: gcp-credential
      secret:
        secretName: gcp-credential
```

This looks complicated, but it is mostly just bloated by the array of environment variables we need for Google Cloud operations.

The `Jenkinsfile` in the root directory will look something like this:

```groovy
pipeline {
  agent {
    kubernetes {
      label 'gitops'
      yamlFile 'pod-template.yml'
    }
  }
  stages {
    stage('Terraform') {
      steps {
        container('terraform'){
          sh '''
            cd terraform
            terraform init
            terraform apply -input=false -auto-approve
            cd ..
          '''
        }
      }
    }
    stage('Setup ingress controller and namespace') {
      steps {
        container('kubectl'){
          script {
            sh '''
              gcloud auth activate-service-account --key-file=$GCP_SERVICE_ACCOUNT_FILE
              gcloud container clusters get-credentials $GCP_CLUSTER_NAME --zone us-east1-b --project $GCP_PROJECT
            '''
            try {
              sh '''
                kubectl create clusterrolebinding cluster-admin-binding  --clusterrole cluster-admin  --user $(gcloud config get-value account)
              '''
            }
            catch(error) {
              sh "echo ''"
            }

            sh '''
              kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/mandatory.yaml
              kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/provider/cloud-generic.yaml
              sleep 60s
            '''
            try {
              sh '''
                kubectl create namespace core
                kubectl label namespace core name=core
              '''
            }
            catch(error) {
              sh "echo ''"
            }
            sh '''
              kubectl config set-context $(kubectl config current-context) --namespace=core
            '''
            env.TARGET_IP = sh(returnStdout: true, script: 'kubectl get service ingress-nginx -n ingress-nginx | awk \'END {print $4}\'').trim()
          } 
        }
      }
    }
    stage('Setup DNS') {
      steps {
        container('ansible'){
          sh """
            ansible-playbook ansible/dns.yml -e target_ip=${env.TARGET_IP}
          """
        }

      }
    }
    stage('Setup cert-manager') {
      steps {
        container('kubectl'){
          sh '''# Install cert-manager
              kubectl create namespace cert-manager
              kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
              kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.8.0/cert-manager.yaml

              sleep 30s

              # Add cert-manager issuers
              kubectl apply -f cert-manager/staging-issuer.yaml
              kubectl apply -f cert-manager/production-issuer.yaml
            '''
        }
      }
    }
  }
}
```

This is not a particularly elegant solution at this point, but for an initial attempt it should be sufficient.

In the next parts of this series we will be taking a look at how to extend this to actually deploy and maintain CloudBees Core or CloudBees Jenkins Distribution.