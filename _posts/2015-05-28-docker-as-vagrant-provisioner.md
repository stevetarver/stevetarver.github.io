---
layout: post
title:  "Docker as Vagrant Provisioner"
subtitle: "How do Vagrant, Ansible, Docker fit together, install Docker and run a container in Vagrant"
date:   2015-05-28 17:03:03
categories: mac ansible docker kitematic vagrant vm
---

![Ansible](/images/docker.png) 

Docker provides another way to run non-native software on your mac. More importantly, Docker allows you to build your own images and run them locally, under Vagrant, or on linux boxes in the cloud.

Ansible and Docker take different approaches to do basically the same thing: deploy a reproducible application configuration on a server. Their different approaches each have strengths that can be combined to form a robust cloud development and deployment platform.


## Vagrant, Ansible, Docker

Each of these technologies are robust and can be used in many ways so let's take a minute to narrow the focus to suit our needs.

Virtual machines provide an abstraction of a linux server that we can run our desktop. Using VMs allows local access to production configured software and an easy way to segregate that software.

Vagrant provides an abstraction and management layer for virtual machines. It simplifies using and provisioning all VM providers.

Docker provides a way to bundle your application package and required infrastructure. That container can be deployed to your laptop under Vagrant or to a linux machine.

Ansible allows us to create servers in the cloud, manage the operating system, install open source software, and install your code on top of that.

These technologies enable two fundamental cloud architecture concepts

- Reproducible deployments
- Easy horizontal scaling

Using production configured VMs and Docker to containerize applications provides a controlled configuration that can applied to local and peer desktops, CI servers, and production to eliminate the  "works on my desktop" syndrome.

Ansible can be used to deploy these building blocks to multi-tier, multi-node application stacks during a zero downtime rolling update.


## Docker Provisioner vs Provider

Docker can be used as both a Vagrant provider and provisioner. As a provider, Docker (conceptually) replaces VirtualBox and runs the Docker image. As a provisioner, Vagrant will install Docker on the VM and use that to run the image.

How to choose? The Docker provider is good for running a single image and provides a decent workflow for developing Dockerfiles.

The Docker provisioner allows multiple images and the ability to configure and link each. Vagrant allows you to use multiple provisioners to accommodate a mix of Docker images and Ansible playbooks. So, if you have a complex configuration, want everything on a single server, or simply want everything running on a production configured server, you will probably want to use the Docker provisioner.


## Where do I get Docker images?

Docker Hub provides an image repository, integrated image builds, workflow tools, and GitHub/BitBucket access. It is a great source of official images from prominent projects as well as a way to publish your own creations.

Step 1: Create an account at https://hub.docker.com/account/signup/.

Step 2: Browse Repos. Search for "jenkins" and you will see there is an official Jenkins Docker image - looks like a suitable candidate for our first image.


## Use Docker as a Provisioner

First, a fresh vagrant. Note that we are using trusty64 instead of precise64; precise64 would require complex kernel updates to meet Docker needs.

{% highlight bash %}
cd ~/containers/vagrant
mkdir jenkins-docker && cd jenkins-docker
vagrant init ubuntu/trusty64
vagrant up
{% endhighlight %}

Now add the following to your Vagrantfile to

- download and install docker
- download the jenkins Docker image
- run Jenkins on host port 4567


{% highlight ruby %}
# install jenkins in a docker container
config.vm.provision "docker" do |d|
  d.pull_images "jenkins"
end
 
config.vm.provision "docker", run: "always" do |d|
  d.run "jenkins",
    args: "-p 8080"
end
 
# forward host port 4567 to jenkins
config.vm.network :forwarded_port, host: 4567, guest: 8080
{% endhighlight %}

This configuration separates the Docker pull from the Docker run command so we only pull images during a "vagrant provision" and can set up the run command to run on each reload or up.

Provision the guest with

{% highlight ruby %}
vagrant provision
{% endhighlight %}

When provisioning is complete (could take a while depending on bandwidth), you should see the Jenkins home page at http://localhost:4567.


## Running Docker outside Vagrant

### Command line

When developing your own Dockerfiles/images, it may be more convenient to run Docker from the command line. Setup on a mac is straight-forward and will provide the CLI interface you have in linux.

The Docker daemon uses some linux specific kernel features so it can't run natively on a Mac. Docker provides Boot2Docker as a work around - it consists of a VirtualBox VM, docker, and a management tool. So, on a mac, we need to install the docker client and boot2docker to run the docker daemon.

{% highlight ruby %}
brew install docker
brew install boot2docker
boot2docker init
boot2docker up
{% endhighlight %}

boot2docker will show some connective tissue required for the client to talk to the daemon after it comes up. Mine looked like this:

{% highlight ruby %}
To connect the Docker client to the Docker daemon, please set:
    export DOCKER_HOST=tcp://192.168.59.103:2376
    export DOCKER_CERT_PATH=/Users/starver/.boot2docker/certs/boot2docker-vm
    export DOCKER_TLS_VERIFY=1
{% endhighlight %}

Adding this to your .bash_profile will make it available to future shells.

You now have complete access to the Docker CLI.

### Preloading Docker images

If you try to run an image that does not exist locally, it will automatically pull the image from Docker Hub. Images can take quite a while to download - you can preload them to avoid this delay.

Earlier, we searched Docker Hub for a Jenkins image. On the top right corner of that page is a docker pull command to pull that image.

{% highlight ruby %}
docker pull jenkins
{% endhighlight %}

You can see all the images you have downloaded with

{% highlight ruby %}
[starver@rakshasa ~ 22:28:39]$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
jenkins             latest              1520f72eb8b6        4 weeks ago         662 MB
{% endhighlight %}

Because we did not specify a tag with docker pull, we got the latest version; the default tag is "latest". You can update this image at any time by re-running the docker pull command.

### Desktop App

If you are not focusing on scripting and automating builds; you just want to run Docker images locally, Kitematic is your tool. Kitematic presents the functionality of Docker in a desktop app and integrates with the CLI.

The home page is https://kitematic.com and you can install with

{% highlight ruby %}
brew cask install kitematic
{% endhighlight %}

This will install ~/Applications/Kitematic (Beta).app. The first time you run it, you should press the Update Now button - the product is evolving rapidly.

After starting Kitematic, you log in to DockerHub and can browse or search the Hub. From there you can view image details on Docker Hub with a click or just download and run the image. After the image downloads and starts, you can click on the web preview to launch a browser if your image has a web app or easily view the logs. This is really a slick app. Kitematic did such a good job that Docker acquired them a couple of months ago.

