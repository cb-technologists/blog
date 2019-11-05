---
authors:
  - "Miguel Suarez"
title: "Using Windows containers with Jenkins on Kubernetes 1.14"
date: 2019-10-27T17:00:00-04:00
showDate: true
draft: false
tags: ["jenkins","kubernetes","windows", "azure", "AKS", "EKS"]
---
Kubernetes 1.14 was released in March 2019 and the release brought [production support](https://kubernetes.io/docs/setup/production-environment/windows/intro-windows-in-kubernetes/#windows-containers-in-kubernetes) for Windows Containers on Windows Server nodes. Before moving on, I would like to highlight a few things from the previous link:  

1. Kubernetes control plane runs in Linux (and there is no plan to change that for a full Windows Kubernetes cluster)
2. Versions supported for worker nodes and containers: Windows Server 1809/Windows Server 2019
3. **Windows containers have to be scheduled on Windows nodes** 

At the time this post was written (Oct 19), [AKS](https://azure.microsoft.com/en-us/blog/announcing-the-preview-of-windows-server-containers-support-in-azure-kubernetes-service/), [GKE] (https://cloud.google.com/blog/products/containers-kubernetes/how-to-deploy-a-windows-container-on-google-kubernetes-engine) and [EKS] (https://aws.amazon.com/blogs/aws/amazon-eks-windows-container-support-now-generally-available/) offer some level of support for Windows based containers (EKS is the first to offer GA support for Windows based containers). The [EKS documentation](https://aws.amazon.com/blogs/aws/amazon-eks-windows-container-support-now-generally-available/) provides instructions on how to setup Windows nodes using ```eksctl``` and EKS. You might want to try that since it is GA. The following are the instructions for AKS 1.14+ (where Windows Containers are still under preview) but the example and instructions related to Jenkins we will use should be able to be used in a similar setup where Kubernetes has a Windows node pool. 

## Infrastructure Setup and Windows Nodepools

Jenkins has the ability to use Kubernetes pods as agents to build and deploy applications thanks to the [Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin). In this blog post, we will create a simple declarative pipeline that has Linux and Windows containers as agents in AKS. 



First, we need to follow [Azure's documentation](https://docs.microsoft.com/en-us/azure/aks/windows-container-cli#before-you-begin) to create the needed infrastructure to be able to deploy Linux and Windows based containers. Please make sure that you review AKS documentation and are aware of the limitations before running this in a production cluster. As explained [here](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools), the registration for the preview features cannot be unregistered at this moment. 

The [documentation](https://docs.microsoft.com/en-us/azure/aks/windows-container-cli) at a high level goes through the following steps (using the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)):

1. Install aks-preview CLI extension for Azure CLI
2. Register the Windows preview feature needed for the Windows based containers
    * As mentioned in the document, the [Multiple Node Pool feature](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools) is also needed to create a separate Windows node pool.
3. Create a new resource group (if needed)
4. Create an AKS cluster
    * You can use ```--nodepool-name``` with ```aks create cluster``` to name your control plane node pool i.e ```default```
5. Add a Windows Server node pool 
    * This will be a node pool for kubernetes pod Windows agents, we can name it ```--name winage```

[Node pools](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools) give us the possibility to extend our Kubernetes cluster with more types of machines depending on our use and budget (See available [options and default values](https://docs.microsoft.com/en-us/cli/azure/ext/aks-preview/aks/nodepool?view=azure-cli-latest#ext-aks-preview-az-aks-nodepool-add)) for AKS. As an example, we are going to add two more identical pools (one for masters and one for Linux agents) but you can pick different machine sizes and node counts depending on your need (just make sure that the VMs used for Jenkins masters support Premium Storage as Jenkins requires high IOPS for better performance):
    
* Linux Jenkins master pool example:

    ```    
    az aks nodepool add \ 
    --resource-group myResourceGroup \
    --cluster-name eastUSAKS \
    --os-type Linux \
    --name masters \
    --node-count 1 \
    --kubernetes-version 1.14.6 \
    --node-vm-size Standard_DS2_v2
    ```

* Linux Jenkins agent pool example:

    ```
    az aks nodepool add \                                                                 --resource-group myResourceGroup \
    --cluster-name eastUSAKS \
    --os-type Linux \
    --name linage \
    --node-count 1 \
    --kubernetes-version 1.14.6 \
    --node-vm-size Standard_DS2_v2
    ```
    Once the pools are created, you can see them in the Azure portal:

    [![](/img/windows-containers/AKS-node-pools.png)](/img/windows-containers/AKS-node-pools.png)


## Jenkins installation using Helm

[Helm](https://helm.sh/) is the Kubernetes Package Manager and we can use it to install Jenkins using a chart([Jenkins chart](https://github.com/helm/charts/tree/master/stable/jenkins)). If you haven't installed Helm before, you can follow [these instructions](https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm) to install it. Using a [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) in the values file (```values.yaml```) used by the chart will allow us to specify in which nodepool Jenkins will be installed (you can find ```master.nodeSelector``` option in the Jenkins chart link). For simplicity, we are only going to configure the ```values.yaml``` file so that it deploys Jenkins using such ```nodeSelector``` option but the file can include a lot more options.  In this case, we need to make sure that Jenkins runs in the nodepool named ```masters``` (AKS assigns the nodepool name as the value of the ```agentpool``` tag, more on this in the next section)

* ```values.yaml```

```
master:
  nodeSelector:
    agentpool: masters
```
* Installing the chart (add ```--namespace yourNamespace``` to the command if you want to deploy Jenkins in a specific namespace):

```
helm install --name jenkins -f values.yaml stable/jenkins 
```

The version that was installed in this example is 2.190.2.

Once installed, follow the "NOTES" section in the console that will allow you to get your Jenkins (user: admin) password and URL. It will include something similar to this:

```
printf $(kubectl get secret jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo 

export SERVICE_IP=$(kubectl get svc jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")

echo http://$SERVICE_IP:8080/login

```
You should be able to access Jenkins with the provided URL at this point.

## Creating the pipeline with Windows and Linux Containers

Make sure to check the version of the Kubernetes plugin installed. This example uses the version 1.21.0 which supports the [Windows container step](https://github.com/jenkinsci/kubernetes-plugin/releases/tag/kubernetes-1.21.0).

* Let's create a pipeline by clicking ```"New Item"```, then enter a name for your pipeline job (i.e ```"win-lin-pipeline"```) and select ```"Pipeline"``` as your job type. 

- Select: 

    * Definition: ```"Pipeline script from SCM"```
    * SCM: ```Git```
    * Repository URL: ```https://github.com/mluyo3414org/pod-templates.git ```

 [![](/img/windows-containers/pipeline-options.png)](/img/windows-containers/pipeline-options.png)

- Click ```Save```

Let's take a look at the repository structure in ```https://github.com/mluyo3414org/pod-templates```:

```

├── Jenkinsfile
├── README.md
├── linux
│   └── nodejs-pod.yaml
└── windows
    └── dotnet-pod.yaml

```
- Both ```nodejs-pod.yaml``` and ```dotnet-pod.yaml``` are files describing the Kubernetes pod agents used in the Jenkinsfile. 

- The ```dotnet-pod.yaml``` has two container definitions: a **Windows based** ```jnlp```(```jenkins/jnlp-agent:latest-windows```) and a ```windows-dotnet``` container (```mcr.microsoft.com/dotnet/core/sdk:2.1```). We need to overwrite the ```jnlp``` container in this pod since otherwise it will use the **Linux based** ```jnlp``` container defined under ```Jenkins --> Configuration --> Pod templates```. In this case, Jenkins was automatically configured with the Linux pod: ```jenkins/jnlp-slave:3.27-1```.

```
kind: Pod
metadata:
  name: windows
spec:
  containers:
  - name: jnlp
    image: jenkins/jnlp-agent:latest-windows
    tty: true
  - name: windows-dotnet
    image: mcr.microsoft.com/dotnet/core/sdk:2.1
    tty: true
  nodeSelector:
    agentpool: winage
```

- The ```nodejs-pod.yaml``` has the ```node```(nodeJS) container definition and will use the default Linux based ```jnlp``` (```jenkins/jnlp-slave:3.27-1```) mentioned before.

```
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:slim
    command:
    - cat
    tty: true
  nodeSelector:
    agentpool: linage
```

- Notice both pod yaml definitions use  ```nodeSelector``` ([nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)) to decide where these pods should be scheduled. If this is not specified, the Kubernetes scheduler will provision the pods following the [default behavior](https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes/) which could possibly start a pod in the a node with the wrong OS. The tags used for the nodeSelectors are the default tags assigned by Azure when specifyng the nodepool name: ``` agentpool : nameOfNodePool ```. To find the node tags you can use the command: ``` kubectl get nodes --show-labels ```. Other options to prevent scheduling errors are ```taints```  ([more info] (https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)).
  


    [![](/img/windows-containers/error.png)](/img/windows-containers/error.png)*Scheduling error when not using nodeSelectors. Debug using ```kubectl describe podname```*
  

- Jenkinsfile:

```
pipeline {
  agent none
  options { 
    buildDiscarder(logRotator(numToKeepStr: '2'))
    skipDefaultCheckout true
  }
  stages {
    stage('Test-linux') {
      agent {
        kubernetes {
          label 'nodejs-pod'
          yamlFile 'linux/nodejs-pod.yaml'
        }
      }
      steps {
        checkout scm
        container('nodejs') {
          echo 'Hello World!'   
          sh 'node --version'
        }
      }
    }
    stage('Test-windows') {
      agent {
        kubernetes {
	  label 'windows-pod'
          yamlFile 'windows/dotnet-pod.yaml'
        }
      }
      steps {
        bat 'dir'
        container(name:'windows-dotnet'){
          bat 'dotnet -h'
      } 
     }
    }
  }
}
```


- This is a declarative pipeline using agent ```none```([none](https://jenkins.io/doc/book/pipeline/syntax/#agent) info) so that we can specify agents per stage. ```yamlFile``` is used to read the pod template from a file location which is also in this repo. You can either define the pod template in another file, in the same [Jenkinsfile](https://github.com/jenkinsci/kubernetes-plugin/blob/master/examples/declarative-multiple-containers.groovy) or in the Jenkins Configuration page. More info about the syntax used in this pipeline can be found [here](https://jenkins.io/doc/book/pipeline/syntax/). 


- The ```node --version``` command is executed inside the ```container``` step otherwise it will get executed inside the jnlp (Linux) container and fail as it doesn't have node installed. Similarly ```bat 'dir ``` gets executed in the jnlp (Windows) container and ```bat 'dotnet -h' ``` is executed inside the dotnet container.



- [Here](https://github.com/jenkinsci/kubernetes-plugin/blob/kubernetes-1.21.0/examples/windows.groovy) is another example on how to use a Windows container using a scripted pipeline and tested in EKS.

- [Windows Containers on the Kubernetes Podcast](https://open.spotify.com/episode/0XXYzjBEj12S39rwcobJ70?si=J-tEvxR8S_qsmJcd-FGn5A)

 [![](/img/windows-containers/node-pools-distribution.png)](/img/windows-containers/node-pools-distribution.png)*This is a high-level diagram of the Kubernetes cluster and containers.*





