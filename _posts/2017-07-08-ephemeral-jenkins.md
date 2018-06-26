---
layout: post
title:  "An Ephemeral Jenkins"
date:   2017-07-08 11:11:11
tags:   jenkins docker ci cd

---

**TL;DR** A strategy for a safely updated, ephemeral, Jenkins build system.

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/jenkins.png">

Have you ever killed your Jenkins server? I don't mean just giving it a bruise that it can heal from, but murder it? We murdered ours with a seemingly simple version upgrade. 

The biggest challenge was undoing the piecemeal configuration changes made during the upgrade and then figuring out what versions of each plugin worked in that previous version. 

Jenkins has long had the dubious distinction of being one of the most despised, and most used, build systems. Its greatest asset is also its greatest problem: the plugin system and scant quality checks or management facilities around it. The migration to a Kubernetes cluster provides the remaining tools I need to insulate myself from danger and create a reasonably worry free, ephemeral, upgradeable Jenkins build system. Today, we're going to walk through that system.

## Goals

To limit article scope and size, we are going to focus on deploying and upgrading Jenkins and avoid how to use Jenkins, pipelines, standards, etc. This also limits the goals we'll address. From a client perspective, we want:

* A recent Jenkins - keep the system up to date to take advantage of the rapidly evolving pipeline system
* No build system outages - daily, during upgrades, ever

## The strategy

A good, disposable Jenkins strategy is built on these pillars:

**A Jenkins build artifact**

We need a fully configured, versioned, tested, immutable, and deployable Jenkins build artifact. Let's break that apart a bit... What this really means is that:

* we have a versioned Jenkins distro
* we have deployed our favorite plugin set to that distro
* Jenkins and all the plugins work properly with all of our build jobs
* we bundle that up in a way that no one can modify it
* and in a way that we can deploy to our platform

**A deployment process**

Now that we have a single distributable, we need a process to:

* introduce the new version
* test it 
* allow the communit to test it
* migrate it into production incorporating current configuration, while rolling the old version out, with minimal downtime
* retain the old version for some time in case there was a subtle bug we missed
* delete the old version

**Automated configuration backups**

Once in production, we need to have periodic backups of the configuration so that when a disk has a problem, or the configuration is bad or corrupted, we can roll back to a previous, known-good state.

**Self-healing**

Java processes get tired, and sick, and they need a little healing - this should be automatically detected and corrected.

## The Jenkins build artifact

If not obvious, 'Kubernetes' and 'binary build artifact for a distro' means a docker image. We need a few more components than the base [Jenkins image](https://hub.docker.com/r/jenkins/jenkins/) provides:

* docker: so we can build docker images - our fundamental deployment unit
* helm: so we can deploy our docker images to our k8s clusters
* vim: diagnostics in the image

and we'll take advantage of some of the configuration facilities that [Jenkins provides](https://hub.docker.com/_/jenkins/):

* load a plugin set during initial startup
* set executor count 

which means we'll build a derived docker image. 

Each derived image we build will be a known good configuration: 

* a not massively broken Jenkins
* with a set of our favorite plugins that work on Jenkins and with each other
* that builds all of our existing jobs

Since we want to move from known good state to known good state, we need to move from one specific version of Jenkins to another specific version of Jenkins. We will abhor Docker `HEAD` tags like `latest` or `lts`. We will pick specific versions that have had a bit of soak time - perhaps (latest_stable - 2). We can pick this version, and review changes, from the Jenkins [LTS Changelog](https://jenkins.io/changelog-stable/). 

Now that we have some rules for picking a specific Jenkins image version, let's walk through the facilities we want to add. Installing Docker requires the most consideration, so let's start there.

Our Jenkins is in a docker container, that lives in a pod, on a node (vm), in a Kubernetes cluster, AND it wants to make and run other docker images. Nesting Docker within Docker can present some significant challenges. As Docker has evolved, there have been a few different strategies like Docker-in-Docker (DinD), Docker-outside-of-Docker (DooD) described by [Jérôme Petazzoni](http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/) and later [A. J. Ricoveri](https://github.com/axltxl/docker-jenkins-dood), [Sreenivas Makam](https://sreeninet.wordpress.com/2016/12/23/docker-in-docker-and-play-with-docker/), and [Teracy](http://blog.teracy.com/2017/09/11/how-to-use-docker-in-docker-dind-and-docker-outside-of-docker-dood-for-local-ci-testing/). That's a lot to wade through, but our prioritization of a couple of concerns makes choosing a solution simple:

* We are constantly building images - caching is critical for speed and bandwidth costs.
* We want simplicity - we want a speedy diagnosis of problems encountered with this nested structure.

Solution? In the Jenkins image, and any CI containers that image will run that need docker, we will run the docker cli connected to the host DOCKER_SOCK and not run the docker daemon. This means that all docker images will reside on the Kubernetes node and we will have image and layer caching at that level. With this strategy, our Jenkins container can build and run Docker images, run CI containers that build and Docker images, as long as we differentiate all images by tag and containers by name. Anything more is asking for trouble, and I haven't run into a legit need for more yet - just people overloading a single facility beyond its intent.

So, how do you connect a docker cli to its host daemon? Here's an example run command from [the jenkins docker image build run command](https://github.com/stevetarver/infra-images/blob/master/jenkins/4.run.sh):

```bash
    ID=$(docker run --name ${CONTAINER_NAME}            \
        -p 8080:8080                                    \
        -p 50000:50000                                  \
        --restart=unless-stopped                        \
        -v /var/run/docker.sock:/var/run/docker.sock    \
        -v ${JENKINS_HOME}:/var/jenkins_home            \
        -d ${IMAGE_NAMETAG})
```

The important line is the first bind mount; it is binding the internal docker.sock fd to the host fd.

On to Helm, which turns out to be a non-issue. You [download the version deployed in your cluster](https://github.com/stevetarver/infra-images/blob/master/jenkins/1.setup.sh), [copy it to the docker image and set appropriate perms](https://github.com/stevetarver/infra-images/blob/master/jenkins/Dockerfile), init it, and eat the error. Then provide a `kube.config` that points to a kubectl in the cluster - perhaps best done as part of the deploy process.

### Developing the plugin list

At this point, we have a Jenkins that is not massively busted because the version we selected has been out for a bit and we have looked for people squaking loudly. Now to the second tedious chore - developing a workable, versioned plugin list.

We need two things:

* A list of plugins and versions that are supposed to work together
* A way to install those plugins in our build artifact

The only way I have found to generate the plugin list is to create a pristine Jenkins deployment, install the plugins I need, and then export that list. Sounds simple, but Jenkins only provides a list of plugins by category, which means each one is frequently repeated, and when I built my original list, there were 90 pages to wade through. That was enough to set my resovlve to get out of this Jenkins hell and into one of the modern replacements, but more on that later. If you need to do this, I am truly sorry, and here is [a cookbook of what I did](https://github.com/stevetarver/infra-images/blob/master/jenkins/README.md#creating-the-initial-plugin-list).

We can really simplify that process if we have a known good Jenkins: your existing system, or my repo for example. We can use a little know Jenkins facility and `jq` to create the specially formatted file. On a running Jenkins, you can browse to:

```
http://${JENKINS_URL}/pluginManager/api/json?depth=1&tree=plugins[shortName,version,active]
```

and see the currently installed plugins. The build process has a script built around that to [generate the plugin-list.txt](https://github.com/stevetarver/infra-images/blob/master/jenkins/update_plugin_list.sh). This manual part of the build process becomes: identify, or standup a Jenkins with a known good set of plugins, and use the script to capture the plugin set, generate a well-formed plugin list, and copy that to the Docker image during build.

**NOTE**: 

* The Dockerfile, build scripts, and admin facilities described above are [here](https://github.com/stevetarver/infra-images/tree/master/jenkins).
* You can download the Jenkins image described above with `docker pull stevetarver/jenkins:2.107.2-r0` or clone the [repo](https://github.com/stevetarver/infra-images), tweek, and build your own.


## The deployment process

Now that we have a suitable Jenkins docker image, let's think through the deployment process.

### Goals

Our Jenkins upgrade process should:

* minimize daily build and cluster deploy interruptions.
* migrate all existing facts, secrets, and other configuration from the old to the new deployment.
* not corrupt the `jenkins_home` directory. This requires a complete Jenkins facility rebuild and longer daily outages.
* allow all current jobs to be tested in the upgraded environment. This provides confidence that upgrading the main facility will not cause build/deploy outages.
* provide for rollback to last known-good state. In cases where testing did not reveal all defects, this restores build/deploy quickly.

### Solution

To provide for roll-forward and roll-back, we will need three logical names:

* `jenkins-next`: the next jenkins deployment - the one under validation, that will become the current
* `jenkins`: the current jenkins deployment - what everyone uses daily
* `jenkins-last`: the last jenkins deployment - what we will roll back to if - break glass in case of emergency

Each logical deployment has a distinct DNS name and is registered in the Kubernetes ingress through our helm chart

* http://jenkins-next.makara.dom
* http://jenkins.makara.dom
* http://jenkins-last.makara.dom

### Process

At a high level, the jenkins upgrade process is:

1. Create a new jenkins derived docker image
1. Run the jenkins helm chart deploy script to create a new `jenkins-next`
1. Follow instructions to migrate the `jenkins_home` directory from `jenkins` to `jenkins-next`
1. Deploy reference apps as a basic test, and resolve issues on `jenkins-next`
1. Have team build all their jobs and resolve issues on `jenkins-next`
1. When `jenkins-next` is deemed of suitable quality:
    1. Run the jenkins helm chart to turn the `jenkins` deployment to `jenkins-last`
    1. Perform some manual grooming of the `jenkins_home` directory on `jenkins_last`: disable jobs, setup redirect, security
    1. Run the jenkins helm chart to turn the `jenkins-next` deployment to `jenkins`
    1. Perform some manual grooming of `jenkins_home` directory on `jenkins`: setup redirect, security
1. Have team build all their jobs and resolve issues on `jenkins`
1. If unresolvable errors are found, reverse the process to restore `jenkins-last` to `jenkins`
1. When sufficient time has passed to have confidence in the new deploy, delete the `jenkins-last` deploy

**NOTE**: This process is cookbooked in the Jenkins Helm chart directory.

## Automated backups

The good news is that Jenkins now stores every piece of configuration and data in the `jenkins_home` directory. I can use my cloud provider's simple backup strategy to keep a month of daily backups on the cheap and simple. If I have a tragic loss of the persistent volume claim holding `jenkins_home`, I can deploy our Jenkins, pause that deploy, restore from backup, continue the deploy.

What if I have a tragic loss of backup? One would think that, well... unthinkable... until the S3 outage in February.

I address this with four strategies:

1. The Jenkins deploy provides base Jenkins configuration.
1. Build jobs are create by script with a couple of input parameters.
1. Facts are stored in the GitHub repository, not in Jenkins.
1. Secrets are stored in a secure location and entered manually.

Secrets are currently a weak point because they must be manually entered. My todo list includes a [Hashicorp Vault](https://www.vaultproject.io/) integration that should eliminate this manual step.

To recap, our failure cases are:

* Jenkins server dies: Stand up a new server, restore the archived `jenkins_home`, deploy Jenkins.
* Jenkins version upgrade failure: Restore the archived `jenkins_home`, deploy Jenkins.
* Jenkins version upgrade failure and complete `jenkins_home` backup failure: Deploy Jenkins and recreate repo build pipeline skeletons. 

**NOTE** When Jenkins is deployed to the Kubernetes cluster, the `jenkins_home` directory will live on a ceph cluster so we can frequently omit the restore `jenkins_home` directory step.

## Self-healing

In theory, this is very simple: define readiness and liveness probes, expose them to Kubernetes, and let k8s decide when the pod needs to be rescheduled. 

To be honest... I haven't done this yet. We dedicated one node to builds and just let Jenkins run. It has months of run time and no problems. I am leaving this chore until it demands my attention. I know it is doable, just not high enough priority yet.

**TODO**: _Boy, oh boy is that a punk thing to do, Steve. (steve hangs head, but only for a minute cause he also holds the pager)._

## Epilog

OK! I'm done! This was some tedious work and I am rededicating myself to getting out of the Jenkins business and on to something like [Spinnaker](https://www.spinnaker.io/) or [Go CD](https://www.gocd.org/) - but not for some months to come.

In the meantime, I think we have a start on a build system that will help keep [PagerDuty](https://www.pagerduty.com/) silent.


