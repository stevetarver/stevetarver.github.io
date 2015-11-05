---
layout: post
title:  "Install Vagrant"
subtitle: "Install VirtualBox and Vagrant and use them to provision a VM"
date:   2015-05-10 17:15:16
categories: mac macdown vagrant virtualbox vm
---

<img style="float: left;" src="/images/vagrant-logo.png">


Vagrant makes it trivial to provision VMs on your desktop.  With it, you can quickly setup, provision and teardown VMs. You can use this for running non-native applications, use complex application stacks without contaminating your machine, test Ansible playbooks, Chef recipes and Docker images.

We’ll run through the basic setup for installing required applications on your desktop and provisioning your first server. 


## Install VirtualBox

Vagrant  manages virtual machine providers like VirtualBox, VMWare, KVM etx. You need a provider to use Vagrant. VirtualBox is a good choice because it is pretty stable and, well, free.

{% highlight bash %}
brew cask install virtualbox
{% endhighlight %}

When complete, there will be a new VirtualBox.app in /Applications. You can start it up, play with it, but we won't really ever need to use VirtualBox in this way - Vagrant will manage our VMs for us.

## Install Vagrant

If we already have VirtualBox, and it can manage VMs, why use Vagrant at all? Most importantly, it makes your desktop look very much like the cloud; you will likely use a very similar setup when deploying to the cloud. Also, everyone on your team can use the same Vagrant configuration so you can avoid the "It works on my computer" kind of problems. Also, also, if your development environment is virtualized, it is very easy to recreate during continuous integration and deployment. Pretty cool stuff when you wrap your head around it.

{% highlight bash %}
brew cask install vagrant
{% endhighlight %}

BTW: If the last post introduced Homebrew, is it paying for itself yet? Mac already made installs easy but this is amazing. 

## Organize your VM directories

You will likely have many VMs, so how do you organize them so they will make sense when you walk back into them next month?

I have a `~/containers/vagrant` directory with role based sub-directories. For example, I have a build directory for a box with Jenkins, Nexus, and Sonar installed, a mongo, redis, and kafka directory for playing with those technologies - you get the picture. I also keep a README.md at the vagrant level to record port mappings and forwarding and another in each sub-directory that describes the contents, where they came from, and how I have changed the VM.

What is an .md file? Markdown is a plain text markup language that can be easily be converted to other richer presentations like html. It is the standard documentation format in GitHub and seems to be invading other arenas as well. Here is a really sweet editor

{% highlight bash %}
brew cask install macdown
{% endhighlight %}

Now, to open file from the terminal with something like ```macdown README.md```, execute this

{% highlight bash %}
ln -s /Applications/MacDown.app/Contents/SharedSupport/bin/macdown /usr/local/bin/macdown
{% endhighlight %}

## Grab your first box

Box? The Vagrant docs say it well:

> Boxes are the package format for Vagrant environments. A box can be used by anyone on any platform that Vagrant supports to bring up an identical working environment.

So, where do I find boxes, and how do I pick one? Hashicorp has a repository of standard boxes at https://atlas.hashicorp.com/boxes/search. You can browse and find one you like. Ubuntu has an official Tasty build that seems pretty reasonable, and provides a solid base for future Docker work.

{% highlight bash %}
vagrant box add ubuntu/trusty64
{% endhighlight %}

## Create your first VM

Now the really interesting part - create and start a new VM. Using the organization described above, we'll create a "test" VM and start it.

{% highlight bash %}
cd ~/containers/vagrant
mkdir test && cd test
vagrant init ubuntu/trusty64
vagrant up
{% endhighlight %}

Which will produce

{% highlight bash %}
[starver@rakshasa test 13:09:05]$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ubuntu/trusty64'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ubuntu/trusty64' is up to date...
==> default: Setting the name of the VM: test_default_1432753761203_45497
==> default: Clearing any previously set forwarded ports...
==> default: Fixed port collision for 22 => 2222. Now on port 2200.
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 => 2200 (adapter 1)
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2200
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: Warning: Connection timeout. Retrying...
    default: Warning: Remote connection disconnect. Retrying...
    default: 
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default: 
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if its present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
==> default: Mounting shared folders...
    default: /vagrant => /Users/starver/containers/vagrant/test
[starver@rakshasa test 13:09:45]$ 
{% endhighlight %}

Initialization and startup in 40 seconds. Nice! Subsequent startups on this OS should take 20 seconds. A smaller, more focused OS will take less time.

From the listing we see that we can ssh to the VM on 2200 and that our local directory is mapped to /vagrant on the VM - a very convenient way to share files. Vagrant is packed full of convenience like this, because that is the whole reason it exists. From the "test" directory, you can

{% highlight bash %}
vagrant ssh
{% endhighlight %}

and you are in the VM.

## Managing your vagrants

You can get an overview of all vagrants with the VirtualBox app - start it up and you should see your "test" VM running. Vagrant provides this summary information as well.

{% highlight bash %}
[starver@rakshasa test 13:10:52]$ vagrant global-status
id       name    provider   state    directory                           
-------------------------------------------------------------------------
0b0c8fd  default virtualbox running /Users/starver/vagrant/test     
{% endhighlight %}


You can manage vagrants from within the vagrant directory or using the id from global-status 
To stop the VM in the "test" directory

{% highlight bash %}
vagrant halt
{% endhighlight %}

Or, from any directory

{% highlight bash %}
vagrant halt 0b0c8fd
{% endhighlight %}

To delete the VM and all cached information about, including that held by VirtualBox

{% highlight bash %}
vagrant destroy 0b0c8fd
{% endhighlight %}

Then delete the "test" directory. If you take a peek in VirtualBox, you will see it is missing from VM list as well.

