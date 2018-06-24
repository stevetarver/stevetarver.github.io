---
layout: post
title:  "Install Homebrew"
date:   2015-05-09 19:32:15
tags: architecture cloud mac
---

**TL;DR** Install Homebrew and some common packages

<img style="float: left;" src="/images/logo/homebrew_osx_logo.png">


Cloud development requires an arsenal of rapidly evolving tools, and the ability to easily upgrade and downgrade them; you need a package manager.

## Evolution

With OS X's heritage in *nix, it was natural for the open source community to create package managers soon after its introduction.

First there was Fink, a dpkg port (2000), then DarwinPorts based on FreeBSD Ports which became MacPorts. Both started with a bang and flourish but then suffered from instability and a lack of support. I was happily searching for something new. 

## How to pick a package manager

What is important to me in a package manager?

- How active is the project
- How many packages are provided - and how interesting are they
- How frequently are packages added/updated
- How well does it integrate into the OS
- How invasive is it

Homebrew is a relatively new package manager built in ruby on GitHub. Starting in 2012, it had the highest number of contributors on GitHub. It currently has a little over 3000 packages, which is small compared to its peers, but they are the important ones. Packages are added by maintainers and pulled from the community. It integrates well into the OS, is kept under its own tree, and is easily removed. 
I really like Homebrew, use it exclusively, and prefer it to downloading and installing packages from the internet. It seems perfectly in tune with modern development, especially in the cloud domain. 

## Install HomeBrew

Installation is straight forward and fully described on the home page http://brew.sh. 
Paste this into a terminal and follow the directions

{% highlight bash %}
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
{% endhighlight %}

Note: If you haven't installed XCode or command line developer tools, you will be prompted to when you execute the above command. If you don't need XCode, install the command line tools only - they are much smaller. 

### Install Cool Stuff

One of the really cool things about brew installs is that the installed packages are available to your IDE or other applications that use them. If you have ever struggled to get your IDE to find your hand installed groovy home, only to have that method break on the next OS upgrade, you get this.

I tend to install all command line and developer tools with brew, and install infrastructure (ActiveMQ, Kafka, Redis, MySQL, Mongo, etc.) under Vagrant and Docker. Keeping infrastructure components on Vagrant boxes prevents my mac from becoming a technology dumping ground and enables a pretty seemless migration to the cloud. More on that later.

So, lets try a couple of installs to watch it work. 

{% highlight bash %}
brew install wget
{% endhighlight %}

Brew tells you what its doing as it does it, and you may notice that it installed the openssl dependency.

Command line tools can be cool, but let's try something a little more interesting. If it isn't already, git will become one of your primary development tools - let's install that now. 

{% highlight bash %}
brew install git
{% endhighlight %}

Now you can do all the advanced things with git that your IDE does not allow. But what's next? How about languages like groovy, go, play, build systems like gradle? Hard to guess what is interesting to you, but if you are installing a development tool, you can see if brew has a formula for it with

{% highlight bash %}
brew search FORMULA
{% endhighlight %}

If you searched for "node", you would see some items prefixed with Caskroom. This is the gateway to about 2500 large binary installs through brew. You can install and update everything from development infrastructure to common apps like Google Chrome. Let's install that now so it is ready when we need it.

{% highlight bash %}
brew install caskroom/cask/brew-cask
{% endhighlight %}

That's it. Go crazy. Install something you don't like? Then just

{% highlight bash %}
brew uninstall FORMULA
{% endhighlight %}
