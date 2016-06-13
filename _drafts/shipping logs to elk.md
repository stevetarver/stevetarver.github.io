## Delivering logs to ELK

A 12 factor app should log to STDOUT and let the execution environment deal with catching and sending those logs to the appropriate destination. 

To provide a unified logging layer, we'll use [Fluentd](http://www.fluentd.org/) to route our application's STDOUT to ELK.


## The ELK stack

ref: [ELK Docker image documentation](https://elk-docker.readthedocs.org/)

I really like Docker for its ability to host many configurations of many pieces of infrastructure without cluttering up my mac and having to deal with version management manually.

Using the current Docker Toolbox (replaces boot2docker with docker-machine), create a new VM.

```bash
$ docker-machine create --driver virtualbox elk
Running pre-create checks...
(elk) Default Boot2Docker ISO is out-of-date, downloading the latest release...
(elk) Latest release for github.com/boot2docker/boot2docker is v1.10.3
(elk) Downloading /Users/starver/.docker/machine/cache/boot2docker.iso from https://github.com/boot2docker/boot2docker/releases/download/v1.10.3/boot2docker.iso...
(elk) 0%....10%....20%....30%....40%....50%....60%....70%....80%....90%....100%
Creating machine...
(elk) Copying /Users/starver/.docker/machine/cache/boot2docker.iso to /Users/starver/.docker/machine/machines/elk/boot2docker.iso...
(elk) Creating VirtualBox VM...
(elk) Creating SSH key...
(elk) Starting the VM...
(elk) Waiting for an IP...
Waiting for machine to be running, this may take a few minutes...
Machine is running, waiting for SSH to be available...
Detecting operating system of created instance...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect Docker to this machine, run: docker-machine env elk
```

Start the docker elk VM, start a shell on that VM, and pull the ELK stack docker image

```bash
$ docker-machine env elk
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/starver/.docker/machine/machines/elk"
export DOCKER_MACHINE_NAME="elk"
# Run this command to configure your shell: 
# eval $(docker-machine env elk)
$
$ eval $(docker-machine env elk)
$ docker pull sebp/elk
...
Digest: sha256:670f7f3fbd21de2ee53a53acf2d4615baf51c6ff046e255f8e3b07a8b64e44a5
Status: Downloaded newer image for sebp/elk:latest
```

You can test everything is working by browsing `http://192.168.99.100:5601`, but there aren't any log entries 

Create a place to store ElasticSearch logs `sudo mkdir /var/lib/elasticsearch`

Create a docker volume around that directory 

```
$ audo docker run -p 5601:5601 -p 9200:9200 -p 5000:5000 -v /var/lib/elasticsearch --name elk_data sebp/elk
```

Now you can start the elk stack with

```
sudo docker run -p 5601:5601 -p 9200:9200 -p 5000:5000 --volumes-from elk_data --name elk sebp/elk
```


## Fluentd

Fluentd has an official [docker image](https://hub.docker.com/r/fluent/fluentd/)

```bash
$ docker-machine create --driver virtualbox fluentd
...
$ docker-machine env fluentd
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/starver/.docker/machine/machines/fluentd"
export DOCKER_MACHINE_NAME="fluentd"
# Run this command to configure your shell: 
# eval $(docker-machine env fluentd)
$ docker pull fluent/fluentd
```



# TODO

setup fluentd in the docker container:http://www.fluentd.org/guides/recipes/docker-logging

connect spring docker image to the elk docker image







## Masking sensitive information

