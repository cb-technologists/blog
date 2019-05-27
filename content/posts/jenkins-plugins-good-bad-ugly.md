---
author:
  name: "Kurt Madel"
title: "Jenkins Plugins: The Good, the Bad and the Ugly"
date: 2019-05-25T10:29:46-04:00
showDate: true
photo: "/img/jenkins-plugins-good-bad-ugly/wythe-alley-electric-pole.jpg"
photoCaption: "Photograph by Kurt Madel ©2019"
exif: "SONY RX-100 ISO 125 32.17mm ƒ/5.6 1/400"
draft: true
tags: ["jenkins","plugins","containers"]
---
There are over 1400 Jenkins plugins and that is both a blessing and a curse. Of those 1400 plugins only a small percentage are well maintained and tested, and even fewer (140 of 1400+) are part of the [CloudBees Assurance Program (CAP)](https://go.cloudbees.com/docs/cloudbees-documentation/assurance-program/) as verified and/or compatible plugins - well tested to interoperate with the rest of the CAP plugins (and their dependencies) and with a specific LTS version of Jenkins. Problems can arise when you use plugins that aren't part of CAP, or a plugin that isn't well maintained or tested to work with all of the other plugins you are using and the specific version of Jenkins that you are using. But the extensiblity offered by plugins has helped make Jenkins the most popular CI tool on the planet.

I typically like to end posts on a good note, so I will start with *The Ugly* and end with *The Good* - and then offer some opinionated ideas/best practices on Jenkins plugin management and usage.

# The Ugly
There are hundreds of Jenkins plugins that have security vuneralabilites. Over 55 plugins were listed as part of the [2019-04-03 Jenkins Security Advisory](https://jenkins.io/security/advisory/2019-04-03/). Even worse is when you find out that a plugin that you are using has a security vunerability and you also find out that the plugin is not maintained anymore. You could search the 1400+ Jenkins plugins to see if there is another plugin that is maintained and that does what you need, or you could become a plugin maintainer - not exactly what you intended to sign up for when you first started using Jenkins. Are you developing your own applications or are you looking to become a Jenkins plugin developer?

Another ugly issue arises when you have numerous Jenkins masters in your organization. These Jenkins instances are often snowflakes comprised of many different plug-ins. So managing more than one Jenkins master with disparate set of plugins can become very ugly, very quickly. CloudBees can certainly help you with this through CAP and something we call Team Masters - easily provisioned and managed team specific  Jenkins masters. However, there is nothing stopping individual Jenkins master admins from manually installing a plugin.

# The Bad
Installing a lot of plugins can result in maintenance hell and sometimes your Jenkins master doesn't even restart successfully after upgrading a plugin. 
{{< image src="/img/jenkins-plugins-good-bad-ugly/jenkins_devil.png" alt="Jenkins Devil" >}}
And although the Jenkins Devil makes for a very cool sticker, it isn't something you ever want to see on your Jenkins Master, especially after restarting Jenkins for a plugin update. Backing out a plugin update that causes Jenkins to crash is not a fun thing to deal with.

Dependency hell is anothet bad thing that Jenkins admins have to deal with all the time. Sometimes upgrading just one plugin results in the need to update dozens others and many Jenkins admins do this directly in on their production Jenkins master. Blue Ocean, while a noble attempt at a new UI for Jenkins Pipelines, requires dozens of dependnecies, many of which you probably have no use for - for example the Blue Ocean plugin suite requires both the Bitbucket Pipeline for Blue Ocean and the GitHub Pipeline for Blue Ocean plugins even you don't use either Bitbucket or GitHub for source control.

Too many plugins that do the same thing - how do you choose? Search the Jenkins plugin site for *Docker* and you get 26 results. If I want Docker based agents should I use the **Docker plugin** or **Yet Another Docker plugin**? With 1400+ plugins, sometimes it can be hard to choose the right one.

# The Good
The extensibility and integrations provided by Jenkins plugins is amazing. I don't believe that there is any other CI platform that integrates as many source control systems as Jenkins. Without Jenkins extensive plugin ecosystem it would not be the CI automation tool of choice that it has become.

Pipeline, JCasC, source control. 

# So What to Do
CloudBees can certainly help. All of the CloudBees distributions, including the free CloudBees Jenkins Distribution, include CAP with Beekeeper. I manage a few demo/workshop environments for the Solution Architecture team and update those environments every month. I have yet to have an update that has resulted in the Devil Jenkins.

There are a few things you can do right now to make using Jenkins Plugins better.

## Test
Always test any new plugin or plugin update before you put it into your production Jenkins master(s). Running Jenkins as a container can certainly make this easier - and is what I suggest - but there is not reason why you can't use Jenkins to automate this kind of testing regardless of how you deploy Jenkins.

The Jenkins X ephemeral masters basically went with this approach - extensive testing whenever a new plugin was added to the the CasC Master container image.

## Use Fewer Plugins
And finally, use fewer plugins - reducing the need for all the above. Migrating as many Jenkins Pipeline steps from plugins to `sh` steps running in containers not only reduces the bad and ugly above, it also makes it easier to tesat and reduce dependencies on plugin maintainers, and provides better portability to other emerging CD technologies - like Jenkins X Pipelines with Tekton.

Do you really need the Docker plugin and the Yet Another Docker plugin? Or the Chuck Noris plugin? The fewer plugins that you install, the fewer plugins you have to manage and the less chance that they will have security issues or even worse, bring your Jenkins master down - Jenkins Devil and all.

Use Jenkins Pipelines with a Jenkinsfile in source control with Pipeline Shared Libraries.

## Manage Plugins with CasC
Never use the Jenkins UI to install plugins.
Maintain your plugins as code in source control, where every new plugin and plugin upgrade can be tracked as commits. The easiest and best way to do this, in my opinion, is to use a customized Docker image for your plugins - in addition to other configuration.

If you have read any of my other posts you will know that I am a big fan of containers - and have always run Jenkins with containers since I started at CloudBees back in 2015. The Jenkins GitHub Org docker project introduced the idea of using a simple `plugins.txt` file to build the plugins you want to use right into your Jenkins master container image.