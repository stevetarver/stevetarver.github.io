---
layout: post
title:  "ReST: Spring Boot starter (part 2)"
date:   2016-05-12 13:25:16
tags: rest spring-boot java lombok spock jdbc 12-factor

---



## Dockerizing

### Create a dedicated VM

We'll use a dedicated docker VM for building and testing.

`docker-machine create -d virtualbox build`

### Building the Docker image

Connect to the Docker `build` image in the terminal you will be building in.

`eval "$(docker-machine env build)"`

Show processes to prove you connected cleanly.

`docker ps`

Build the Docker image

`mvn package docker:build`

On success, you can see the new image with `docker images` as `stevetarver/rest-start-spring-boot`

One would normally push to a public repo with

`docker push stevetarver/rest-start-spring-boot`

### Running the docker image locally

Run the newly built image with

`docker run -p 8080:8080 -t stevetarver/rest-start-spring-boot`

You will see the familiar Spring Boot log entries and the app announce on port 8080 - but there will likely not be anything serving `http://localhost:8080`. Each Docker VM will have it's own IP; we need to find that.

```
$ docker-machine env build
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.101:2376"
export DOCKER_CERT_PATH="/Users/starver/.docker/machine/machines/build"
export DOCKER_MACHINE_NAME="build"
# Run this command to configure your shell: 
# eval $(docker-machine env build)
```
Given the information above, you can visit

```
http://192.168.99.101:8080/health
http://192.168.99.101:8080/metrics
```

Of course, visiting `health` will tell you the app is down: the execution environment was not initialized. We need to connect the rest docker to a MySQL docker.

You can tail the logs from any terminal that is connected to the `build` docker VM.

```
$ docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                    NAMES
a5660b9ddd3d        stevetarver/rest-start-spring-boot   "java -Djava.security"   22 minutes ago      Up 22 minutes       0.0.0.0:8080->8080/tcp   sleepy_poitras
[starver@bhooteshwara ~ 21:32:07]$ docker logs -f a566
```


### Stopping the container

If you `CTRL-C` in the terminal that you started the app in, logging will stop but the image is still running as shown in

```
$ docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                    NAMES
a5660b9ddd3d        stevetarver/rest-start-spring-boot   "java -Djava.security"   8 minutes ago       Up 8 minutes        0.0.0.0:8080->8080/tcp   sleepy_poitras
```

Stop the image with 

`docker stop a56`

### Connecting the docker image to MySQL

ref: [MySQL on DockerHub](https://hub.docker.com/_/mysql/)

So, let's setup a separate docker vm to host MySQL. We could add the mysql image to the existing `build` vm but I want to keep these separate so I can use the MySQL docker VM for other projects.

`docker-machine create -d virtualbox mysql`

Connect the terminal to the new vm and prove it was successful with `docker ps`

```
$ docker-machine env mysql
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.104:2376"
export DOCKER_CERT_PATH="/Users/starver/.docker/machine/machines/mysql"
export DOCKER_MACHINE_NAME="mysql"
# Run this command to configure your shell: 
# eval $(docker-machine env mysql)
[starver@bhooteshwara ~ 23:07:53]$ eval $(docker-machine env mysql)
[starver@bhooteshwara ~ 23:08:24]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

Create a data directory to store persistent MySQL data

```
$ mkdir -p ~/data/mysql-rest-start-spring-boot
```

Install the mysql docker image with `docker pull mysql` and then run the image and link it to the persistent data directory.

```
$ docker run --name mysql -v ~/data/mysql-rest-start-spring-boot -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
aae976c3cb00402c574778676ae66b17ac4b99a8ed27a43f042c25f8f119eb6c
```

Let's prove that we can connect to the command line client

```
$ docker run -it --link mysql:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.11 MySQL Community Server (GPL)

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
| sys                |
+--------------------+
4 rows in set (0.00 sec)
```
Now, create a MySQL WorkBench connection to the database (because it will be convenient to use later). Using the information above, this will use host 192.168.99.104 and the standard port 3306 with account `root/root`.

Paste the sql script from my `sample-data` gitHub repo into the MySQL WorkBench query.



Stop the container

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
a1aa4e391e8c        mysql:latest        "/entrypoint.sh mysql"   23 minutes ago      Up 23 minutes       3306/tcp            mysql
$ docker stop a1
```


Connect to the docker image with the MySQL WorkBench

Paste the sql script from my `sample-data` gitHub repo into the MySQL WorkBench query.


Create a place to store the data


Initialize the database


delete the application.properties file

Now we can start the rest service with

```
docker run -p 8080:8080 \
-e spring.datasource.url=jdbc:mysql://localhost/test?autoReconnect=true \
-e spring.datasource.username=root \
-e spring.datasource.password= \
-e spring.datasource.driver-class-name=com.mysql.jdbc.Driver \
-t io.ctl.rdbs/ds-canary
```


# Externalized configuration


# Access events provide call duration, count, status
and more

https://github.com/logstash/logstash-logback-encoder#accessevent-fields




## Metrics: are metrics exposed for systems like Prometheus and are they extensible?

What I had hoped for in Spring Boot's `/metrics` was to capture ReST API call counts, duration, and response status - keys for fundamental charting. It gets about half way there, but call counts are only by URI, we can't tell which HTTP verb was applied. Also, the gauge.response metrics only show the last call duration, not an average, min, or max.

The system is extensible with hooks into the CounterService, GaugeService, and even posting your own public metrics. I wonder if anyone has built an AOP that would collect these items by RestController class annotation.

```json
{
  "mem": 493664,
  "mem.free": 390587,
  "processors": 8,
  "instance.uptime": 2972521,
  "uptime": 2975873,
  "systemload.average": 1.791015625,
  "heap.committed": 439296,
  "heap.init": 262144,
  "heap.used": 48708,
  "heap": 3728384,
  "nonheap.committed": 55616,
  "nonheap.init": 2496,
  "nonheap.used": 54368,
  "nonheap": 0,
  "threads.peak": 19,
  "threads.daemon": 17,
  "threads.totalStarted": 23,
  "threads": 19,
  "classes": 6510,
  "classes.loaded": 6511,
  "classes.unloaded": 1,
  "gc.ps_scavenge.count": 5,
  "gc.ps_scavenge.time": 48,
  "gc.ps_marksweep.count": 2,
  "gc.ps_marksweep.time": 106,
  "httpsessions.max": -1,
  "httpsessions.active": 0,
  "datasource.primary.active": 0,
  "datasource.primary.usage": 0.0,
  "gauge.response.contacts.id": 7.0,
  "gauge.response.metrics": 10.0,
  "gauge.response.contacts": 75.0,
  "counter.status.200.metrics": 1,
  "counter.status.400.contacts.id": 2,
  "counter.status.200.contacts": 1,
  "counter.status.200.contacts.id": 1
}
```

## Health: do health metrics exist and are they extensible

## Security


## API doc: is there a suitable API doc generation system like Swagger

## Dockerize

## Testing

Testing is with Spock because it approaches the expressiveness of modern test frameworks in other languages. Many things are appropriate for Groovy in a highly structured Java environment. Lombok removes the boiler plate but other language conveniences simplify and clarify.
