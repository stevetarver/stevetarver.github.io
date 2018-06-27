---
layout: post
title:  "Jenkins in Docker building docker images"
date:   2017-01-19 08:04:19
tags:   jenkins docker ci cd haproxy
---

**TL;DR** Deploy Jenkins in a Docker container and build docker images

<img style="float: left; border:15px solid white;" width="180" height="250" src="/images/logo/jenkins.png">

We're setting up a build pipeline that will include a Nexus Docker registry, a Jenkins build server, and then Kubernetes integration. The Jenkins facility is expected to pull from GitHub, build Docker images, push them to Nexus and insert them into various k8s clusters. This month will focus on simply setting up the Jenkins facility.

Why this article? Instructions on the [Jenkins Docker Hub](https://hub.docker.com/_/jenkins/) seem pretty clear.

I want to address the pain points presented in the [Nexus setup article](http://stevetarver.github.io/2016/11/22/nexus-in-docker.html) and there are four things that I wanted to add to have a more complete Jenkins install.

**Self-signed certs and SSL off-loading**: SSL off-loading is a common feature for components that live outside the cluster; it provides secure communication and is a base requirement for securing user passwords.

**Where to store data**: Data can be stored in a Docker volume or on a mounted disk. I use a mounted disk because it makes backup and restore from disk failure simple.

**How to build a Docker image inside a docker container**: Actually building docker images is not supported in the Jenkins base image. It took a bit of research to figure out which of the two standard approaches, DinD or DooD, is a best fit.

**TLS certificate install for the Docker Registry**: Finally, I want to be able to stand this whole build pipeline up on my laptop for fast iteration in its development. I am using self-signed certs for this POC and the install is not completely obvious.

Let's back up for a minute and ask "Why Jenkins?" Well, it's free, it has a ton a plugins, it is extensible, and it is well documented. Also, I have always used it, and so has our staff, so everyone is already familiar with it. I did make a cursory survey of the state of modern build tools, and none hit the sweet spot - most missed on the "free" part.

With this Jenkins deployment, I also want to fix some pain points I have suffered in the past, much like the previous Nexus article. Briefly, because these are the same issues discussed with Nexus:

* **Upgrades**: I have had some tragic fails in the past with Jenkins upgrades. What looked like a minor version upgrade turned into a three day build outage for the team. Using Jenkins in a Docker container provides simple upgrades and is part of upgrade failure mitigation.
* **Backups**: I want to provide a plan for server migration and disk failures. Having regular backups also supports the tragic upgrade failure plan.
* **HTTPS**: Providing secure communications is a basic step in protecting externally managed user credentials.
* **Authentication**: External account management offloads account management chores, reduces user account bloat, and provides a base for auditing build and deploy actions.

## Jenkins deployment

The [Jenkins Docker Hub page](https://hub.docker.com/_/jenkins/) provides good base instructions; let's build from there.

Starting with a Ubuntu 16 OS, 2 CPU, 4 GB RAM server, add a `/data` partition of 100GB. We will store all Jenkins data on a mounted volume within `/data`.

Now install Docker. After living with Docker for a while, it seems that Docker installations have become very personalized. Everyone has a favorite collection of tools they think belong in a basic docker install. This could include docker-compose, Ansible tools for managing docker, python libraries for scripting docker, etc. 

Nothing fancy is required here. You can use the basic [Docker install instructions](https://docs.docker.com/engine/installation/linux/ubuntu/), or, if you're an Ansible fan, you can use the `docker-ce` playbook and role in [my github repo](https://github.com/stevetarver/ansible-roles).

Once Docker is installed, we can configure and start Jenkins.

```shell
# All jenkins artifacts and configuration are store in jenkins_home
# which ww will map to jenkins-data to be consistent with other 
# mounted volumes
$ mkdir /data/jenkins-data

# Make that directory accessible to the jenkins user uid 1000
$ chown -R 1000 /data/jenkins-data

# Create and start the jenkins container
$ docker run --name jenkins   \
    -p 8080:8080            \
    -p 50000:50000          \
    -v /data/jenkins-data:/var/jenkins_home \
    -d jenkins

# Now tail the logs and look for the admin validation passwords
$ docker logs -f jenkins
# ...
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

18824e874ffb4f609a7cba40506d761c

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

Copy the key presented in the logs for Jenkins initialization - we'll need that shortly.

Then, set up a DNS name and you are ready for HAProxy SSL off-loading.

## HAProxy SSL offloading

On Ubuntu 16, HAProxy is installed with `apt-get install haproxy`.

Copy your server .pem to `/etc/haproxy/certs`. For these examples, we have been using the self-signed `splat.internal.mdg.com.pem` certificate. You can create your own self-signed cert following instructions in the [Self-signed CA and cert article](http://stevetarver.github.io/openssl/ca/certificate/jenkins/nexus/2016/10/11/self-signed-ca-cert.html) if needed.

Then adapt `/etc/haproxy/haproxy.cfg` to include these ideas:

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    tune.ssl.default-dh-param 2048

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # Default ciphers to use on SSL-enabled listening sockets.
    # For more information, see ciphers(1SSL). This list is from:
    #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 500
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# force https use via port 80 redirect
frontend http-in
     bind *:80
     redirect scheme https code 301 if !{ ssl_fc }

frontend jenkins
    bind *:443 ssl crt /etc/haproxy/certs/splat.internal.mdg.com.pem no-sslv3
    acl restricted_page path_beg /env
    http-request deny if restricted_page
    reqadd X-Forwarded-Proto:\ https if { ssl_fc }
    reqadd X-Proto:\ https if { ssl_fc }
    default_backend nodes

backend nodes
    mode http
    stats enable
    stats uri /haproxy?stats
    option forwardfor
    cookie SRVNAME insert
    server jenkins 10.98.100.214:8080
```

Verify the config, and apply it:

```shell
$ haproxy -c -f haproxy.cfg
Configuration file is valid
$ service haproxy restart
```

## Jenkins configuration

Browse to the new Jenkins server and you will be prompted for the admin verification captured in the logs or available on the server - you will only need this once to create your first admin user. Follow the prompts and install suggested plugins.

## Nexus integration

To connect Jenkins to our existing Nexus 3 deployment, we need to:

1. Create a Nexus jenkins user
1. Place the Nexus jenkins user credentials in Jenkins
1. Install the CloudBees Docker Build and Publish plugin
1. Install the CA root certificate if using a self-signed certificate.

Create a jenkins account in nexus

1. Click on the "Server administration and configuration" wheel icon at the top of the web page
2. Select Security -> Users in the left sidebar
3. Click the Create user button
4. ID: jenkins
5. First name: Leroy
6. Last name: Jenkins
7. Email: 
8. Password: 7qARmtzGBjrfon4HHATPWm46wmWeK8
9. Status: Active
10. Roles: nx-admin

Add our Nexus "jenkins" account to the Jenkins global credential store

1. Click on Credentials in the left sidebar
2. Click on the System link that appears
3. Click on Global credentials in the page body
3. Click on the adding some credentials link
4. Select Username with password
5. Use Scope: Global
6. username: jenkins
7. password: from nexus step above
8. ID: nexus-jenkins-account

Install the Docker Build and Publish plugin

1. Click on Manage Jenkins in the left sidebar
2. Click on the Manage Plugins link
3. Click on the Available tab
4. Filter by "docker" to find the CloudBees Docker Build and Publish plugin
5. Check that plugin, install, and restart from the web app

Install the CA root certificate if using a self-signed server certificate

```shell
# Create a separate directory for our certs to make them easy to find
mkdir /usr/share/ca-certificates/extra

# Copy the CA .crt file to this directory - must end in crt:
cp platform_ca.crt /usr/share/ca-certificates/extra/platform_ca.crt

# Let Ubuntu add the .crt file's path relative to /usr/share/ca-certificates to /etc/ca-certificates.conf:
dpkg-reconfigure ca-certificates
```


## DooD!

For Jenkins to build docker images, it needs access to the docker cli. There are two approaches: Docker in Docker: **DinD**, and Docker outside of Docker: **DooD**.

There is already decent coverage on which strategy to use. Jérôme Petazzoni, who wrote the first version of DinD, describes why you should use DooD [here](http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/).

Then, Adrian Mouat describes how to setup DooD for Jenkins [here](http://container-solutions.com/running-docker-in-jenkins-in-docker/), listing the benefits as:

> This looks like Docker-in-Docker, feels like Docker-in-Docker, but it’s not Docker-in-Docker: when this container will create more containers, those containers will be created in the top-level Docker. You will not experience nesting side effects, and the build cache will be shared across multiple invocations.

Mouat's instructions need a little tweeking; the derived Jenkins image can't build Docker images. Not finding fault here; that article was a great help, but it was written 2 years ago and needs a bit of updating to work with current infrastruction. I also want to add a little more functionality to appeal to a wider variety of Jenkins Docker plugins. So, let's pull together these ideas to provide up to date instructions. But first, a little background.

For the jenkins user inside the Jenkins Docker container to build Docker images, it must have access to the Docker socket and cli on the host server - it needs to be a sudoer. For clarity, this jenkins account only operates inside the Docker container, but giving it root access makes the Docker socket and cli we will bind mount into the container available.

Installing sudo in the Jenkins image turned out to be a bit of trouble. The Jenkins Docker image is built from `openjdk:8-jdk` which is built from `buildpack-deps:jessie-scm` which does not include the `sudo` package. Various grunting and groaning can be found on the issue, but, it will not be fixed - it is by design. We could script the sudo package install after the container is running, but it turns out to be easier if we just create a derived Jenkins image.

What's the problem? The `sudo` package needs `apt-utils` to complete `sudo` package installation and that is interactive; we need a headless install - no prompts. The solution is to use an environment variable and some `apt-get` arguments shown in the Dockerfile below.

The second problem is sudoer management; his instructions blow away the sudoer file with something that also doesn't work on Ubuntu 16. The "proper" way to setup a sudoer is shown below; by adding a jenkins sudo config to `/etc/sudoers.d` using the modern syntax.

Create this Dockerfile in `/root/jenkins-image/`.

```
FROM jenkins:2.32.1

USER root
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update \
      && apt-get install -y --no-install-recommends apt-utils \
      && apt-get install -y --no-install-recommends sudo \
      && rm -rf /var/lib/apt/lists/*
RUN echo "jenkins ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers.d/jenkins
ENV DEBIAN_FRONTEND teletype
```

Note that we are using a specific version tag so we know exactly what version we are using. After an upgrade failure, this provides a known-good version to roll back to. We will build our derived image using the same version tag.

```
docker build -t jenkins-mdg:2.32.1 .
```

Because of this article's presentation, we already have a Jenkins container running. This allows a simple test of the upgrade procedure: stop and delete the old container, run a new container.

This docker run command is a little different than the first; it includes bindings for both the docker socket and the docker cli. Our derived image makes the jenkins account a NOPASSWD sudoer so it can access the docker socket and cli, as well as all other "root" facilities on the host. In our scenario, we aren't too concerned because we are behind a VPN and all users will have dedicated accounts with strong passwords. You should pause and give some thought to what evil a rogue user could do if a Jenkins account was captured and they had full root access to the host server.

```shell
$ docker stop jenkins
# ...
$ docker rm jenkins
# ...
# docker run --name jenkins   \
    -p 8080:8080            \
    -p 50000:50000          \
    --restart=unless-stopped \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $(which docker):/usr/bin/docker \
    -v /data/jenkins-data:/var/jenkins_home \
    -d jenkins-mdg:2.32.1
```

At this point, you should take some time and create a decent README in `/root/jenkins-image/` to describe common operations and a process that ensures this image is always used.

If you visit the Jenkins web page, you can now create your first project. You will have to create a GitHub user that can pull source and tag your Docker image to point to your Docker Registry if you don't want that image pushed to Docker Hub.

## Epilog

Now you have a working Jenkins with the ability pull from GitHub, build Docker images, and push them to a repository, but there are a couple of items left on our todo list. The steps will vary based on your environment so I'll leave that as an exercise for the interested reader.

**TODO**:

* Establish a backup policy with your cloud provider for `/data/jenkins-data/`
* Integrate AD as an identity provider to off-load account management

