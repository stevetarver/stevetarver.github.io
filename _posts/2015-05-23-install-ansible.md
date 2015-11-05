---
layout: post
title:  "Install Ansible"
subtitle: "Install Ansible and deploy Jenkins"
date:   2015-05-23 17:16:26
categories: mac ansible cloud homebrew vagrant vm
---

<img style="float: left;" src="/images/ansible_logo_black_square_small.png">


Ansible is an automated configuration management tool that integrates well with Vagrant, CI systems, and can be used to manage your operating system, infrastructure, and your code.

Ansible, Chef, Puppet, and SaltStack are the kings in this space. Ansible is the best choice for developers because it is the simplest to learn, the simplest to walk back into after a month of disuse, and is free. Most importantly, it was also built from the ground up to support multi-node, multi-tier, orchestrated installations including zero down time rolling updates.


## No Pets Allowed

One of the key concepts in cloud architecture is "No pets! Only herds."

In the olden days, it took forever to get a shiny new server. You would nurture and feed it and grow it into the perfect host for your application. If something happened to it, you (and your project) were devastated. It was your pet.

In self-service cloud, you can spin up a new server in a few minutes. You can deploy your application in perhaps less time. If one server breaks a leg, you have it destroyed and create a new one. You have a server farm and your servers are your herd.

Pet management focuses on incrementally building your server. Herd management focuses on incrementally building your server build script. Ansible is absolutely key to this whether you use it to build the whole stack from the ground up, or to coordinate container deployments among servers.

## Install Ansible

Once again, through the beauty of Homebrew

{% highlight bash %}
brew install ansible
{% endhighlight %}

## Install Ansible Playbooks

Ansible consists of playbooks and modules. Playbooks use a declarative language to define the desired end state of your server; modules interpret the declarations and bring the server to that state.

Many major infrastructure open source projects provide an Ansible playbook to install them. The Ansible community has provided playbooks to cover most of the remaining projects.

Community authors can contribute playbooks to the Ansible Galaxy. The Galaxy provides a centralized repository, community ratings, and the simplest way to install a playbook because it also manages dependencies.

Head out to http://galaxy.ansible.com, click on Browse Roles and you can search for playbooks by category or name. If you search for roles named "jenkins", sort by Average Score, and then click on Reverse Order, you will find a 5.0 rated jenkins role by geerlingguy. Clicking on the geerlingguy link reveals that he has contributed 58 roles with a pretty high average rating. With very little effort, you will find out that this is Jeff Geerling who wrote Ansible for DevOps. Seems like a reasonable choice.

The Ansible install included ansible-galaxy which is a "package manager" for Ansible roles. Once you find a role in Ansible Galaxy, you can install it with the user and role name.

{% highlight bash %}
[starver@rakshasa ~ 12:46:56]$ ansible-galaxy install geerlingguy.jenkins
- downloading role 'jenkins', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-jenkins/archive/1.2.3.tar.gz
- extracting geerlingguy.jenkins to /usr/local/etc/ansible/roles/geerlingguy.jenkins
- geerlingguy.jenkins was installed successfully
- adding dependency: geerlingguy.java
- downloading role 'java', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-java/archive/1.1.0.tar.gz
- extracting geerlingguy.java to /usr/local/etc/ansible/roles/geerlingguy.java
- geerlingguy.java was installed successfully
{% endhighlight %}

Terms: Playbooks are generally small yaml files that apply roles to servers. Roles are directories of yaml files that include plays, tasks, and other supporting files like templated configuration files, custom modules, and variants for different operating systems.

## Ansible + Vagrant

The easiest way to start playing with Ansible is to use it to provision a Vagrant. Create your vagrant with

{% highlight bash %}
cd ~/containers/vagrant
mkdir jenkins && cd jenkins
vagrant init hashicorp/precise64
vagrant up
{% endhighlight %}

Vagrant's Ansible integration makes applying roles trivial. Edit your Vagrantfile and add

{% highlight yaml %}
# provision with our playbook
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "jenkins-playbook.yml"
end
{% endhighlight %}

Now create the jenkins-playbook.yml

{% highlight yaml %}
---
- hosts: all 
  sudo: yes 
  roles:
    - name: Jenkins is installed
      role: geerlingguy.jenkins
{% endhighlight %}

Let the magic begin

{% highlight bash %}
vagrant provision
{% endhighlight %}

Ansible will update the shell with its status as it installs Java and Jenkins and some plugins. This will take a bit because it is downloading these packages in addition to installing them. We can cache these downloads for faster second uses, but we'll leave that for a future article.

## Using Jenkins

When Ansible finishes, the PLAY RECAP should show all tasks completed successfully, no failures. We can verify that by ssh'ing into the VM and grabbing the Jenkins home page.

{% highlight bash %}
[starver@rakshasa jenkins2 13:16:59]$ vagrant ssh
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)
 
 * Documentation:  https://help.ubuntu.com/
New release '14.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.
 
Welcome to your Vagrant-built virtual machine.
Last login: Sat May 23 19:16:57 2015 from 10.0.2.2
vagrant@precise64:~$ wget http://localhost:8080
--2015-05-23 19:19:37--  http://localhost:8080/
Resolving localhost (localhost)... 127.0.0.1
Connecting to localhost (localhost)|127.0.0.1|:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 11426 (11K) [text/html]
Saving to: `index.html'
 
100%[===========================================================>] 11,426      --.-K/s   in 0s      
 
2015-05-23 19:19:38 (234 MB/s) - `index.html' saved [11426/11426]
 
vagrant@precise64:~$ exit
logout
Connection to 127.0.0.1 closed.
{% endhighlight %}

Pretty interesting, but not so useful. Let's make Jenkins available to our browser.

Vagrant provides port forwarding from the guest to the host and like most everything in Vagrant, it is easy. Add this to your Vagrantfile (port 4567 is just a random port).

{% highlight bash %}
config.vm.network :forwarded_port, host: 4567, guest: 8080
{% endhighlight %}

Now, you can browse http://localhost:4567 and configure your Jenkins server.

**Tip**: I keep a port mapping in ~/containers/vagrant/README.md to avoid hard to resolve port collisions as the collection of vagrants grow.

## Epilog

This brief introduction hints at how easy using Ansible to create temporary, repeatable, infrastructure deployments is. Future articles will show how to separate provisioning from Vagrant so it can be applied to any server, writing your own playbooks for CI testing, and eventually, zero down time rolling updates. But, first, we still need to build on our foundations - figuring out how to use docker, cloud scale services, cloud scale applications, etc.