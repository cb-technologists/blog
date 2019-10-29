---
authors:
  - "Miguel Suarez"
title: "Using Windows containers with Jenkins on Kubernetes 1.14 - AKS preview"
date: 2019-10-27T17:00:00-04:00
showDate: true
draft: false
tags: ["jenkins","kubernetes","windows", "azure", "AKS"]
---
Kubernetes 1.14 was released in March 2019 and the release brought [production support](https://kubernetes.io/docs/setup/production-environment/windows/intro-windows-in-kubernetes/#windows-containers-in-kubernetes) for Windows Containers on Windows Server nodes. Before moving on, I would like to highlight a few things from the previous link:  

1. Kubernetes control plane runs in Linux (and there is no plan to change that for a full Windows k8s cluster)
2. Versions supported for worker nodes and containers: Windows Server 1809/Windows Server 2019
3. **Windows containers have to be scheduled on Windows nodes** 

At the time this post was written, [AKS](https://azure.microsoft.com/en-us/blog/announcing-the-preview-of-windows-server-containers-support-in-azure-kubernetes-service/), [GKE] (https://cloud.google.com/blog/products/containers-kubernetes/how-to-deploy-a-windows-container-on-google-kubernetes-engine) and [EKS] (https://aws.amazon.com/blogs/aws/amazon-eks-windows-container-support-now-generally-available/) offer Windows based containers at some level of support (EKS is the first to offer GA support for Windows based containers).

## Infrastructure Setup 

Jenkins has the ability to use containers as dynamic agents to build and deploy applications thanks to the [Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin). In this blog post, we will create a simple declarative pipeline that has Linux and Windows containers as agents in AKS. 



First, we need to follow [Azure's documentation](https://docs.microsoft.com/en-us/azure/aks/windows-container-cli#before-you-begin) to create the needed infrastructure to be able to deploy Linux and Windows based containers. Please make sure that you review AKS documentation and are aware of the limitations before running this in a production cluster. As explained [here,](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools) the registration for the preview features cannot be unregistered at this moment. The [EKS documentation](https://aws.amazon.com/blogs/aws/amazon-eks-windows-container-support-now-generally-available/) provides quite similar instructions for eksctl and EKS. You might want to try that since it is GA. The example and instructions related to Jenkins we will demonstrated might be able to be used in a similar setup in EKS with a Windows node pool. 

The documentation at a high level goes through the following steps (using the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)):

1. Install aks-preview CLI extension for Azure CLI
2. Register the Windows preview feature needed for the Windows based containers
    * As mentioned in the document, the [Multiple Node Pool feature](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools) is also needed to create a separate Windows node pool.
3. Create a resource group (if no existing resource group for AKS deployment exists)
4. Create an AKS cluster
    * You can use ```--nodepool-name``` with ```aks create cluster``` to name your default node pool i.e ```default```
5. Add a Windows Server node pool 
    * This will be a node pool for agents, we can name it ```--name winage```

Node pools gives us the possibility to extend our Jenkins cluster for more types of machines depending on our use and budget (See available [options and default values](https://docs.microsoft.com/en-us/cli/azure/ext/aks-preview/aks/nodepool?view=azure-cli-latest#ext-aks-preview-az-aks-nodepool-add)). As an example, we are going to add two more identical pools but you can pick different machine sizes and node counts depending on your need (just make sure that the master VMs supports Premium Storage as Jenkins requires high IOPS):
    
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
    az aks nodepool add \                                                                              
    --resource-group myResourceGroup \
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

[Helm](https://helm.sh/) is the Kubernetes Package Manager and we can use it to install Jenkins. If you haven't installed Helm before, you can follow [these instructions](https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm) to install it. [Using a nodeSelector will allow us to specify in which node Jenkins will be installed](https://github.com/helm/charts/tree/master/stable/jenkins). For simplicity, we are only going to configure the ```values.yaml``` file so that it deploys Jenkins using a [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) but the file can include a lot more configurations.  In this case, we need to make sure that Jenkins runs in the node-pool named ```masters```.

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

The version installed in this example is 2.190.2.

Once installed, follow the "NOTES" section in the console that will allow you to get your Jenkins (user: admin) password and URL. It will include something similar to this:

```
printf $(kubectl get secret jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo 

export SERVICE_IP=$(kubectl get svc jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")

echo http://$SERVICE_IP:8080/login

```
You should be able to access Jenkins with the provided URL at this point.

## Creating the pipeline with Windows and Linux Containers

Make sure that the Kubernetes plugin installed is updated. This example uses the version 1.20.2 and had to be updated as the Kubernetes plugin version in the chart was not able to create a pod agent with this Kubernetes cluster.

Let's create a pipeline by clicking ```"New Item"```, then enter a name for your pipeline job (i.e ```"win-lin-pipeline"```) and select ```"Pipeline:"```. 

Select: 

* Definition: ```"Pipeline script from SCM"```
* SCM: ```Git```
* Repository URL: ```https://github.com/mluyo3414org/pod-templates.git ```

 [![](/img/windows-containers/pipeline-options.png)](/img/windows-containers/pipeline-options.png)

Click ```Save```

**Note**: sometimes images might take some time to download. Check if the pod timesout.





