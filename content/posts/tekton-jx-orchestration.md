# Jenkins X and it's orchestration power

We may know Jenkins X as the new pure cloud native implementation of Jenkins. It is based on the use of Kubernetes Custom Resource Definitions (CRD's) to experience a seamless execution of CI/CD pipelines. This happens by leveraging the power of Kubernetes in terms of scalability, infrastructure abstraction and velocity.

The new main approach of Jenkins X is about to work on what we call a serverless experience, because there is no more traditional Jenkins engine running. So, it relies on a CI/CD serverles pipeline engine that can run on any standard Kubernetes deployment. This engine is the [Tekton CD project ](https://github.com/tektoncd/pipeline), a Google project that is part of the [Continuous Delivery Foundation](https://cd.foundation/).

In this post, we are showing the power of Tekton as a decoupled CI/CD pipeline engine execution. But more important, why an orchestration pipeline platform - like Jenkins X - is practically needed to design, configure and run your pipelines for the whole Delivery process.

## The Tekton base

Why then Tekton is a cool CI/CD engine?

First of all, Tekton is built on and for Kubernetes. This means that containers are the building blocks of any pipeline definition and execution, and Kubernetes orchestrates the container's magic. One step, one container. But it's more than that:
- Everything is decoupled, so for example, a group of steps, or any pipeline resource can be shared and reused through different pipeline executions
- Kubernetes is the platform, meaning that pipelines can be deployed and executed on any standard Kubernetes cluster.
- Sequential execution of tasks defines a pipeline. So creating a pipeline conceptually is as easy as defining the order of the tasks that we want to run and that may be already deployed in our Kubernetes cluster.
- Any task can be run by instantiating it from a `TaskRun` with desired parameter values. Because every previous task can be parametrized, reusing them is just a matter of calling the right task with a specific parameter. Again, decoupling and reusing.
- Pipelines usually consume different resources like code, containers, files, etc. So Tekton uses `PipelineResources` as inputs and outputs between tasks to execute the pipeline workflow. That means that pipeline resources can be shared between pipelines in a descriptive way.
- Every pipeline component is a CRD, so pipeline execution is a matter of containers orchestration, something that Kubernetes does really well and pretty fast. It is reliable, stable, scalable and performant

Let's try to understand Tekton pipelines decoupled architecture in the following diagram :
![Tekton Pipeline Architecture](../../static/img/tekton-jx-orchestration/TektonPipelineArch.png)

In terms of scalability, it's a nice way to isolate objects that can be reused easily, right?

## Decoupling CI/CD is good, but...

Tekton then is about the power of a decoupled CI/CD engine, which has many advantages. But no one said that defining decoupled CI/CD pipelines was an easy task. It can be complex from a conceptual point of view if you don't change the mindset. Traditionally, we've been defining pipelines as different stages with their current steps to be executed. So, everything is only about how to orchestrate stages as sequential or parallel tasks, configuring parameters or conditions into the pipeline. Let's say that CI/CD has a monolithic mindset to configure and define pipelines.

If we start designing reusable components, then the resources, the pipeline flow, the execution parametrization and then feeding all components back and forth... This can be messy and error prone till having the complete decoupled pipeline map configured.

## An orchestration engine to manage pipelines

Let's then think about three decoupling best practices for pipelines in containerized ecosystems:
- The CI/CD execution  must be flexible, reusable and decoupled.
- The pipeline orchestration must be manageable, understandable and easy to deploy.
- Resources configuration for pipeline components needs to be standardized and easy to manage.

CI/CD in not only about designing automation pipelines to deliver software. It is also about managing and providing all resources required for automation execution. In Kubernetes we will need to deal also with different objects like `Secrets`, `ServiceAccounts`, `PersistentVolumes`, `ConfigMaps`, `Ingress`, etc.

So, focusing only on CI/CD pipelines, resources management shouldn't be complicated as well as easy to configure.


## An example to understand Tekton and Jenkins X orchestration

We are going to use an example of building an application to see how to design and run CI/CD pipelines with Tekton. Then, we are taking the same application and see how Jenkins X pipelines, using Tekton, generate similar objects, but from a different focus. It will show how pipeline orchestration capabilities are applied to a decoupled CI/CD pipeline engine, no matter how I need to deal with platform or infrastructure resources.

To do that let's use a well known Spring Boot application like the [Spring Petclinic example](https://projects.spring.io/spring-petclinic/). And let's use a repo using traditional Jenkins pipeline to build the application, create a Docker container and deploy into Kubernetes cluster.

In my traditional Jenkins example I use two pipelines in fact that are automated using [CloudBees Core Cross Team Collaboration](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/) features:
- [Repo with a Jenkins pipeline to build](https://github.com/dcanadillas/petclinic-kaniko) the app, the Docker container and push it into Docker Registry
- A simple [Jenkins pipeline repo to deploy](https://github.com/dcanadillas/petclinic-kaniko-deploy) the previous container published

In our case, we are using one pipeline to do the build and deploy. First with a "pure Tekton" pipeline definition, and later using Jenkins X serverles pipelines.

### The pure Tekton way

So let's try to configure and execute the pipeline from a pure Tekton pipeline point of view. That means:
- Creating [`PipelineResources`](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md) that are going to be used by `Tasks`
- Defining and creating [`Tasks`](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md) that contains the steps to be executed in the `Pipeline`
- Defining and creating the [`Pipeline`](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md) that orchestrates the execution of `Tasks` with `Resources`
- Creating the [`PipelineRun`](https://github.com/tektoncd/pipeline/blob/master/docs/pipelineruns.md)
- Installing required Kubernetes resources, like `Secrets`, `ServiceAccounts` or permissions, in order to execute required steps within right containers (e.g. secrets used by Kaniko builder)

I already created a [GitHub Repo](https://github.com/dcanadillas/petclinic-tekton) with all YAML files needed (except secrets, which are explained in the MD file). But let's go through them.

We can create a YAML file with all `Tasks` objects and the `Pipeline` definition. We can call this file `petclinic-pipeline.yaml`:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-maven
spec:
  inputs:
    resources:
      - name: workspace
        type: git
    params:
      - name: workingDir
        description: Working directory parameter
        default: /workspace/workspace
  outputs:
    resources:
      - name: workspace
        type: git
  steps:
    - name: maven-build
      image: gcr.io/cloud-builders/mvn:3.5.0-jdk-8
      workingDir: ${inputs.params.workingDir}
      command: ["mvn"]
      args:
        - "clean"
        - "install"
    - name: ls-target
      image: ubuntu
      command:
        - "ls"
      args:
        - "-la"
        - "/workspace/workspace/target"
---
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-kaniko
spec:
  inputs:
    resources:
      - name: workspace
        type: git
        targetPath: petclinic
    params:
      - name: workingDir
        description: Working directory parameter
        default: /workspace/petclinic
      - name: DockerFilePath
        decription: Path to DockerFile
        default: /workspace/petclinic/Dockerfile
  outputs:
    resources:
      - name: dockerImage
        type: image
  steps:
    - name: kaniko-build
      image: gcr.io/kaniko-project/executor:latest
      command:
        - /kaniko/executor
      args:
        - "--dockerfile=${inputs.params.DockerFilePath}"
        - "--context=${inputs.params.workingDir}"
        - "--destination=${outputs.resources.dockerImage.url}"
      env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /secret/kaniko-secret.json
      volumeMounts:
        - name: kaniko-secret
          mountPath: /secret
  volumes:
    - name: kaniko-secret
      secret:
        secretName: emea-sa-secret
---
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-kubectl
spec:
  inputs:
    resources:
      - name: workspacedeploy
        type: git
    params:
      - name: workingDir
        description: Working directory parameter
        default: /workspace/workspacedeploy
      - name: deployFile
        description: Deployment file for app
        default: test-deploy.yaml
        # Just a parameter to check the depployment name, because one step will force redeploy
      - name: deploymentName
        description: The K8s deployment object name
        default: petclinic
  steps:
    - name: kubectl-clean
      image: gcr.io/cloud-builders/kubectl:latest
      workingDir: ${inputs.params.workingDir}
      command: ["/bin/bash"]
      args:
        - -c
        - MYDEPLOY=$(kubectl get deployments -l app=petclinic -o name | awk -F'/' '{print $2}');
        - if [ "$MYDEPLOY" = "${inputs.params.deploymentName}" ];
        - then kubectl delete deployment $MYDEPLOY;
        - fi
    - name: kubectl-deploy
      image: gcr.io/cloud-builders/kubectl:latest
      workingDir: ${inputs.params.workingDir}
      command: ["kubectl"]
      args:
        - apply
        - -f 
        - ${inputs.params.deployFile}
---
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: petclinic-pipeline
spec:
  resources:
    - name: source-repo
      type: git
    - name: docker-container
      type: image
    - name: deploy-repo
      type: git
  # params:
  #   - name: pathToContext
  #     description: Path of context build
  #     default: src
  tasks:
    - name: petclinic-maven
      taskRef:
        name: build-maven
      # params:
      #   - name: pathToContext
      #     value: "${params.pathToContext}"
      resources:
        inputs:
          - name: workspace
            resource: source-repo
        outputs:
          - name: workspace
            resource: source-repo
    - name: petclinic-kaniko
      taskRef:
        name: build-kaniko
      # params:
      #   - name: pathToContext
      #     value: "${params.pathToContext}"
      resources:
        inputs:
          - name: workspace
            resource: source-repo
            from: 
              - petclinic-maven
        outputs:
          - name: dockerImage
            resource: docker-container
    - name: petclinic-deploy
      taskRef:
        name: deploy-kubectl
      runAfter:
        - petclinic-kaniko
      resources:
        inputs:
          - name: workspacedeploy
            resource: deploy-repo
      params:
        - name: deployFile
          value: test-deploy-secret.yaml
```

Then we can define also a file `pipeline-resources.yaml` with all required `PipelineResources` objects needed:
```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-git
spec:
  type: git
  params:
    - name: url
      value: https://github.com/dcanadillas/petclinic-kaniko.git
    # - name: revision
    #   value: master
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-image
spec:
  type: image
  params:
    - name: url
      value: eu.gcr.io/emea-sa-demo/petclinic-kaniko:latest
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-deploy
spec:
  type: git
  params:
    - name: url
      value: https://github.com/dcanadillas/petclinic-kaniko-deploy.git
```

We are using two GitHub repositories as inputs (one for the application and the other for deployment definitions and one `PipelineResource` for the container image to be built).

Next, let's define the `PipelineRun`, that executes and instantiate the pipeline in a file `petclinic-run.yaml`:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: petclinic-pipelinerun
spec:
  pipelineRef:
    name: petclinic-pipeline
  serviceAccount: 'default'
  resources:
  - name: source-repo
    resourceRef:
      name: petclinic-git
  - name: docker-container
    resourceRef:
      name: petclinic-image
  - name: deploy-repo
    resourceRef:
      name: petclinic-deploy
```

In the `PipelineRun` object is important to understand that is intended in the way of defining the specific running instance of a deployed pipeline. So it's a perfect time to assign running input/output parameter values, specify `serviceAccounts` to be used, or referencing the right `PipelieResources` to be used.

Once we have the YAML definitions, it's only a matter of Kubernetes deployment of the Tekton CRD objects. So, to deploy them we can use `kubectl` command line tool:

```bash
$ kubectl apply -f petclinic-resources.yaml,petclinic-pipeline.yaml
```
``` 
pipelineresource.tekton.dev/petclinic-git created
pipelineresource.tekton.dev/petclinic-image created
pipelineresource.tekton.dev/petclinic-deploy created
task.tekton.dev/build-maven created
task.tekton.dev/build-kaniko created
task.tekton.dev/deploy-kubectl created
pipeline.tekton.dev/petclinic-pipeline created
```

We can see then the objects already deployed, as `CRDs` objects.

```bash
$ kubectl get tasks,pipelines,pipelineresources
```
```
NAME                             AGE
task.tekton.dev/build-kaniko     2s
task.tekton.dev/build-maven      2s
task.tekton.dev/deploy-kubectl   2s

NAME                                     AGE
pipeline.tekton.dev/petclinic-pipeline   2s

NAME                                           AGE
pipelineresource.tekton.dev/petclinic-deploy   2s
pipelineresource.tekton.dev/petclinic-git      2s
pipelineresource.tekton.dev/petclinic-image    2s
```

So we can check our pipeline to be executed:
```bash
$ kubectl describe pipeline.tekton.dev/petclinic-pipeline
```

```yaml
Name:         petclinic-pipeline
Namespace:    tekton-pipelines
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"tekton.dev/v1alpha1","kind":"Pipeline","metadata":{"annotations":{},"name":"petclinic-pipeline","namespace":"tekton-pipelin...}}
API Version:  tekton.dev/v1alpha1
Kind:         Pipeline
Metadata:
  Creation Timestamp:  2019-07-13T10:19:12Z
  Generation:          1
  Resource Version:    22835394
  Self Link:           /apis/tekton.dev/v1alpha1/namespaces/tekton-pipelines/pipelines/petclinic-pipeline
  UID:                 a8d4e6ca-a557-11e9-b2c9-42010a8400ab
Spec:
  Resources:
    Name:  source-repo
    Type:  git
    Name:  docker-container
    Type:  image
    Name:  deploy-repo
    Type:  git
  Tasks:
    Name:  petclinic-maven
    Resources:
      Inputs:
        Name:      workspace
        Resource:  source-repo
      Outputs:
        Name:      workspace
        Resource:  source-repo
    Task Ref:
      Name:  build-maven
    Name:    petclinic-kaniko
    Resources:
      Inputs:
        From:
          petclinic-maven
        Name:      workspace
        Resource:  source-repo
      Outputs:
        Name:      dockerImage
        Resource:  docker-container
    Task Ref:
      Name:  build-kaniko
    Name:    petclinic-deploy
    Params:
      Name:   deployFile
      Value:  test-deploy-secret.yaml
    Resources:
      Inputs:
        Name:      workspacedeploy
        Resource:  deploy-repo
    Run After:
      petclinic-kaniko
    Task Ref:
      Name:  deploy-kubectl
Events:      <none>
```

So we can see that the `Pipeline` object is going to execute in order the tasks already existing in K8s cluster: `build-maven`, `build-kaniko` and `deploy-kubectl`. And for that we are setting as inputs and outputs the different `PipelineResources` 

But, we are using some resources expected in the cluster that are not created at pipeline definition, like `Secrets` for pushing into private Docker Registry (GCR in my case), `ServiceAccount`, `Roles` and `RoleBindings` to deploy in Kubernetes with the specific permissions. I am not focusing in doing this in this post, but you can read how to do it in my [original repo documentation](https://github.com/dcanadillas/petclinic-tekton/blob/master/README.md#configuration-requirements).

Now, running the pipeline is just about deploying the `PipelineRun` definition:

```bash
$ kubectl apply -f petclinic-run.yaml
```
```
pipelinerun.tekton.dev/petclinic-pipelinerun created
```

```bash
kubectl get pods -w
```
```
NAME                                                     READY   STATUS     RESTARTS   AGE
petclinic-pipelinerun-petclinic-maven-9gnpf-pod-fa757f   0/5     Init:0/3   0          15s
tekton-pipelines-controller-6b565f9859-knqsb             1/1     Running    0          8h
tekton-pipelines-webhook-7f47c995cd-db2rv                1/1     Running    0          8h
petclinic-pipelinerun-petclinic-maven-9gnpf-pod-fa757f   0/5     Init:1/3   0          18s
petclinic-pipelinerun-petclinic-maven-9gnpf-pod-fa757f   0/5     Init:2/3   0          20s
petclinic-pipelinerun-petclinic-maven-9gnpf-pod-fa757f   0/5     PodInitializing   0          22s
```

```bash
$ kubectl get pods
```
```
NAME                                                      READY   STATUS      RESTARTS   AGE
petclinic-6f668b59b5-w8zrr                                1/1     Running     0          21s
petclinic-pipelinerun-petclinic-deploy-54kws-pod-c5a511   0/3     Completed   0          1m
petclinic-pipelinerun-petclinic-kaniko-s4nnw-pod-67721d   0/4     Completed   0          1m
petclinic-pipelinerun-petclinic-maven-9gnpf-pod-fa757f    0/5     Completed   0          4m
tekton-pipelines-controller-6b565f9859-knqsb              1/1     Running     0          8h
tekton-pipelines-webhook-7f47c995cd-db2rv                 1/1     Running     0          8h
```


### Playing with Jenkins X Serverless Pipelines

To understand what we mean about Tekton orchestration, let's try to simulate the same specific tasks and steps defining a Jenkins X serverless pipeline.

NOTE: This is not yet the standard way to work with Jenkins X, but using a standalone `jenkins-x.yml` pipeline, we can understand the relation of Tekton objects and how Jenkins X creates and orchestrates them.

#### Deploy Jenkins X


#### Standalone Jenkins X pipeline



### The pure Jenkins X way



## Some thoughts and conclusions


