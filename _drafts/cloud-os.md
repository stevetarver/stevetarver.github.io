# Cloud OS

Question:
* What OS should we be using
* How do we monitor the OS (can we install check_mk on it)
* How do we provision it
* How do we patch it
* Can we manipulate it with ansible/salt
* How well does it integrate with docker
* Is it mature enough to bet our product on
* How much do we have to learn

Benefits
* Reduced provision time
* Reduced attack surface
* Reduced OS patching (needs analysis, actual patching)

Why Snappy
* All the benefits/reasons we originally selected ubuntu, we know how to provision, patch, monitor, orchestrate it, no learning curve, minimal orchestration/code changes.
* It is one of the docker minimal recommended host os
* transactional or image-based systems management (updates/patching). guaranteed to succeed every time, never left in an incomplete state.
* everything is snappy (like rancheros) and it replaces the package manager
* Applications are pretty strictly limited, can't touch each other or poke around the system. Smallest and most secure ubuntu ever.
* There is a docker package you install with snappy
* Patch at your convenience: download the new version and let the customer trigger the reboot

Concerns
* Snappy is not officially supported until 1 year after major version released (ubuntu 15.04).
* Snappy pm is new - any broken stuff?
* Are any packages that we need missing?
* Smaller dedicated user community, problems found/solved in forums

Unanswered Questions
* Can we put Percona in a snappy app? 

Recommendations
* Of the OS tested, Snappy is the most promising OS choice for customer servers, and warrants further investigation.

Next Steps
* Provision a one-off Snappy, install Percona, include it in our testing, get some run time


Actions
* Change our build system to use the new OS
* Submit a platform feature request to add our choice
* What role does kubernetes play, why do we or don't we use it. Create a para for Christine/Wayne to talk to other products.



__________________________________

Docker OS's

References
* https://blog.inovex.de/docker-a-comparison-of-minimalistic-operating-systems/
* https://blog.docker.com/2015/02/the-new-minimalist-operating-systems/


Rancher
* tiny: 20MB
* OS is a docker container, that only runs docker containers
* docker semantics for OS updates

Core OS
* service discover: etcd automatically notices new dockers and adds them to the cluster
* fleet service and kubernetes (cluster mgmt)
* automatic kernel rolling updates throughout the cluster

Ubuntu Core (Snappy)
* signed, transactional delta based updates with rollback

RHEL Atomic Host (includes CentOS, Fedora, Red Hat)
* atomic updates and rollbacks
* SELinux
* Kubernetes & Flannel (easy clustering)
* cockpit gui for managing containers among multiple hosts

VMWare Photon


Mesophere DCOS



---------------------------------------------------------------------

RancherOS

-> Run in local vagrant, install/run percona

Install VirtualBox, Vagrant, Docker

git clone https://github.com/rancher/os-vagrant.git
vagrant up
vagrant ssh
docker ps
docker pull percona
docker run --name percona -e MYSQL_ROOT_PASSWORD=foo -d percona:latest
docker ps
docker exec -it <percona inst id> bash
mysql -u root -p -e 'show database;'

see Vagrantfile changes that automatically load percona

----------------------------------------------------------------------

Snappy (Ubuntu Core)

References
* https://developer.ubuntu.com/en/snappy/start/using-snappy/

Why
* transactional or image-based systems management (updates/patching). guaranteed to succeed every time, never left in an incomplete state.
* minimal server image, same as ubuntu
* everything is snappy (like rancheros) and it replaces the package manager
* Applications are pretty strictly limited, can't touch each other or poke around the system. Smallest and most secure ubuntu ever.
* There is a docker package you install with snappy
* Patch at your convenience: download the new version and let the customer trigger the reboot

Cons
* playbooks 

Demo
* build a snappy 

* vagrant box add snappy http://goo.gl/6eAAoX
* vagrant init snappy
* vagrant ssh
* 


To Prove
Is OS mature enough for production
How to provision - use CLC
Can we alter the BOSH base image - use apt-get which integrates with ubuntu
Can we get a manifest of BOSH OS contents - use dpkg

How frequently do they update images - slacked #platform
How do we patch an existing BOSH - slacked #platform

Will it work with ansible/salt - sure

How does it integrate with docker?
* Any special features or caveats?

How check-mk monitor mysql?
How to handle out of date packages on a BOSH image? Only update if needed for security or feature

Benefits
Provisions quickly - time it 
How often is our stemcell updated 


Questions
How do we update the OS in case of security patch
Can we independently update a package - security vulnerability and we want to patch it first


Actions 

1. Provision using our current MySQL deploy
2. How do we monitor mysql, research - pull the default percona, 


Installing Docker

https://docs.docker.com/engine/installation/ubuntulinux/
prior to apt-get update:
* apt-get install apt-transport-https

docker pull percona
See https://hub.docker.com/_/percona/ for more instructions

Start
docker run --name bosh-percona -e MYSQL_ROOT_PASSWORD=my-secret-pw -d percona:latest
