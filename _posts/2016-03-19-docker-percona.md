---
layout: post
title:  "Percona as a Docker container"
date:   2016-03-19 17:25:16
tags: docker percona mysql sql
---

**TL;DR** Setup a Docker Percona image for general local use.

<img style="float: left;" src="/images/logo/percona_logo.png">

When you start a new project, how do you guarantee a pristine database as a starting point? How do you switch between projects without one database contaminating the other? What about testing one version of MySQL against another prior to rolling out an upgrade to production? How do you make production rollouts as simple and low risk as possible, including the ability to rollback if you find an obscure bug you can't live with?

In short: Docker.

Using docker containers, you can have multiple versions of multiple databases at the ready: dormant, running individually, or running concurrently. You can also quickly tear down the cruft and spin up new, pristine instances. 

What about production? If your database server is running a dockerized MySQL, you can download the next upgrade image, apply your own hardening, and test the new image in DEV. When proven, you can roll that image out to production, turn down the old image, turn up the new and you are good to go - you eliminate the possiblity of production only bugs. And if you find a difference in the new image that you can't live with, you can turn down the new and turn up the old in a minute, removing the bug.

So, Docker sounds pretty good, but what about Percona? I have been using Percona in production for a while now and am kinda impressed.

[Percona](https://www.percona.com/software/mysql-database), in their own words:

> Percona is committed to producing open source software for Percona Server, MySQL®, MariaDB®, and MySQL DBaaS users. Our solutions are also commonly adopted by OpenStack users who require high performance and high availability for optimal cloud operations. We offer our own MySQL software solutions and also participate actively in many non-Percona software projects. All Percona software is open source and free of charge.

**OS Note**: For this post, if you are on Windows, the Docker Toolbox concepts and commands should be similar. If on Linux, you can omit all the Docker Machine parts and work on your base distro using just the docker commands. Eleven linux distro's are supported, Ubuntu instruction are [here](https://docs.docker.com/engine/installation/linux/ubuntulinux/). You can omit the optional configurations for your local machine for now.

## Strategy

How do we get started with Docker and Percona/MySQL for your personal DEV environment?
For my laptop, I

- run MySQL compatible databases in Docker Machines
- create scripts to generate or extract sample data for easy seeding of new database
- document how to stand up specific docker machines, and the purpose and content of long running instances

## Install Docker

I used to recommend installing Docker with HomeBrew. The Docker Toolbox consolidates all the docker pieces needed by Mac (and Windows) into a simple installer. If you don't have VirtualBox installed, it will install it for you. Upgrading is as simple as downloading the latest installer and running it. All the things you would expect from a mature product.

If you have the old boot2docker and other tools installed, remove them, head out to the [Docker Toolbox download page](https://www.docker.com/products/docker-toolbox), grab the latest version, and install it - it's what I'll be using throughout the rest of this post.

If you haven't used Kitematic yet, it is a slick GUI on top of docker that allows you to quickly find and spin up images for use. I usually do everything in the terminal to keep my docker tool knowledge up to date and practice the commands I will use building images and operating on them in production.

## On Docker Machines

There are two ways to run docker images on a Mac: 

- run all containers on a single docker machine
- run each container on a separate docker machine

Each docker machine represents a separate server, a separate VirtualBox VM. I tend to run every image in a separate docker machine to mimic what I want in production - every tier of functionality is conceptually a cluster that can grow or shrink or upgrade as necessary. This strategy makes the docker machine and its image a consumable; something I can readily construct or throw away. 

In other situations I might use docker compose to link containers within a docker machine. If I was building a low volume ELK stack, this would make a lot of sense. For my database DEV use, being able to easily replace only the database part of the stack makes one container per docker machine the right solution.

## Create the Docker Machine

`docker-machine` is used to create and manage Docker VMs. Once the machine is started, you connect your terminal to it and have full `docker` command access.

Because I want to compare different versions of Percona, I always use a tagged docker image; never latest. My docker machine reflects that tag:

```
$ docker-machine create -d virtualbox percona-5.6
Running pre-create checks...
Creating machine...
(percona-5.6) Copying /Users/starver/.docker/machine/cache/boot2docker.iso to /Users/starver/.docker/machine/machines/percona-5.6/boot2docker.iso...
(percona-5.6) Creating VirtualBox VM...
(percona-5.6) Creating SSH key...
(percona-5.6) Starting the VM...
(percona-5.6) Check network to re-create if needed...
(percona-5.6) Waiting for an IP...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env percona-5.6
```
Now connect the terminal to the new docker machine

```shell
$ docker-machine env percona-5.6
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/starver/.docker/machine/machines/percona-5.6"
export DOCKER_MACHINE_NAME="percona-5.6"
# Run this command to configure your shell: 
# eval $(docker-machine env percona-5.6)

$ eval $(docker-machine env percona-5.6)
```

It doesn't look like much has changed, but docker is now available and I can

```shell
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

## On persistent storage

ref: [MySQL Docker storage options](http://blog.arungupta.me/docker-mysql-persistence/)

How do I ensure that my data persists through docker and docker-machine restarts? One might think, as I did, to map a local directory to the docker container. This is the common method for sharing files between the host and container, right? After a lot of playing around and research, I found that this is not practical for containers that write their own directory structure.

There are some [real problems](https://github.com/boot2docker/boot2docker/issues/581) using a local directory as persistent storage for directory writing containers on Mac (Windows). The underlying boot2docker mounts `/Users` allowing container root accounts to write to sub-directories. The problem is that every container has its own user that will want to write to the volumne: e.g `mysql`, `www-data`, etc. and getting the perms right when creating files and directories is tough for the general case. This issue has been open for a year and a half and has no end in sight.

There are work-arounds listed in the above thread that consist of mounting shares with the appropriate user/group and specifying those shares in the boot2docker config. But that implies you would need a separate boot2docker config for each container user - just too tedious for me unless I really need it.

This post on [Docker MySQL Persistence](http://blog.arungupta.me/docker-mysql-persistence/) provides a good comparison of persistent storage options. The author omits working with mapped `/Users` directories - perhaps because of the problems above.

For most uses, having the data persist across docker machine restarts will be sufficient, no need to mount a `/data` directory as a local volume - and bonus - you get that with a standard docker config.

## Run Percona

If you visit the [Percona Docker Hub page](https://hub.docker.com/_/percona/), you will notice that it looks very much like the [MySQL Docker Hub page](https://hub.docker.com/_/mysql/). Percona is intended as a drop-in MySQL replacement and you should be able to substitute MySQL for Percona throughout these instructions.

The docker `run` command combines the `create` and `start` commands. `run` provides a simple way to pull the container, configure it, and start it. After the initial run, you will `docker start image-name`.

If you decide you want a different configuration later, you can `docker rmi -f image:tag` and re-run `run` - one good reason to document what you are doing as you go along. Also, once you have pulled the image, it is cached in the machine, and creating a new container based on that image is near instantaneous.

Our `docker run` arguments are:

- `--name`: the name of the docker image inside the docker machine. I "percona" (without a tag) so any scripts I develop will work with any percona target.
- `-p`: the mysql port that will be exposed to the docker host.
- `--restart=always`: automatically start this container when the docker machine is started
- `-v`: map a local directory onto `/etc/mysql/conf.d` as a way to override `my.cnf` properties.
- `-e MYSQL_ROOT_PASSWORD`: an environment variable that will be used to initialize the database root account password.
- `-d`: run the named container detached; in the background

```shell
$ docker run --name percona \
    -p 3306:3306 \
    --restart=always \
    -v ~/containers/docker/percona/conf.d:/etc/mysql/conf.d \
    -e MYSQL_ROOT_PASSWORD=root \
    -d percona:5.6
Unable to find image 'percona:5.6' locally
5.6: Pulling from library/percona
fdd5d7827f33: Pull complete 
a3ed95caeb02: Pull complete 
2d9b55a37647: Pull complete 
88a4bacbf934: Pull complete 
d2f0eb2850d3: Pull complete 
b62e4169c09f: Pull complete 
cc90f737067f: Pull complete 
38ce52ca4637: Pull complete 
ddecfc474034: Pull complete 
25674476437b: Pull complete 
Digest: sha256:054a7e20db5c130871803bfaa43c845a34215700493717517be2bd81dbb53c6b
Status: Downloaded newer image for percona:5.6
875195b37308dd6986089f9c6fab0904d2c56810b6e5f746a350a5c4313c9807
[starver@bhooteshwara ~ 11:52:00]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
875195b37308        percona:5.6         "/docker-entrypoint.s"   7 seconds ago       Up 7 seconds        3306/tcp            percona
```

Right now, I can connect to the MySQL command line client. Note that I use `--rm percona:5.6`; including the version tag. If left bare, docker will assume `latest` and try to pull that image.

```shell
$ docker run -it --link percona:mysql --rm percona:5.6 sh \
  -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.29-76.2 Percona Server (GPL), Release 76.2, Revision ddf26fe

Copyright (c) 2009-2016 Percona LLC and/or its affiliates
Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show schemas;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

mysql> 
```

Wondering what is going on with your database? You can view the logs from the terminal with

`$ docker logs percona`

You can stop the container by using 2 digits of its contain id.

```shell
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
875195b37308        percona:5.6         "/docker-entrypoint.s"   31 minutes ago      Up 31 minutes       3306/tcp            percona

$ docker stop 87
87
$ 
```

## Seed data

There are several options for seeding the database including by script, by data dump, by persistent docker data container, and persistent volume. The simplest is probalby pasting a script into the mysql command line client.

If you don't have your own data, you can clone my [sample-data](https://github.com/stevetarver/sample-data) repo and use the mysql script for contacts.

Connect to the mysql command line client as shown above, create a `test` schema and use it, and paste in your init script.

```shell
$ docker run -it --link percona:mysql --rm percona:5.6 sh \
  -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
# ...

mysql> create schema test;
Query OK, 1 row affected (0.00 sec)

mysql> use test;
Database changed

mysql> DROP TABLE IF EXISTS `contacts`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `contacts` (
# ...

mysql> select * from contacts;
# ...
```

## Connect MySQL Workbench to Percona

Our run (create) command exposed port 3306 on the docker machine. We can find the docker machine ip with

```shell
$ docker-machine ip percona-5.6
192.168.99.100
```
Open MySQL Workbench and create a new connection using

```
Connection Name: percona-5.6 docker local
Hostname: 192.168.99.100
Username: root
Password: root
```
Now you should be able to open the connection, `select * from test.contacts` and see all your data.

## Externalized configuration

The Percona my.cnf is modified in the standard way: add *.cnf files to `/etc/mysql/conf.d`. Our run command mapped `~/containers/docker/percona/conf.d` to `/etc/mysql/conf.d` so any additions to that directory will be picked up on conatainer restart.

Additionally, most of the options you would normally set in `my.cnf` can be passed as docker `run` arguments or in a docker config file. Of course since these are specified in the create phase, changing them requires a container tear down and re-create - using the conf.d extension point is a clear winner.

To see all options you can configure:

`docker run -it --rm percona:tag --verbose --help`

## Let's make this easier

Pulling all this together, we can create a bash script that will tear down and recreate the Percona docker machine and container for us.

```bash
#!/bin/bash

# Creates and initializes a percona-5.6 docker machine, pulls
# the percona:5.6 docker image, configures it, and seeds it.
#
# Sources my.cnf additions from
#   ~/containers/docker/percona/conf.d
# Initializes database from
#   ~/containers/docker/percona/scripts/us-500.sql

MACHINE_NAME=percona-5.6
DOCKER_IMAGE=percona:5.6
ROOT_PASSWORD=root
MY_CNF_DIR=~/containers/docker/percona/conf.d

# Delete any existing docker machine named percona-5.6
docker-machine stop $MACHINE_NAME
docker-machine rm -y $MACHINE_NAME

# Create the new docker-machine and connect to it
docker-machine create -d virtualbox $MACHINE_NAME
eval $(docker-machine env $MACHINE_NAME)

# Create the conf.d directory if it does not already exist
mkdir -p $MY_CNF_DIR

# Pull the docker image, create our container, and start it
docker run --name percona \
    -p 3306:3306 \
    --restart=always \
    -v $MY_CNF_DIR:/etc/mysql/conf.d \
    -e MYSQL_ROOT_PASSWORD=$ROOT_PASSWORD \
    -d $DOCKER_IMAGE

# Give the Percona image some time to start and initialize
sleep 10

# Connect to MySQL and seed the database
docker run -it --link percona:mysql \
  -v ~/containers/docker/percona/scripts:/tmp/import --rm percona:5.6 sh \
  -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" < /tmp/import/us-500.sql'
``` 

_NOTE_: If you are using the contacts script from my `sample-data`, you need to add this to the top.

```sql
create schema test;
use test;
```

This takes about 4 minutes to run on my laptop. We could really save some time by detecting that the docker machine already existed and just deleting the existing container. I'll leave that as an exercise to the interested reader.

