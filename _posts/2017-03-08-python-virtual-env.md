---
layout: post
title:  "Python Virtual Env Setup"
date:   2017-03-08 17:25:16
tags:   python pyenv virtualenv venv microservices

---

**TL;DR** It's just `pyenv`

<img style="float: left;" src="/images/logo/python.jpeg">

I work in a full-stack devops shop migrating a monolithic control portal to a Cloud Native Architecture. The most important thing in that shop is not the work we tackle today, but creating an amazing work place so that any work we may tackle can be fun. One of our core tenants is polyglot everything: we want to create developer joy by allowing developers to work in languages, environments, and datastores they love. Managing many technologies can be challenging, but arriving at that amazing workplace is well worth it.

One of the fundamental challenges is making each developer reasonably effective in each environment. When a java developer switches into a python environment, for example, they should not be befuddled, frustrated, etc. We address that through standards and best practices for each language: a java dev can look up how to manage python versions and environments and it will be the same for every python project in our environment.

When I jumped into a new software defined firewall project, I saw that standard was missing, and this is my journey creating it.


## Python virtual environments

**GOAL**: Provide seemless use of the correct python version and isolated environment for every project.

By default, python installs all dependencies into `site-packages` for a specific python version. This presents two problems:

* How do I keep each project's dependencies separate?
* How do I effortlessly switch between python projects and their environments?

Python provides virtual environments to address these needs, but these have some convenience problems:

* There is a [`virtualenv`](https://pypi.python.org/pypi/virtualenv) for Python2 and `pyvenv` and [`venv`](https://docs.python.org/3/library/venv.html) for Python3 - different managers for different language versions. This presents a bootstrapping problem if, for example, your OS has python2 installed and your project work is in python3.
* You must manually activate and deactivate the virtual environment. This can be tedious or error prone if you are switching projects frequently or have to revert to python2 for some ops work.

There is another option that feels a bit more comfortable: [pyenv](https://github.com/pyenv/pyenv). It:

* Provides simple access to many python2 and python3 versions.
* Avoids the bootstrapping problem; it is implemented as shell scripts.
* Consolidates all virtual environments in your home directory.
* Has a companion project [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) to isolate project dependencies.
* Activates the proper python version and environment by simply changing to your project directory.

## Install pyenv

`pyenv` and `pyenv-virtualenv` are most easily installed with Homebrew:

```
$ brew install pyenv
$ brew install pyenv-virtualenv
```

Now, connect it to your shell environment. I use [Oh My Zsh](https://github.com/robbyrussell/oh-my-zsh) so I add these lines near the end of `~/.zshrc`:

```
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

Complete install instructions: [`pyenv`](https://github.com/pyenv/pyenv#installation), [`pyenv-virtualenv`](https://github.com/pyenv/pyenv-virtualenv#installation)

## Create a virtual environment

There are three steps to creating a new virtual environment:

1. Install the python version (if it doesn't already exist)
1. Create a new virtual environment for the project
1. Link the project to that virtual environment

### Install a python version

First, ensure our mac is up to date. We need various xcode command line tools, like `zlib`. XCode is frequently updated, so if you already have these installed, you may have to acknowledge the license agreement for the latest version.

```
$ xcode-select --install
```

Our project deserves the latest and greatest python, so let's see what is available:

```
~ ᐅ pyenv install -l
Available versions:
  2.1.3
  2.2.3
  2.3.7
  2.4
  ...
```

This list is long and includes base distros as well as anaconda, miniconda, jython, pypy variants and stackless. We'll use the base 3.6.2 version.

```
$ pyenv install 3.6.2
```

Behind the scenes, `pyenv` has installed python 3.6.2 in your home directory.

```
~ ᐅ ll .pyenv/versions
total 0
drwxr-xr-x  3 starver  staff  102 Aug 12 12:57 .
drwxr-xr-x  4 starver  staff  136 Aug 12 12:23 ..
drwxr-xr-x  6 starver  staff  204 Aug 12 12:58 3.6.2
```

### Create a project virtual environment

The `pyenv-virtualenv` plugin manages our virtualenv within a `pyenv` context and intelligently uses `virtualenv` or `venv` appropriate to the python version. 

Our new project `ember-falcon-mongo` will use python 3.6.2, so to create a virtual environment for that project:

```
$ pyenv virtualenv 3.6.2 ember-falcon-mongo-3.6.2
```

`pyenv-virtualenv` created a symbolic link in `pyenv` versions

```
~ ᐅ ll .pyenv/versions
total 8
drwxr-xr-x  4 starver  staff  136 Aug 12 13:12 .
drwxr-xr-x  5 starver  staff  170 Aug 12 13:12 ..
drwxr-xr-x  7 starver  staff  238 Aug 12 13:12 3.6.2
lrwxr-xr-x  1 starver  staff   66 Aug 12 13:12 ember-falcon-mongo-3.6.2 -> /Users/starver/.pyenv/versions/3.6.2/envs/ember-falcon-mongo-3.6.2
```

which looks like:

```
~ ᐅ ll .pyenv/versions/ember-falcon-mongo-3.6.2/
total 8
drwxr-xr-x   6 starver  staff  204 Aug 12 13:12 .
drwxr-xr-x   3 starver  staff  102 Aug 12 13:12 ..
drwxr-xr-x  12 starver  staff  408 Aug 12 13:12 bin
drwxr-xr-x   2 starver  staff   68 Aug 12 13:12 include
drwxr-xr-x   3 starver  staff  102 Aug 12 13:12 lib
-rw-r--r--   1 starver  staff  101 Aug 12 13:12 pyvenv.cfg
```

### Link the project to the virtualenv

The project is linked to its virtualenv through a project file `.python-version` containing the name of the virtualenv.

Let's watch this work:

```
~/code/makara/ember-falcon-mongo (master ✘)✭ ᐅ python --version
Python 2.7.8
~/code/makara/ember-falcon-mongo (master ✘)✭ ᐅ echo 'ember-falcon-mongo-3.6.2' > .python-version
(ember-falcon-mongo-3.6.2) ~/code/makara/ember-falcon-mongo (master ✘)✭ ᐅ python --version
Python 3.6.2
(ember-falcon-mongo-3.6.2) ~/code/makara/ember-falcon-mongo (master ✘)✭ ᐅ cd ..
~/code/makara ᐅ python --version
Python 2.7.8
```
We can see python 3.6.2 activate when we add the `./python-version`, and then deactivate when we exit the project directory. Note that when activated, the virtualenv is shown as the first item in the command prompt.

**NOTE**: With Oh My Zsh, I have to restart the shell to pick up the changes.

### Recap

For brevity, setting up a new project looks like:

```
~/code/makara ᐅ mkdir new-project && cd new-project

~/code/makara/new-project ᐅ pyenv install 3.6.0
Downloading Python-3.6.0.tar.xz...
-> https://www.python.org/ftp/python/3.6.0/Python-3.6.0.tar.xz
Installing Python-3.6.0...
Installed Python-3.6.0 to /Users/starver/.pyenv/versions/3.6.0

~/code/makara/new-project ᐅ pyenv virtualenv 3.6.0 new-project-3.6.0
Requirement already satisfied: setuptools in /Users/starver/.pyenv/versions/3.6.0/envs/new-project-3.6.0/lib/python3.6/site-packages
Requirement already satisfied: pip in /Users/starver/.pyenv/versions/3.6.0/envs/new-project-3.6.0/lib/python3.6/site-packages

~/code/makara/new-project ᐅ echo 'new-project-3.6.0' > .python-version

(new-project-3.6.0) ~/code/makara/new-project ᐅ python --version
Python 3.6.0
```

and tearing down that virtualenv looks like:

```
(new-project-3.6.0) ~/code/makara/new-project ᐅ pyenv uninstall new-project-3.6.0
pyenv-virtualenv: remove /Users/starver/.pyenv/versions/3.6.0/envs/new-project-3.6.0? y

~/code/makara/new-project ᐅ
```

## Dependency management

I'm using the [ember-falcon-mongo](https://github.com/stevetarver/ember-falcon-mongo) project as a proving ground. This is a full stack POC with top level directories `backend`, `datastore`, and `frontend` - the falcon micro-service will go in the `backend` directory.

Dependency management presents three concerns we want to separate:

* development dependencies: `requirements.txt`
* production dependencies: `requirements-pd.txt`
* dependency versions: `constraints.txt`

These three files exist at the project root (`backend`) and are the only way we will install dependencies.

The `requirements.txt` file contains only development dependencies and links to `requirements-pd.txt`:

```
-r requirements-pd.txt

colorama
py
pytest
ipython
```

The `requirements-pd.txt` file contains only production dependencies and links to `constraints.txt`:

```
-c constraints.txt

falcon
gunicorn
pymongo
structlog
```

The `constraints.txt` lists semantic versions for production dependencies:

```
falcon==1.2.0
gunicorn==19.7.1
py==1.4.34
pymongo==3.4.0
pytest==3.1.3
structlog==17.2.0
```

When adding a dependency, add it to the appropriate requirements file and then:

```
$ pip install -r requirements.txt
```

After adding a production dependency, use `pip freeze -l` to identify the dependency version and add that to `constraints.txt`.

What did that command do to our virtualenv? Buried in our `~/.pyenv`, we can see that the base python installation is untouched; only `ember-falcon-mongo-3.6.2` site-packages has been modified:

```
~ ᐅ ll .pyenv/versions/ember-falcon-mongo-3.6.2/lib/python3.6/site-packages
total 160
drwxr-xr-x  56 starver  staff   1904 Aug 12 13:56 .
drwxr-xr-x   3 starver  staff    102 Aug 12 13:12 ..
drwxr-xr-x  24 starver  staff    816 Aug 12 13:56 IPython
drwxr-xr-x  10 starver  staff    340 Aug 12 13:56 Pygments-2.2.0.dist-info
drwxr-xr-x   9 starver  staff    306 Aug 12 13:56 __pycache__
drwxr-xr-x  39 starver  staff   1326 Aug 12 13:56 _pytest
drwxr-xr-x   6 starver  staff    204 Aug 12 13:56 appnope
```

## Epilog

Pretty happy with the approach overall - specifically the effortless `virtualenv` activation and keeping python and dependencies out of my project folder.

If this was new or interesting at all, you should checkout `pyenv` [advanced features](https://github.com/pyenv/pyenv/blob/master/COMMANDS.md) - lots of opportunities there to craft a comfy environment. 
