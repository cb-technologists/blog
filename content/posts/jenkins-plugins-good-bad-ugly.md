---
authors:
  - "Kurt Madel"
title: "Jenkins Plugins: The Good, the Bad and the Ugly"
date: 2019-05-30T05:50:46-04:00
showDate: true
photo: "/img/jenkins-plugins-good-bad-ugly/wythe-alley-electric-pole.jpg"
photoCaption: "Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 32.17mm ƒ/5.6 1/400"
draft: false
tags: ["jenkins","plugins","containers","CasC"]
---
There are over [1600 Jenkins plugins](http://updates.jenkins.io/pluginCount.txt) and that is both a blessing and a curse. Of those 1600 plugins only a small percentage are well maintained and tested, and even fewer (140 of 1600+) are part of the [CloudBees Assurance Program (CAP)](https://go.cloudbees.com/docs/cloudbees-documentation/assurance-program/) as verified and/or compatible plugins - well tested to interoperate with the rest of the CAP plugins (and their dependencies) and with a specific LTS version of Jenkins. Problems can arise when you use plugins that aren't part of CAP, or a plugin that isn't well maintained or tested to work with all of the other plugins you are using and the specific version of Jenkins that you are using. But the extensibility offered by plugins has helped make Jenkins the most popular CI tool on the planet.

I typically like to end posts on a good note, so I will start with *The Ugly* and end with *The Good* - and then offer some opinionated ideas/best practices on Jenkins plugin management and usage.

# The Ugly
There are almost always a number of Jenkins plugins that have security vulnerabilities. Over 55 plugins were listed as part of the [2019-04-03 Jenkins Security Advisory](https://jenkins.io/security/advisory/2019-04-03/). Even worse is when you find out that a plugin that you are using has a security vulnerability and you also find out that the plugin is not maintained anymore. You could search the 1600+ Jenkins plugins to see if there is another plugin that is maintained and that does what you need, or you could become a plugin maintainer - not exactly what you intended to sign up for when you first started using Jenkins. Are you developing your own applications or are you looking to become a Jenkins plugin developer?

Another *ugly* issue arises when you have numerous Jenkins masters in your organization. These Jenkins instances are often snowflakes comprised of many different plug-ins. So managing more than one Jenkins master with disparate sets of plugins can become very ugly, very quickly. CloudBees can certainly help you with this through CAP and something we call [Team Masters - easily provisioned and managed team specific Jenkins masters](https://go.cloudbees.com/docs/cloudbees-documentation/admin-cje/cje-ux/#_when_to_use_a_team_master_when_to_use_a_managed_master) with an opinionated set of very stable and tested plugins. However, there is nothing stopping individual Jenkins master admins from manually installing a plugin and sometimes ending up with an unusable Jenkins master.

# The Bad
Installing a lot of plugins can result in maintenance hell and sometimes your Jenkins master doesn't even restart successfully after upgrading a plugin. 

{{< image src="/img/jenkins-plugins-good-bad-ugly/jenkins_devil.png" alt="Jenkins Devil" >}}

And although the Jenkins Devil makes for a very cool sticker, it isn't something you ever want to see on **your** Jenkins Master, especially after restarting Jenkins for a plugin update. Backing out a plugin update that causes Jenkins to crash is not a fun thing to deal with and will slow down your software delivery.

Dependency hell is another *bad* thing that Jenkins admins have to deal with all the time. Sometimes upgrading just one plugin results in the need to update dozens others, and many Jenkins admins do this directly on their production Jenkins master. Blue Ocean, while a noble attempt at a new UI for Jenkins Pipelines, requires dozens of dependencies, many of which you probably have no use for - for example the Blue Ocean plugin suite requires both the *Bitbucket Pipeline for Blue Ocean* and the *GitHub Pipeline for Blue Ocean* plugins even if you don't use either Bitbucket or GitHub for source control.

Too many plugins that do the same thing - how do you choose? Search the Jenkins plugin site for *Docker* and you get 26 results. If I want Docker based agents should I use the **Docker plugin** or **Yet Another Docker plugin**? With 1600+ plugins, sometimes it can be hard to choose the right one.

# The Good
The extensibility and integrations provided by Jenkins plugins are amazing. I don't believe that there is any other CI platform that integrates with as many source control tools/platforms as Jenkins. Without Jenkins' extensive plugin ecosystem it would not be the CI automation tool of choice that it has become. Jenkins is by far the most flexible CI platform available, bar none, and the Jenkins plugin ecosystem is a big reason why.

There are a lot of very *good*, and even necessary, plugins. Like plugins for credentials and for source control - Jenkins has awesome integration with GitHub and Bitbucket for example. And the Jenkins Pipeline plugin suite (although another example of dependency hell) provides a [Declarative approach to building you CI/CD pipelines](https://jenkins.io/doc/book/pipeline/syntax/#declarative-pipeline) that can be [easily managed as-code in source control](https://jenkins.io/doc/book/pipeline/jenkinsfile/). And finally, the [JCasC plugin](https://jenkins.io/projects/jcasc/) makes it easier than ever to manager your Jenkins master configuration as-code in source control.

So there are some very *good* reasons to use **some** plugins.

# So What to Do
CloudBees can certainly help. All of the CloudBees distributions, including the [free CloudBees Jenkins Distribution](https://www.cloudbees.com/products/cloudbees-jenkins-distribution), include CAP with Beekeeper. I have managed a few demo/workshop environments for the CloudBees Solution Architecture team for the last 4 years and update those environments almost every month. I have yet to have an update that has resulted in the Jenkins Devil - ok maybe one.

There are a few other things you can do right **now** , whether you use a CloudBees Distro or not, to make using Jenkins Plugins easier to manage and less impactful to your production Jenkins master - allowing you to focus on CD for the applications you are delivering instead of spending too much time managing Jenkins.

## Use Jenkins Pipeline
Although Jenkins Pipeline does require a [number of plugins and plugin dependencies](https://plugins.jenkins.io/workflow-aggregator) its advantages far outweigh the disadvantages of using Jenkins without Pipeline jobs. Using Jenkins Pipelines with a Jenkinsfile in source control and [Pipeline Shared Libraries](https://jenkins.io/doc/book/pipeline/shared-libraries/) can greatly reduce the number of additional plugins you need to install and manage. For example if you need to send a Slack message, just run a simple `curl` command in a lightweight container instead of installing the [Jenkins Slack plugin](https://github.com/jenkinsci/slack-plugin/issues):

```bash
curl -X POST -H 'Content-type: application/json' --data '{"text":"The build is broken :("}' YOUR_WEBHOOK_URL
```

This is actually considered a best practice for Jenkins Pipelines as any `step` that is run from a plugin will actually run on the Jenkins master, not on the agent (other than the `sh`, `bat` and `pwsh` steps). This will result in worse performance for your Jenkins master and may even bring your Jenkins master down - once again slowing down your application delivery.

Another big plus with replacing Jenkins Pipeline plugin based steps with lightweight shell scripts is that it provides easier testing and more portability of your CD pipelines to other platforms. For example, Jenkins X Pipelines with Tekton runs every pipeline step as a command in a container - adopting that approach with Jenkins Pipelines now will make it much easier to migrate to better emerging solutions in the future.

## Use Fewer Plugins
Using fewer plugs will reduce the amount of pain you will incur from many of the *ugly* and *bad* issues mentioned above. Migrating as many Jenkins Pipeline `steps` from plugins to `sh` steps running in containers not only reduces the *bad* and *ugly* above, it also makes it easier to test and reduce dependencies on the less than stellar plugin maintainers (like me), and provides better portability to other emerging CD technologies - like [Jenkins X Pipelines with Tekton](https://kurtmadel.com/posts/native-kubernetes-continuous-delivery/jenkins-x-goes-native/#re-tooling-with-tekton).

Do you really need the Docker plugin and the Yet Another Docker plugin? Or the Chuck Noris plugin? The fewer plugins that you install, the fewer plugins you have to manage and the less chance that they will have security issues or even worse, bring your Jenkins master down - Jenkins Devil and all.

## Test
Always test any new plugin or plugin update before you put it into your production Jenkins master(s). Running Jenkins as a container can certainly make this easier - and is what I suggest - but there is no reason why you can't use Jenkins to automate this kind of testing regardless of how you deploy Jenkins. Just spin up a Jenkins master with a few *fake* jobs that use the plugins in a similar way to how you use them in your *real* jobs. All of this can be automated with Jenkins itself.

The [Jenkins X ephemeral masters](https://github.com/jenkins-x/jenkins-x-serverless-filerunner) basically went with this approach - extensive testing whenever a new [plugin was added to the the CasC Master container image](https://github.com/jenkins-x/jenkins-x-serverless-filerunner/blob/master/pom.xml#L32).

## Manage Plugins with CasC
Never use the Jenkins UI to install plugins. Maintain your plugins as code in source control, where every new plugin and plugin upgrade can be tracked as commits. The easiest and best way to do this, in my opinion, is to use a customized Docker image that includes the plugins you **absolutely need** - in addition to other configuration via JCasC (and if necessary, [`init` scripts](https://wiki.jenkins.io/display/JENKINS/Post-initialization+script)). If you have read any of my other posts you will know that I am a big fan of containers - and have always run Jenkins with containers since I started at CloudBees back in 2015. The Jenkins GitHub Org *docker* project [provides a script](https://github.com/jenkinsci/docker/blob/master/install-plugins.sh) for [preinstalling plugins](https://github.com/jenkinsci/docker#preinstalling-plugins) from a simple `plugins.txt` file so your Jenkins master container image has all the plugins you need on startup. This makes it easier to test plugin changes and all of your plugin changes are captured as code commits - and a tool like Git (GitHub, BitBucket, even GitLab) is much better at tracking/auditing/controlling such changes than Jenkins was ever meant to be. Here is a simple `plugins.txt` file and `Dockerfile` to get you started:

*plugins.txt*
```txt
configuration-as-code:1.19
credentials:2.2.0
```

Yes, only two plugins. The reason why we only need these two plugins is because the [CloudBees Jenkins Distribution](https://www.cloudbees.com/blog/cloudbees-jenkins-distribution-adds-stability-and-security-your-jenkins-environment) already contains a curated set of plugins for Jenkins Pipeline, Blue Ocean, source control management and everything else we need - all well tested for us already.

This version of the Credentials plugin is an exception, because the recent version of the plugin with JCasC support has not been integrated into CAP yet (coming soon!).

*Extending the CloudBees Jenkins Distribution container image with plugins and JCasC*
```Dockerfile
FROM cloudbees/cloudbees-jenkins-distribution:2.164.3.2

# optional, but you might want to let everyone know who is responsible for their Jenkins ;)
LABEL maintainer "kmadel@cloudbees.com"

#set java opts variable to skip setup wizard; plugins will be installed via license activated script
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"
#skip setup wizard; per https://github.com/jenkinsci/docker/tree/master#preinstalling-plugins
RUN echo 2.0 > /usr/share/jenkins/ref/jenkins.install.UpgradeWizard.state

# diable cli
ENV JVM_OPTS -Djenkins.CLI.disabled=true -server
# set your timezone
ENV TZ="/usr/share/zoneinfo/America/New_York"

#config-as-code plugin configuration
COPY config-as-code.yml /usr/share/jenkins/config-as-code.yml
ENV CASC_JENKINS_CONFIG /usr/share/jenkins/config-as-code.yml

# use CloudBees' update center to ensure you don't allow any really bad plugins
ENV JENKINS_UC http://jenkins-updates.cloudbees.com

#install suggested and additional plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
COPY jenkins-support /usr/local/bin/jenkins-support
COPY install-plugins.sh /usr/local/bin/install-plugins.sh
RUN bash /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
```

# Use Plugins You Need and No More
So, don't avoid Jenkins plugins - they are an important part of what makes Jenkins great and add critical features to the way you will use Jenkins - but be smart about the plugins you use and keep your application delivery your primary focus - not your CI tool.
