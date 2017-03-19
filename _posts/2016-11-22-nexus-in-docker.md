---
layout: post
title:  "Nexus in Docker as a Docker Registry"
date:   2016-11-22 14:35:56
tags:   nexus docker docker-registry ci cd haproxy

---

**TL;DR** Deploy Nexus in a Docker container and provide Docker Registry, Maven, NuGet, npm, ruby, etc., repos

<img style="float: left;" height="50" width="283" src="/images/Nexus.jpeg">

I've been a fan of Nexus for a decade, since I converted our maven repo on an NFS share accessed via ssh to a 1.x hosted proxy. It was an easy 10x increase in build and deploy times - yes I timed it. Nexus 3 is a huge advance for cloud shops: adding Docker, npm, Ruby, Bower, PyPi, NuGet, and static site repos to the original maven repositories.

I recently had the wonderful opportunity to stand up yet another CI/CD pipeline, allowing me to address pain points from previous implementations. It's kinda like cleaning out the back closet, belly button lint, pick your metaphor.

What pain points?

* **Disparate repos**: We have had multiple repositories with varying levels of sophistication. This required infrequent, yet regular, maintenance chores and not everyone had the knowledge to do it. I want a system that is easy to maintain and run books to describe the processes.
* **Upgrades**: Nexus has a pretty good upgrade story, but upgrading major versions always involves some work. I want anyone to be able to keep the service current and if there is a flaw, rollback without impact to the team.
* **Backups**: If the Nexus server has a disk failure, the team loses all builds and cannot continue development work until a new Nexus is deployed. I want a backup of all artifacts and configuration and a quick way to deploy a new Nexus server.
* **HTTPS**: Previous implementations have used HTTP communications because of the difficulty getting corporate to purchase an SSL certificate. Docker, as well as other thought leaders, expect secure communications, and it is downright painful to use a Docker registry without TLS. Additionally, TLS makes hacker lateral movement within a compromised system more difficult.
* **Authentication**: Previously we had small teams that either had Nexus based users and the required user management, or wide-open admin access. I need to audit builds for compliance and easily add and remove access on staff rotation.

## Strategies

### Consolidated repos

Nexus provides this out of the box. We just have to update existing build and deploy scripts to point to the new repo and age out the old repositories according to compliance concerns.

### Upgrades

Nexus can now be deployed as a Docker container. We thrive on Docker containers. This means that when using a Nexus Docker deployment, everyone already knows how to monitor and maintain the container, they just need configuration specifics.

Upgrades are pretty easy: turn down the running container, stand up a new one, test it, move on. If tests fail, turn down the new container, turn up the old one. If data was corrupted, the backup strategy provides simple recovery.

### Backups

In the process of moving to a Docker deployment, Nexus has consolidated all data and configuration into a single directory. Since we live in the cloud, we will use our cloud provider's backup service to create a backup policy for a single directory.

### HTTPS

We will use a 10 year self-signed CA and wildcard certificate that you can generate from a previous article [Self-signed CA and cert](http://stevetarver.github.io/openssl/ca/certificate/jenkins/Nexus/2016/10/11/self-signed-ca-cert.html).

This avoids corporate difficulties getting the cert, and cert replacement after the standard two year term.

### Authentication

Most corporations already have some kind of LDAP implementation, as we do. Nexus provides LDAP integration so we just need to find the right corporate resource to stand up the LDAP facilities for us to connect to.


## Deployment

We start with a Ubuntu 16 base because it offers no-reboot kernel upgrades and is a favorite flavor for our SREs. The POC starts with 2 CPUs, 4GB RAM, and a 100GB `/data` partition. Since its a VM, we can upgrade any of these resources with a trivial outage.

Next, we use Ansible playbooks to harden the OS and install Docker - the ones I used are in my GitHub repo, as examples.

Then, a little bit of config magic. The [Docker Hub Nexus 3 image](https://hub.Docker.com/r/sonatype/Nexus3/) is a bit hard to find; they haven't marked it as official, but it provides enough information to get started. 

There are two choices to make

* where to store data
* what ports to expose

On the data front, we want a simple path for backups, one that we can grow as the need presents itself, and the ability to restore to a new server on disk failure. This means that we don't want a Docker volume, we want a bound mount point. For Nexus to properly manage its external data directory, it exports the nexus UID 200. 

```
mkdir /data/nexus-data
chown -R 200 /data/nexus-data
```

Nexus exposes a single web app port, so selecting ports is simple until you hit the Docker Registry use case. 

Good engineering practices suggest that we version releases sent to production and that image is immutable; no one can overwrite a versioned release with a modified version. We also need to keep images for audit for some period of time to adhere to our compliance concerns.

On the other hand, we only need the latest build for daily CI/CD to lower level platforms, like dev and qa; only one version and no retention policy.

We will need separate Nexus repositories to implement the separate redeploy and retention policies, and we will have to use separate ports to distinguish them. 

What happens when another team wants to use our Nexus repo? We could just identify the owner by the group part of the image name. We could also create separate storage and ports for each group to more easily identify groups associated with discovered problems. That choice will likely depend on client count and activity, but, there are potentially 2 ports per team.

That covers build use cases, but what about deploying the images? We would like a single port to pull any image, instead of each deploy script having to know what port each release or latest image is stored in. We need a separate port for pulls.

We will start with a single team requiring a pull port, a release push port and a latest push port. If another team wants to use our registry, we can tear down the Nexus container and recreate it with two additional ports; one for release, one for latest.

Docker run arguments - for those not already familiar:

- `--name`: the name of this container used in future Docker commands and referenced in run books.
- `-p`: expose port 8081 on the IPV4 host interface. Otherwise, Docker may start on IPV6 only. Also expose our Docker registry ports.
- `--restart`: always restart the container when Docker starts, server reboots, etc.
- `-v`: map the container data directory to a host directory. This allows simple upgrades without losing data and backups.
- `-d`: run the named image detached (in the background).

```
docker run --name nexus \
    -p 10.98.100.208:8081:8081 \
    -p 10.98.100.208:18000:18000 \
    -p 10.98.100.208:18001:18001 \
    -p 10.98.100.208:18002:18002 \
    --restart=always \
    -v /data/nexus-data:/nexus-data \
    -d sonatype/nexus3:3.0.1
```

Notes:

* Initialization takes 2-3 minutes. Use `Docker logs -f Nexus` to follow that progress.
* You can prove Nexus is up with `curl -u admin:admin123 http://10.98.100.208:8081/service/metrics/ping`
* Since `run` creates and starts the container, you only use it once with the image. Afterwards, use `docker stop nexus` to stop the container and  `docker start nexus` to start it back up.

At this point, if you don't have a wildcard or server certificate, generate one using this article: [Self-signed CA and cert](http://stevetarver.github.io/openssl/ca/certificate/jenkins/Nexus/2016/10/11/self-signed-ca-cert.html).

Then create a DNS entry in your DNS provider that matches the certificate; we will use that in SSL off-loading below.


## SSL off-loading

We could let Nexus handle SSL certificates but it turns out to be just too [tedious](https://support.sonatype.com/hc/en-us/articles/217542177) and we would have to repeat those steps on every upgrade. Instead, we will separate out that concern and install a reverse proxy: HAProxy.

On the Nexus server, install HAProxy. In Ubuntu 16.04, getting a recent version is as simple as `apt-get install haproxy`.

We will map external facing 16000 ports to interal facing 18000 ports; in HAProxy we listen on 1600* and in Nexus, we use corresponding 1800* ports.

Copy your server certificate pem file into `/etc/haproxy/certs` and set up the front/backend, via haproxy.cfg, like:

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

listen docker-all-group
    bind *:16000 ssl crt /etc/haproxy/certs/splat.internal.mdg.com.pem no-sslv3
    mode http
    server UC1MDGNEXUS01 10.98.100.208:18000

listen platform-docker-release
    bind *:16001 ssl crt /etc/haproxy/certs/splat.internal.mdg.com.pem no-sslv3
    mode http
    server UC1MDGNEXUS01 10.98.100.208:18001

listen platform-docker-latest
    bind *:16002 ssl crt /etc/haproxy/certs/splat.internal.mdg.com.pem no-sslv3
    mode http
    server UC1MDGNEXUS01 10.98.100.208:18002

# force https use through http redirect
frontend http-in
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

frontend nexus
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
    server Nexus 10.98.100.208:8081 
```

The above config cleans up the Nexus GUI urls by binding to ports 443 and redirecting port 80 access to 443. It then listens and forwards ports 16000-16002 to the Nexus http ports 18000-18002.

You can add the following to the bottom of `/etc/services` to document what we've done - depends on if you think this adds clarity or obfuscates. After this modification, `netstat -a` will show the names instead of the ports.

```
# Local services
fe-docker-all-group          16000/tcp
fe-docker-platform-release   16001/tcp
fe-docker-platform-latest    16002/tcp
be-docker-all-group          18000/tcp
be-platform-docker-release   18001/tcp
be-platform-docker-latest    18002/tcp
```

Use `netstat -a` to ensure all ports are listed. If there is a port collision hidden because netstat displays a name instead of port, you can see port to name mappings in `cat /etc/services`.

Verify the HAProxy config file with `haproxy -c -f haproxy.cfg`, then restart HAProxy with `service haproxy restart`.

## Nexus configuration

I took a look at the state of the Nexus 3 ReST API and found it so poorly documented that it is unusable. They have tried to simplify orchestrating several ReST APIs by wrapping them in groovy scripts for things like create blob store, create registry. If it were simple ReST calls, I would have spent an hour to automate the initial configuration. The unpleasant prospect of digging through their jars to see what they are doing and then having mixed results at the end tells me: Do it manually - it only takes 5 minutes.

### Create blobstore and repos

ref: [Nexus Book](https://books.sonatype.com/Nexus-book/3.0/reference/Docker.html)

In the Server administration and configuration page (click toolbar gear icon)

#### Create blobstores

This step prepares the disk to store Docker images. I am creating a store to cache Docker Hub images, and one to store images for our first team: platform.

_**NOTE**: you cannot delete blobstores once created_

1. Click on the Blobstores item in the left sidebar
2. Click on create blobstore
3. Set the name = docker-hub
4. Repeat for docker-platform
4. Save

#### Create team repos

We want a separate repo for latest and versioned tags, much like maven's SNAPSHOTS and RELEASES, for the same reasons. We don't want to allow accidentally overwriting a released image due to a misconfiguration, and we really only need the latest version of incremental builds.

Also note that we are only using HTTP in Nexus - HAProxy is providing SSL off-loading.

Create the docker-platform-release repo

1. Click on Repositories.
2. Click on the Create Repository button
3. Click on Docker (hosted)
4. Set name = docker-platform-release
5. Check the Docker Connectors HTTP option and enter 18001
6. Enable Docker V1 API: checked
7. Select Storage: Blobstore: docker-platform
8. Set Deployment policy = Disable redeploy
8. Click the Create repository button

Repeat for docker-platform-latest with

1. Name = docker-platform-latest
2. Docker Connector = HTTP: 18002
3. Blobstore = default
4. Deployment policy = Allow redeploy

### Create proxy repo for DockerHub

2. Click on the Create Repository button
3. Click on Docker (proxy)
4. Set name = docker-hub
6. Enable Docker V1 API: checked
5. Set Remote storage = `https://registry-1.docker.io`
6. Select Docker Index = Use Docker Hub
6. Set Blobstore = docker-hub
8. Click the Create repository button

### Create read-only group for all Docker registries

2. Click on the Create Repository button
3. Click on Docker (group)
4. Set name = docker-all
5. Set Docker Connectors = HTTP: 18000
6. Set Blobstore = default
7. Add all Docker repos to this group with docker-platform-release first, latest second, and docker-hub last
8. Click the Create repository button


### Setup common tasks

Create these tasks

1. In the Administration facility, click on System->Tasks
2. Click the Create task button
3. Select Compact blob store
4. Task name: Compact platform blob store
5. Blob store: default
6. Task frequency: Weekly
7. Time to run this task: 02:30
8. Days to run this task: Sunday
9. Click the Create task button

Repeat for the docker-hub and docker-platform blob store.

## Test Docker registry

Using Docker for Mac/Windows really simplifies local Docker use; you don't have to connect to a Docker machine, etc. If you are a new Docker user, this will look just like the unix commands.

If you are using a self-signed CA certificate, and have not already installed it on your laptop, do that now and restart the Docker service to pick up the new cert.

I am using a Docker projects in this example, but you could just as easily pull any image from Docker Hub and re-tag it to point to our new Nexus server: nexus.internal.mdg.com.

```
$ mvn clean install
# ...

# tag this build to point to the Nexus platform team's latest registry
$ docker build -t nexus.internal.mdg.com:16002/com.mdg.internal/canary .
# ...

$ docker images
REPOSITORY                                              TAG                 IMAGE ID            CREATED             SIZE
nexus.internal.mdg.com:16002/com.mdg.internal/canary    latest              44ca46e13c36        27 seconds ago      198 MB

# log in to our repo
$ docker login nexus.internal.mdg.com:16002
Username: admin
Password: 
Login Succeeded

# now push it
$ docker push nexus.internal.mdg.com:16002/com.mdg.internal/canary 
The push refers to a repository [nexus.internal.mdg.com:16002/com.mdg.internal/canary]
f5cb5bda55c5: Pushed 
9639aaf1419f: Pushed 
162f935b1198: Pushed 
dcf909146faa: Pushed 
23b9c7b43573: Pushed 
latest: digest: sha256:87469ab2532f54326e204e21fa6926dbc2dc86983bba658e563866dcf4f5fdc0 size: 1375
```

Now you can view the image in Nexus 3 by clicking the "Components" item in the left sidebar, then the docker-platform-latest repository.

If we delete the local image, we can test pull

```
$ docker images
REPOSITORY                                             TAG                 IMAGE ID            CREATED             SIZE
nexus.internal.mdg.com:16002/com.mdg.internal/canary   latest              c8f3b1b5b9e1        23 hours ago        198.8 MB
frolvlad/alpine-oraclejdk8                             slim                7471ceae9b37        43 hours ago        167.4 MB

$ docker rmi nexus.internal.mdg.com:16002/com.mdg.internal/canary 
Untagged: Nexus.internal.mdg.com:16002/com.mdg.internal/canary:latest
Deleted: sha256:c8f3b1b5b9e18bb1708f9e6e9c952ba45488b97edcf9e1519a10713d9d056b2c
Deleted: sha256:7eed6fd7906739f3fdfa3f07397718eafb3121cae94e913b7932f90718d02bc6
Deleted: sha256:43937def6014db8ac28432d16fe2d2f603a8c4f2d54cc47eecf1620019049419
Deleted: sha256:e2482b0476c7bdce961d925e2cb2ab7f91dec519a8b397bfc8b743552a73c7dd
Deleted: sha256:d99ef992071037765ef816259768d6e9673aa87774ecad8f0ffab075c943b752

$ docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
frolvlad/alpine-oraclejdk8   slim                7471ceae9b37        43 hours ago        167.4 MB

$ docker pull nexus.internal.mdg.com:16002/com.mdg.internal/canary
Using default tag: latest
latest: Pulling from nexus.internal.mdg.com:16002/com.mdg.internal/canary
4d06f2521e4f: Already exists 
f6c060698554: Already exists 
04ce03671524: Already exists 
b2c6a680320f: Pull complete 
9e7e6565022c: Pull complete 
Digest: sha256:88dfc2490c364b22212b148b93e6762b3a20045c2d26c562eb4388a89462a418
Status: Downloaded newer image for nexus.internal.mdg.com:16002/com.mdg.internal/canary:latest

$ docker images
REPOSITORY                                             TAG                 IMAGE ID            CREATED             SIZE
nexus.internal.mdg.com:16002/com.mdg.internal/canary   latest              c8f3b1b5b9e1        23 hours ago        198.8 MB
frolvlad/alpine-oraclejdk8                             slim                7471ceae9b37        43 hours ago        167.4 MB
```

## Backups

Backups and restores are provided as a service by your cloud provider and the process will vary. In general, you will create a backup policy for `/data/Nexus-data`. See your cloud provider's knowledge base to develop run books for restoration after a disk failure.

## LDAP integration

Nexus provides LDAP integration as a first class citizen. You will have to decide how to organize groups and membership, but once that is complete, you just fill in a dialog and LDAP integration is complete.


## Operations audits

We need to track repository modification for forensic purposes. You can set up a log forwarder to your log aggregation facility and build dashboards and alerts there.


## Maintenance

What kind of maintenance can we expect; what will initial run books need to document?

### Disk use high

Indicated by monitoring agent alert.

Check your repository retention policies. Ensure that you trim releases to an acceptable retention period and latest images to only one. Trigger the cleanup to see the effects. 

If no problems found there, you will need to grow the `/data/Nexus-data` partition.

### Upgrade

Nexus releases Docker image updates every few months. You should upgrade often, remaining a version behind current to let others identify the bugs. Upgrade failures are in proportion to distance between versions.

Remember that you should always run a versioned release so that you can rollback to that versioned release.

When an upgrade is available, the basic process is:

1. Stop and delete the old container
1. Run a new container from the new versioned image
1. If tests fail, stop the new, and run the old version

```shell
$ docker stop nexus
# ...
$ docker rm nexus
# ...

$ docker run --name nexus \
    -p 10.98.100.208:8081:8081 \
    -p 10.98.100.208:18000:18000 \
    -p 10.98.100.208:18001:18001 \
    -p 10.98.100.208:18002:18002 \
    --restart=always \
    -v /data/nexus-data:/nexus-data \
    -d sonatype/nexus3:3.1.0
# ...

# Test, and if failure
$ docker stop nexus
# ...
$ docker rm nexus
# ...
$ docker run --name nexus \
    -p 10.98.100.208:8081:8081 \
    -p 10.98.100.208:18000:18000 \
    -p 10.98.100.208:18001:18001 \
    -p 10.98.100.208:18002:18002 \
    --restart=always \
    -v /data/nexus-data:/nexus-data \
    -d sonatype/nexus3:3.0.1
# ...

```

### Disk failure

On Nexus disk failure, recover from the last backup taken. Follow the procedure to create a new Nexus server and stop at the Docker run step. Use your cloud provider to restore the last backup to `/data/nexus-data`. Then execute the Docker run command.

### Unknown

There are several tools for remediating new errors. Besides the facilities in the Nexus webapp:

Start a bash shell in the Nexus container as the nexus user

```Docker exec -it --user nexus nexus bash```

Start a bash shell in the Nexus container as the nexus user

```Docker exec -it --user root nexus bash```

View Docker logs

```
docker logs -f nexus
```

## Epilog

There you have it. A fully functional Docker Registry with the convenience of a Nexus webapp to admister it and the bonus of several other repository types with a common front end. Hope this helps fast-track your adventures.
