---
author:
  name: "Kurt Madel"
title: "Jenkins Plugins: The Good, the Bad and the Ugly"
date: 2019-05-25T10:29:46-04:00
showDate: true
draft: true
tags: ["jenkins","plugins","containers"]
---
There are over 1400 Jenkins plugins and that is both a blessing and a curse. Of those 1400 plugins only a small percentage are well maintained and tested, and even fewer are part of the [CloudBees CAP program](https://go.cloudbees.com/docs/cloudbees-documentation/assurance-program/) - well tested to interoperate with the rest of the CAP plugins (and their dependencies) and with a specific LTS version of Jenkins. Problems can arise when you use plugins that aren't part of CAP, or a plugin that isn't well maintained or tested to work with all of the other plugins you are using and the specific version of Jenkins that you are using. But the extensiblity offered by plugins has helped make Jenkins the most popular CI tool on the planet.

I typically like to end posts on a good note, so I will start with The Ugly and end with The Good - along with some opinionated thoughts on Jenkins plugin management and usage.

# The Ugly
There are hundreds of Jenkins plugins that have security vuneralabilites. Over 55 plugins were listed as part of the [2019-04-03 Jenkins Security Advisory](https://jenkins.io/security/advisory/2019-04-03/).

# The Bad
Maintenance hell. Jenkins doesn't start after upgrading a plugin. 

{{< image src="/img/jenkins-plugins-good-bad-ugly/jenkins_devil.png" alt="Jenkins Devil" >}}
And even worse is when you see the devil Jenkins when restarting Jenkins after a plugin update. Backing out a plugin update that causes Jenkins to crash is not a fun thing.

Dependency hell. Sometimes upgrading just one plugin results in the need to update dozens others and many Jenkins admins do this directly in on their production Jenkins master.

# The Good
Pipeline, JCasC, source control. The extensibility and integrations provided by Jenkins plugins are amazing. THere is probably no other CI platform that integrates with as many version control systems as Jenkins.

# So What to Do
Config-as-Code is good and possible for Jenkins and Jenkins plugins.

Maintain your plugins as code in source control, where every new plugin and plugin upgrade can be tracked as commits. The easiest and best way to do this, in my opinion, is to use a customized Docker image for your plugins - in addition to other configuration.

Test plugin changes.

## Use Fewer Plugins

And finally, use fewer plugins - reducing the need for all the above. Migrating as many Jenkins Pipeline steps from plugins to `sh` steps running in containers not only reduces the bad and ugly above, it also makes it easier to tesat and reduce dependencies on plugin maintainers, and provides better portability to other emerging CD technologies - like Jenkins X Pipelines with Tekton.