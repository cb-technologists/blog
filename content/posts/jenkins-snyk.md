---
author:
  name: "Casey Vega"
title: "Safety First with Jenkins and Snyk"
date: 2019-07-31T07:00:00-00:00
showDate: true
draft: false
tags: ["cloudbees","jenkins","snyk","security","vulnerability","infosec", "devsecops"]
---

## Implementing security scanning as a preventative measure in your CI pipeline.

![Jenkins + Snyk = Love Security](/img/jenkins-snyk/jenkins-snyk.png)

### Overview
Today’s developers are being empowered to expeditiously innovate, creating new software capabilities and building continuous customer value. At CloudBees, we commonly tell our customers:

> "Every business is a software business and is under pressure to innovate constantly. This increased velocity introduces new business risks. CloudBees is building the world's first end-to-end automated software delivery system (SDM), enabling companies to balance governance and developer freedom."  

Many of the products getting built today contain or rely on sensitive consumer data and tend to have wide-spread privacy implications should the consumer's data get leaked. Unfortunately, in today's world, we're seeing more and more companies fall victim to security vulnerabilities, and the exposure of sensitive data is also happening with a semi-regular cadence. 

The Equifax Breach, which resulted in an up to $700 million settlement last week, could have likely been avoidable had a mechanism existed that made mitigating security issues reliable and straightforward for developers. In this case, a lapse in process and legacy code get attributed with the blame.  

Recent supply chain and [typo squatting](https://www.techrepublic.com/article/malicious-libraries-in-package-repositories-reveal-a-fundamental-security-flaw/) attacks against popular open-source libraries and repositories are also becoming a more significant attack vector. A simple typo or desire to stay current with the latest stable release can have a significant impact on business security and customer privacy.

### Enter Snyk

![snyk](/img/jenkins-snyk/snyk-logo.png)


For those of us that are not security experts, the ability to arm oneself with relevant, actionable data can be the difference between an all hands on deck security catastrophe and a bland Tuesday at the office. In 2018 an average of 45 new CVE entries were created daily for a total of 16555 entries. That's quite a bit of information to sift through, especially when you're a developer. Snyk provides multiple mechanisms that not only inform developers but can also provide valuable feedback before a vulnerability gets committed to a code repository.

Snyk is a SaaS offering that manages it's own vulnerability database and provides continuous monitoring of an application's dependencies and their respective docker containers. Snyk can also provide additional security metrics coupled with historical project reporting and has features for managing OSS license policies and compliance reporting as a paid feature. NOTE: They also offer on-prem licensing for enterprises. 

The Snyk offering focuses on 5 specific actions when encountering a vulnerability. These actions include monitoring, prevention, finding, fixing, and alerting.

I chose to explore Snyk because it offers a free, open-source tier available to the public and also because it provides different ways of interacting with the service depending on your use case. Snyk has support for quite a few languages (Java, Ruby, Node, Python, Scala, Golang, .NET, and PHP) and integrates with other popular services like Github, Docker Hub, Slack, Jira and of course Jenkins.

### Jenkins and CloudBees CI/CD
![cloudbees](/img/jenkins-snyk/cloudbees-logo.png)

Jenkins is fairly ubiquitous when it comes to continuous integration and continuous delivery. It’s easily one of the most battle-tested, configurable, and programmable automation tools being used by developers and enterprises alike.

For those not as familiar, Welcome, Jenkins is an open-source project that offers smaller teams and individuals the ability to create CI/CD pipelines plus more. The CloudBees offering focuses on economies of scale, security, administration, and support for medium-sized businesses to large scale enterprises. Jenkins is a component of the CloudBees offering. As good stewards do, CloudBees donates about 80% of the code it writes back to the Jenkins open-source project. 

Many of the customers I speak to have a desire for developer self-service capabilities when it comes to CI/CD. While enterprises want to enforce specific policies, especially around security, they also want to promote developer autonomy and creativity.

Security teams typically want to draw a line in the sand when it comes to critical and high severity security issues. Who could blame them? Using Jenkins and Snyk together, vulnerability scanning becomes a first-class citizen of the CI/CD pipeline. Developers and Security teams can invoke failure for a build based on a predefined policy, preventing security incidents early in the development cycle. Developers are also empowered with data and the `snyk wizard` capability to fix reported issues.

On the CloudBees side, we take this one step further. Using custom marker files, we allow CloudBees administrators to control immutable events that happen in the pre and post-build phase of the pipeline outside the scope of a Jenkinsfile. Custom marker files allow security operations to impose vulnerability scanning at the top-level versus a team or developer having to add it to a pipeline for each project. When compliance and auditing are essential to the customer, the custom marker file plays a pivotal role. If you're interested in learning more about custom marker files, you can read about them [here](https://go.cloudbees.com/docs/plugins/workflow/#pipeline-custom-factories').

### Getting Started

* Sign up for a free account at https://app.snyk.io/signup
	* Take note of your Snyk API token here: https://app.snyk.io/account
* You must be running an instance of Jenkins or CloudBees Core
	* You must have the ability to configure Jenkins and install plugins if you want to use Snyk’s Jenkins plugin.
	* You must have a docker daemon present on the agent including your language runtime if you’re using the plugin
* You must have commit access to a git repository. (We use Github)
* Prior Jenkins knowledge will be required. You should also be familiar with installing dependencies for your project and language.

Snyk provides a Jenkins plugin for developers looking to get started quickly. This comes in the form of a freestyle build step, and a pipeline function for developers using Jenkinsfile.

Snyk also provides a container (snyk-cli) that provides users the CLI interface for container based builds. Containers are tagged for corresponding languages and runtimes (e.g. `snyk/snyk-cli:python-3`).

#### 1. **Install the Snyk plugin, optional**

The first part of this tutorial highlights the functionality of the Snyk plugin. The second part will focus on using the snyk-cli with container technology (Docker, Kubernetes) and does not require a plugin install.

[Snyk Jenkins Plugin Page](https://plugins.jenkins.io/snyk-security-scanner)


![plugin install](/img/jenkins-snyk/install-plugin.png)

If you’re using a CloudBees plugin catalog you can update your catalog schema with the following plugin details:

```JSON
"includePlugins" : {
  "snyk-security-scanner": {
    "version" : "2.10.0"
  }
}
```

The plugin has one additional requirement. You must configure Snyk with a global configuration (Manage Jenkins -> Global Tool Configuration)

![Global Tool Configuration](/img/jenkins-snyk/global-tool.png)

#### 2. **Add your Snyk token to Jenkins**

Get your Snyk token from https://app.snyk.io/account

Add a global credential in Jenkins. Under the kind option make sure you choose Snyk API Token. (Credentials -> Global -> Add Credentials)	

> If you do not intend on using the Snyk plugin you'll still be required to add a credential. Use the secret text credential type instead and add the Snyk token as the secret.

#### 3. **Freestyle Jobs (with plugin)** 

Snyk requires two steps in order to scan successfully. Step one, installing dependencies. To do this use an Execute Shell step on a new or existing freestyle job.

![build step shell](/img/jenkins-snyk/build-step-shell.png)

In the command box you’ll need to provide the command to install your dependencies. We’re using python and the Snyk supported requirements.txt file to pip install.  

```
$ pip install -r requirements.txt
```

Now we can add a second build step, Invoke Snyk Security task

![build step snyk](/img/jenkins-snyk/build-step-snyk.png)

After choosing Synk Security Task, several options will need to be configured.

![configure snyk](/img/jenkins-snyk/configure-build-step-snyk.png)


**When issues are found**

* Fail on Build if severity is high, medium or low or continue regardless of vulnerability severity (We’ve chosen high).


**Monitor project on build**

* Continue monitoring project will scan the repo once a day and provide notifications outside of Jenkins.

>This is a good feature if you’re not always building your project or code that isn’t changed often. Developers can continue to get access to vulnerability notifications regardless of mean lead time or deploy frequency.


**Snyk details**

* Snyk API token
* Target file (We’re using a requirements.txt file and assuming python)
* Organization
* Name of the project you’re scanning (We’re using git repo name)


**Advanced**

* Snyk Install (Added in the global configuration in Step 1)
* Snyk CLI arguments
	* [Snyk CLI cheat sheet](https://snyk.io/blog/snyk-cli-cheat-sheet/)

    
Once the project is configured it’s time to build. Trigger or start a build 

**PASS:**
```
Testing for known issues...
> /home/jenkins/tools/io.snyk.jenkins.tools.SnykInstallation/snyk-latest/snyk-alpine test --json --severity-threshold=high --file=requirements.txt --org=cloudbees --project-name=project-python
Result: 0 known issues | No high severity vulnerabilities
```

**FAIL:**
```
Testing for known issues...
> /home/jenkins/tools/io.snyk.jenkins.tools.SnykInstallation/snyk-latest/snyk-alpine test --json --severity-threshold=high --file=requirements.txt --org=cloudbees --project-name=project-python
Result: 2 known issues | 2 high severity vulnerable dependency paths
```

Snyk will also provide a link to the security report for each build run:

![snyk build report](/img/jenkins-snyk/build-report.png)

Looking forward, you can add multiple build steps to freestyle jobs if you want to for example scan your Dockerfile and your Python dependencies in a single job.

At CloudBees we typically recommend users start with pipeline jobs if they have the opportunity. While freestyle jobs work well, pipelines jobs can provide several distinct advantages (e.g. parallel steps, shared libraries).

#### 4. **Pipeline Jobs (with plugin)**

The Snyk plugin provides a pipeline function for scanning. Per the official documentation:

The snykSecurity function accepts the following parameters:

* **snykInstallation** - Snyk installation configured in the Global Tool Configuration.
* **snykTokenId** - The ID for the API token from the Credentials plugin to be used to authenticate to Snyk.
* **additionalArguments** (optional, default none) - Refer to the Snyk CLI help page for information on additional arguments.
* **failOnIssues** (optional, default true) - This specifies if builds should be failed or continued based on issues found by Snyk.
* **organisation** (optional, default none) - The Snyk organisation in which this project should be tested and monitored.
* **projectName** (optional, default none) - A custom name for the Snyk project created for this Jenkins project on every build.
* **severity** (optional, default low)- Only report vulnerabilities of provided level or higher (low/medium/high).
* **targetFile** (optional, default none) - The path to the manifest file to be used by Snyk.

An example Jenkinsfile using the Snyk Jenkins plugin: 

```groovy
pipeline {
  agent any
  stages {
    stage('snyk dependency scan') {
      tools {
        snyk 'snyk-latest'
      }	
      steps {
        snykSecurity(
          organisation: 'cloudbees',
          severity: 'high',
          snykInstallation: 'snyk-latest',
          snykTokenId: 'snyk',
          targetFile: 'requirements.txt',
          failOnIssues: 'true'
        )		
      }
    }
  }
}
```

#### 4. **Pipeline Jobs (without plugins)**

> To get started without a plugin you need to have a Snyk token stored as a secret text credential outlined in **Step 2**.

Using Snyk in your pipeline is also possible without the using Snyk plugin for Jenkins. In fact, I would say depending on the user or company, this is typically what a CloudBees Solutions Architect or Professional Services Consultant would recommend when setting up your Pipeline, Multi-Branch or Github Organization job. If you're curious or asked why, here are a couple of reasons:

1. Not every user has admin privileges. A plugin install and global tool configuration require escalated privileges. This may not be an issue for some, in enterprise settings this isn't always a simple hurdle to overcome.
2. Closer to what a developer would do if they wanted to run Snyk manually or locally
3. Configuration as Code and GitOps. The benefits are numerous. Empowered developers, and reduced Jenkins administration overhead are just two.  

Using the snyk-cli tool, developers can craft steps using the command line interface. For Jenkins and CloudBees users, this means you'll need to invoke `sh` within your stage and then pass the CLI parameters required to satisfy a successful snyk.

> If you want to install the snyk-cli locally and have a working NodeJS environment you can run the following command:

```bash
$ npm install -g snyk
```

**DOCKER:**

```groovy
pipeline {
  agent none
  stages {
    stage('snyk dependency scan') {
      agent {
        docker {
          image 'snyk/snyk-cli:python-3'
        }
      }
      environment {
        SNYK_TOKEN = credentials('snyk-token')
      }	
      steps {
        sh """
          pip install -r requirements.txt
          snyk auth ${SNYK_TOKEN}
          snyk test --json \
            --severity-threshold=high \
            --file=requirements.txt \
            --org=cloudbees \
            --project-name=project-python
        """		
      }
    }
  }
}
```

**KUBERNETES**

In the example below we're using some of the additional docker functionality provided by Snyk. We've also added the required docker integration to our pod. 

> Docker in docker is [probably](http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/) not the way you want to build your production containers, scanning however should not be an issue. Projects such as [Kaniko](https://github.com/GoogleContainerTools/kaniko) provide a path for developers that want to use Kubernetes for building docker containers with Dockerfile. See Matt Elgin's [post](https://cb-technologists.github.io/posts/cjd-casc/) for some good examples on using Kaniko. 

The pipeline is scanning both the container and the python dependencies in parallel with the `failFast` option set to `true`. This enables the build to run quickly, if failure is detected in either stage, the build is short circuited and both stages are stopped.


```groovy
pipeline {
  agent {
    kubernetes {
      label 'python-dev'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: snyk-python
    image: snyk/snyk-cli:python-3
    command:
    - /bin/cat
    tty: true
  - name: snyk-docker
    image: snyk/snyk-cli:docker
    command:
    - /bin/cat
    tty: tru
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2375
  - name: dind
    image: docker:18.09-dind
    securityContext:
      privileged: true
    volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
  volumes:
  - name: dind-storage
    emptyDir: {}
"""
    }
  }
  stages {
    stage('Snyk Scan') {
      failFast true
      environment {
        SNYK_TOKEN = credentials('snyk-token')
      }	
      parallel {
        stage('dependency scan') {
          steps {
            container('snyk-python') {
              sh """
                pip install -r requirements.txt
                snyk auth ${SNYK_TOKEN}
                snyk test --json \
                  --file=requirements.txt \
                  --severity-threshold=high \
                  --org=cvega \
                  --project-name=project-python
              """
            }
          }
        }
        stage('docker scan') {
          steps {
            container('snyk-docker') {
              sh """
                docker build -t python-project .
                snyk auth ${SNYK_TOKEN}
                snyk test --json \
                  --docker python-project:latest \
                  --file=Dockerfile \
                  --severity-threshold=high \
                  --org=cvega \
                  --project-name=project-python
              """
            }
          }
        }
      }
    }
  }
}
```

#### 5.  **Snyk Reports and Wizard**

Like Jenkins and CloudBees, Snyk has quite a few valuable features. The Snyk portal provides data about  vulnerabilities in your project.

![snyk portal](/img/jenkins-snyk/snyk-web.png)

The CLI wizard feature of Snyk enables developers to take action and update dependencies. Using multi-branch jobs this makes testing updates to dependencies manageable.

![snyk wizard](/img/jenkins-snyk/snyk-wizard.png)

#### 6. **Summary**

We've presented several different ways to integrate Snyk and Jenkins using different types of technology and all of it fits together well. This is exactly the type of testing that could have prevented recent data breaches. This is also the type of development self-service that actually empowers developers and security teams to take the proper mitigation steps.

**Last but not Least**

Join CloudBees and Snyk at DevOps World.

[![](/img/jenkins-snyk/dwjw.png "DevOps World - Jenkins World")](https://www.cloudbees.com/devops-world/)

