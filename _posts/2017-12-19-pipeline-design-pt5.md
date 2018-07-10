---
layout: post
title:  "Pipeline Design Pt 5: A local Jenkins pipeline"
date:   2017-12-19 11:11:11
tags:   ci cd pipeline jenkins automation

---

**TL;DR** Deploy our WiP Jenkins pipeline to [Minikube](https://github.com/kubernetes/minikube), and, actually use it.

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/jenkins.png">

Over the last few months, we've developed some notions around a build and deploy Jenkins pipeline and implemented parts of it as stub code. Now it is time to prove that system with an actual deployment. 

This article has a lot of application setup and configuration, but along the way, we'll create our own GitHub based Helm chart repository and see how to work with ServiceAccounts from the command line inside a pod.

If Kubernetes is in your future, Minikube should be in your present. It is a great Kubernetes playground, providing most of the features, with sensible defaults, so we can have k8s up and running in ~ 10 minutes - no kidding, just timed a fresh install and the majority of the time is waiting on downloads.

## Install Minikube

A fresh Minikube is trivial to install:

```
ᐅ brew cask install minikube
```
**NOTE**: Minikube will install kubectl (the Kubernetes command line client) if it is not already installed.

Minikube can have problems with version upgrades - if you have problems, the simplest solution is to nuke & pave:

```
ᐅ minikube delete
ᐅ brew cask uninstall minikube
ᐅ sudo rm -rf ~/.minikube
ᐅ brew cask install minikube
```

If you will use Minikube for more than following this article, I suggest installing/using `hyperkit` VM driver for Mac, or `kvm2` for linux. Minikube uses VirtualBox by default; hyperkit starts much more quickly and uses 25% less CPU. You can see all VM driver options and installation instructions [here](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md). If I convinced you, once hyperkit is installed, set the default driver in Minikube config:

```
ᐅ minikube config set vm-driver hyperkit
```

If you need more complete install instructions, you can find them in the [Minikube README](https://github.com/kubernetes/minikube). The install transcript for Minikube, kubectl, and hyperkit looks like:

```
ᐅ brew cask install minikube
==> Satisfying dependencies
==> Installing Formula dependencies: kubernetes-cli
==> Installing kubernetes-cli
==> Downloading https://homebrew.bintray.com/bottles/kubernetes-cli-1.11.0.high_sierra.bottle.tar.gz
######################################################################## 100.0%
==> Pouring kubernetes-cli-1.11.0.high_sierra.bottle.tar.gz
==> Caveats
Bash completion has been installed to:
  /usr/local/etc/bash_completion.d

zsh completions have been installed to:
  /usr/local/share/zsh/site-functions
==> Summary
🍺  /usr/local/Cellar/kubernetes-cli/1.11.0: 196 files, 53.7MB
==> Downloading https://storage.googleapis.com/minikube/releases/v0.28.0/minikube-darwin-amd64
######################################################################## 100.0%
==> Verifying checksum for Cask minikube
==> Installing Cask minikube
==> Linking Binary 'minikube-darwin-amd64' to '/usr/local/bin/minikube'.
🍺  minikube was successfully installed!

# Install hypekit
ᐅ curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-hyperkit \
&& chmod +x docker-machine-driver-hyperkit \
&& sudo mv docker-machine-driver-hyperkit /usr/local/bin/ \
&& sudo chown root:wheel /usr/local/bin/docker-machine-driver-hyperkit \
&& sudo chmod u+s /usr/local/bin/docker-machine-driver-hyperkit
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 25.5M  100 25.5M    0     0  3993k      0  0:00:06  0:00:06 --:--:-- 4018k
ᐅ minikube config set vm-driver hyperkit
These changes will take effect upon a minikube delete and then a minikube start
```

Now we can start minikube:

```
ᐅ minikube start
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
ᐅ minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.64.9
ᐅ kubectl version
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.0", GitCommit:"91e7b4fd31fcd3d5f436da26c980becec37ceefe", GitTreeState:"clean", BuildDate:"2018-06-27T22:29:25Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-03-26T16:44:10Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
```

Voila! Instant Kubernetes cluster!. But Minikube is more; it can also be thought of as a Kubernetes version management system like `pyenv`, `sdkman` or `rbenv`. Each minikube will show you its supported versions:

```
ᐅ minikube get-k8s-versions
The following Kubernetes versions are available when using the localkube bootstrapper:
	- v1.10.0
	- v1.9.4
	- v1.9.0
# ...
```

and you can start any of those versions with the start command:

```
ᐅ minikube delete
ᐅ minikube start --kubernetes-version v1.9.4
```

## Install Helm

We deploy all Kubernetes services with [Helm](https://github.com/kubernetes/helm) charts. Helm is implmented in two parts: a command line client and a Kubernetes deployed server component named Tiller. Helm uses your `~/.kube/config` to identify the target Kubernetes cluster to interact with. Now that your Minikube cluster is running and kubectl is pointing at it we can install both helm and tiller:

```
ᐅ brew install kubernetes-helm
# Install Tiller in the k8s cluster ~/.kube/config is pointing at
ᐅ helm init
```

which will look like:

```
ᐅ brew install kubernetes-helm
==> Downloading https://homebrew.bintray.com/bottles/kubernetes-helm-2.9.1.high_sierra.bottle.tar.gz
######################################################################## 100.0%
==> Pouring kubernetes-helm-2.9.1.high_sierra.bottle.tar.gz
==> Caveats
Bash completion has been installed to:
  /usr/local/etc/bash_completion.d
==> Summary
🍺  /usr/local/Cellar/kubernetes-helm/2.9.1: 50 files, 66.2MB
ᐅ helm init
Creating /Users/starver/.helm
Creating /Users/starver/.helm/repository
Creating /Users/starver/.helm/repository/cache
Creating /Users/starver/.helm/repository/local
Creating /Users/starver/.helm/plugins
Creating /Users/starver/.helm/starters
Creating /Users/starver/.helm/cache/archive
Creating /Users/starver/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /Users/starver/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

More install details can be found [here](https://github.com/kubernetes/helm).

## Create our own Helm Chart repo

Helm has a [default set of charts](https://github.com/kubernetes/charts) but it is also easy to add your own chart repository to hold your own tweeked charts, or charts for propietary software. 

Our initial build pipeline required developers to provide a helm chart for each repo, in that repo. There is some convenience having the chart next to the code, especially while developing the helm chart. After developing several services and charts, I see that they become repititive, and somewhat standard.

After living with that decision, I am starting to see some benefit in having a dedicated helm chart repo that hosts charts for all your services. Service charts could exist independently, or you could arrange them to provide a comprehensive chart for a cluster. This allows you to:

* easily standup a new cluster, or recover from a tragic cluster loss.
* easily make changes to all charts to accomodate infrastructure changes.
* easily compare charts to ensure consistency and best practices are met.

Perhaps a private Helm chart repo should be a standard feature.

So, let's take a brief detour and [set up a dedicated helm repo](https://github.com/kubernetes/helm/blob/master/docs/chart_repository.md). A Helm Chart repository consists of chart tarballs, an `index.yaml`, and a http server that provides bundled charts to clients. Helm makes this easy in GitHub: you can store your charts in a repo, serve the charts through gh-pages, and connect the two with some automation.

After creating the [charts](https://github.com/stevetarver/charts) repo, I seeded it with a directory structure like [kubernetes/charts](https://github.com/kubernetes/charts) and placed a Jenkins chart in `stable`.

To serve charts, I am using the gh-pages [`docs/` directory solution](https://pages.github.com/) to keep everything in the same branch. Basically, you add a `docs/` directory to your repo, visit repo -> Settings -> GitHub Pages and select "master branch /docs folder" and then click "Save". Within seconds, the site is live at [https://stevetarver.github.io/charts/](https://stevetarver.github.io/charts/).

Now to build the content I want to serve. From the charts `docs/` directory:

```
ᐅ helm package ../stable/jenkins
Successfully packaged chart and saved it to: /Users/starver/code/makara/charts/docs/jenkins-1.0.0.tgz
ᐅ helm repo index ./ --url https://stevetarver.github.io/charts
ᐅ ll
total 24
-rwxr-xr-x  1 starver  staff   384B Jul  3 11:46 index.yaml
-rw-r--r--  1 starver  staff   6.8K Jul  3 11:45 jenkins-1.0.0.tgz
```

After commit, these files are available on the [gh-pages site](https://stevetarver.github.io/charts/index.yaml). Every chart change will require recreating the tarball and the index - clearly an automation, but for a later date.

Working through this points out an obvious limit: I have a single index, how do I serve both stable and incubator charts? Perhaps a problem to solve for another day.

## Register our Helm repo

For access to our custom charts, we need to tell Helm about that repository. 

```
ᐅ helm repo add makara-stable https://stevetarver.github.io/charts
"makara-stable" has been added to your repositories
ᐅ helm repo list
NAME         	URL
stable       	https://kubernetes-charts.storage.googleapis.com
local        	http://127.0.0.1:8879/charts
makara-stable	https://stevetarver.github.io/charts
```
Success means that Helm was able to load our `index.yaml` - looking good so far. Now lets verify that we can pull the chart:

```
ᐅ helm fetch makara-stable/jenkins
ᐅ ll
# ...
-rw-r--r--  1 starver  staff   6.8K Jul  3 12:01 jenkins-1.0.0.tgz
# ...
```

**NOTE**: After each chart repo modification, we will have to let our local Helm know about those changes with: `helm repo update`.

## Use a ServiceAccount from the command line, inside a pod

Between this article's first draft and publication, I switched from an unsecured K8S 1.6 cluster to a secure 1.10 version. The first thing I learned is that minikube has a bug that prevents changing the API Server authorization mode through configuration - it is always `Node,RBAC`. This led to the second thing I learned: how to actually use a ServiceAccount from the command line. [Helm provides some good information](https://github.com/kubernetes/helm/blob/master/docs/rbac.md), as well as the official [jenkins helm chart](https://github.com/kubernetes/charts/tree/master/stable/jenkins) but I had a hard time pulling it all together - in hindsight, obvious, but up front, I spent some hours trying to wrap my head around RBAC so let's walk through that configuration.

In this POC, Jenkins uses a helm client to talk to the tiller server deployed in `kube-system`. We have isolated Jenkins in a `dev` namespace; how do we connect all the pieces to let Jenkins shell out helm commands and actually talk to tiller? 

There are three API resources involved: ServiceAccount, RoleBinding, and the Jenkins Deployment. The ServiceAccount must be defined for the Jenkins pod so we can easily mount the ServiceAccount token in the pod. This happens automatically when we include the ServiceAccount in our Jenkins chart.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: \{{ .Values.rbac.serviceAccountName }}
  labels:
    app: {{ .Release.Name }}
automountServiceAccountToken: true
```

Tiller needs to be able to get, create, and delete just about everything - very much like a `cluster-admin`. So much so, that we will create a RoleBinding to that existing role - `cluster-admin` is created in the `kube-system` namespace by default. 

What type of binding? A RoleBinding which will be scoped to a single namespace, or a ClusterRoleBinding which can span namespaces. It is probable that some deployments will modify multiple namespaces, so ClusterRoleBinding.

Next, where to deploy that role binding. Since the RoleBinding ties a Role to a ServiceAccount, both listed in the manifest, and we really only want Jenkins to use the Service account, it makes sense to deploy the RoleBinding to the `dev` namespace with our Jenkins helm chart as well.

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: {{ .Values.rbac.serviceAccountName }}-cluster-role-binding
  labels:
    app: {{ .Release.Name }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ .Values.rbac.roleRef }}
subjects:
  - kind: ServiceAccount
    name: {{ .Values.rbac.serviceAccountName }}
    namespace: {{ .Release.Namespace }}
```

For the last part of the chart modifications, we need to update the Jenkins deployment.yaml to identify the desired ServiceAccount. In the pod spec:

```yaml
      serviceAccountName: {{ .Values.rbac.serviceAccountName }}
```

Now, how to make the ServiceAccount available to the Jenkins pod helm client? Helm uses a kubectl configuration for identifying the Kubernetes cluster to talk to and user credentials. Because of the chart additions above, the tiller service account information will be mounted in the Jenkins pod at `/var/run/secrets/kubernetes.io/serviceaccount/`. 

In the Jenkins docker image, there is a kube-config and a Jenkins startup script patch that does some initial setup. We start with a bare-bones kube.config:

```
---
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    server: ~~K8S_API_SERVER~~
  name: default
contexts:
- context:
    cluster: default
    user: jenkins
  name: default
current-context: default
preferences: {}
users:
- name: jenkins
  user:
    token: ~~TILLER_SA_TOKEN~~
```
and during initial Jenkins startup, fill in the k8s api server from environment variables and the token from the mounted ServiceAccount in the Jenkins startup script patch:

```
sed -i "s/~~K8S_API_SERVER~~/https:\/\/${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT_HTTPS}/g" /etc/kubernetes/kube.config
sed -i "s/~~TILLER_SA_TOKEN~~/$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)/" /etc/kubernetes/kube.config
helm init
```

## Deploy Jenkins

At this point, we have Minikube, kubectl, and helm installed, and helm configured to pull from our custom repo. Now we need to configure k8s to support our opinionated Jenkins pipeline: label nodes and create namespaces.

We've found that having a dedicated Jenkins node (VM) makes life a lot simpler:

* no Jenkins application reschedules resulting in false build failures or missed builds
* you can scale the Jenkins node resources to accommodate build job growth without interfering with other services
* when you have to debug Jenkins, it is always at the same location

We identify the Jenkins target node with a `node-type` label. Normal workloads are deployed to nodes with `node-type = generic`, and Jenkins deploys to nodes with `node-type = dev`. 

We also separate workloads by namespace: `dev` for development chores, `chaos` for mainline development, `pre-prod` for testing, and `prod` for production. We label nodes and create namespaces easily from the command line:

```
kubectl create namespace chaos
kubectl create namespace dev
kubectl create namespace pre-prod
kubectl create namespace prod
kubectl label --overwrite nodes --all node-type=dev
```

Now we can deploy Jenkins using our custom Helm chart repo:

```
helm upgrade --install --wait                                       \
    --namespace=dev                                                 \
    --set service.initialStartDelay=0                               \
    --set service.image.nameTag='stevetarver/jenkins:2.121.1-r0'    \
    --set minikube.enabled=true                                     \
    jenkins-1                                                       \
    makara-stable/jenkins
```

We can watch the deploy progress on the command line - when the pod switches to Running, tail the jenkins log for the first time admin password, then get the service url:

```
ᐅ kubectl get pods -n dev --watch
NAME                         READY     STATUS              RESTARTS   AGE
jenkins-1-675c79ccbd-dmwb7   0/1       ContainerCreating   0          1m
jenkins-1-675c79ccbd-dmwb7   1/1       Running   0         2m
ᐅ kubectl logs -n dev jenkins-1-675c79ccbd-dmwb7
# ...
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

084fbc410f814997a250aaf2bc04f82e

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
# ...
ᐅ minikube service --url jenkins-1 -n dev
http://192.168.64.12:30091
http://192.168.64.12:30163
```

Browse to `http://192.168.64.12:30091`, enter the password, click "Select plugins to install", click "None" in the menu bar, and then "Install" - our plugin list is bundled in the image. Create your admin user and complete the install.

**NOTE** Jenkins has a long readiness probe to accommodate worst case download speeds, etc., so you will have to wait about 4 minutes for everything to startup.

**TIP**: If you want to start over from a clean slate:

```
ᐅ helm list
NAME     	REVISION	UPDATED                  STATUS   CHART         NAMESPACE
jenkins-1	1       	Tue Jul  3 12:51:22 2018 DEPLOYED jenkins-1.0.0 dev
ᐅ helm delete --purge jenkins-1
release "jenkins-1" deleted
```

## Configure Jenkins

This pipline has hard-coded references to Jenkins configuration which must exist prior to the first build:

* Credentials
    * `dockerhub-jenkins-account`: permission to pull from private repos
    * `github-jenkins-account`: permission to pull from private repos during build
    * `nexus-jenkins-account`: permission to push/pull from our private Nexus
* Environment variables:
    * `TARGET_ENV`: identifies what part of the pipeline to execute, target environment. One of `dev`, `pre-prod`, or `prod`.
    * `K8S_CLUSTER_TYPE`: allows us to configure our projects to deploy to minikube, a standard k8s cluster, or something else.

Since the Docker and GitHub repos we'll be using are public, you can use your own creds for Docker/GitHub.

To set up credentials in Jenkins, from the Jenkins landing page:

1. Click on "Credentials" in the left sidebar
1. Click on "System" that pops up below "Credentials"
1. Click on "Global credentials"
1. Click on "Add Credentials" and add items for each "*-account" item in the list above. The Nexus account can have any creds - we won't use it today, it just needs to be present.
    * Kind: Username with password
    * Scope: Global
    * Username:
    * Password:
    * ID: {an id from the credential list above}
    * Description: {the same id from the credential list above}

To set environment variables in Jenkins, form the Jenkins landing page:

1. Click on "Manage Jenkins" in the left sidebar
1. Click on "Configure System"
1. Under "Global Properties", check "Environment variables"
1. Click the "Add" button to add the following:
    * Name: TARGET_ENV
    * Value: dev
    * Name: K8S_CLUSTER_TYPE
    * Value: minikube
1. Click the "Save" button at the bottom of the page

**TIP**: Be careful not to add a blank environment variable. Your build will abort in the first pipeline stage that may, or may not, show an error indicating a bad "env" use.

Next, we need to register our Jenkins shared library with Jenkins so it is trusted and available to pipelines:

1. Click on "Manage Jenkins" in the left sidebar
1. Click on "Configure System"
1. Under "Global Pipeline Libraries", click "Add"
    * Name: jenkins-pipe
    * Default version: master
    * Click "Retrieval method" -> "Modern SCM"
    * Select "GitHub"
    * Credentials: github-jenkins-account
    * Owner: stevetarver
    * Repository: jenkins-pipe
1. Click the "Save" button

Note how little configuration is required to get this Jenkins building jobs. I'd call that success for one of our primary goals. The challenge is to keep people from adding configuration to Jenkins instead of keeping it in their source code.


## Create the first build job

In the dev environment, we will build and deploy the master branch AND build every branch that is checked in, using the Multibranch Pipeline job type. Let's create one of those as a test subject.

1. From the Jenkins landing page, click "New Item" in the left sidebar.
1. Enter the GitHub repo name as "Item name": ms-ref-java-spring
1. Select "Multibranch Pipeline"
1. Click OK

In the job configuration:

* Branch Sources
    * Select GitHub
    * Credentials: github-jenkins-account
    * Owner: stevetarver
    * Repository: ms-ref-java-spring
    * Add Behavior: Filter by name (with wildcards) and Exclude "candidate", "hotfix" and "release"; these branches are built in other enviornments.
* Click the "Save" button

After the repository scan, Jenkins should recognize the master branch - and start building it. You can switch to the console view and follow along.

The `ms-ref-java-spring` Jenkinsfile `containerPipeline()` call inspects global environment variable `TARGET_ENV` to determine which pipeline to run. In this case, it will execute the `dev` pipeline which includes build, test, package, archive, and integration-test stages for the master branch.

### Inspecting the deployed service

There are several views of our service. So far we have been focusing on the command line, let's switch to the k8s dashboard. You can open a browser page to that with:

```
ᐅ minikube dashboard
```
From the dashboard, change the namespace to "chaos" and look at the "Deployments", "Pods", "Replica Sets", and "Services". Note that from the "Pods" page, you can select the ms-ref-java-spring pod, and from that page, bash into the container or see its logs.

We can also interact with the service from outside the cluster. To find the service url:
```
ᐅ minikube service --url -n chaos ms-ref-java-spring
http://192.168.64.12:30095
```
Then we can browse to:

* data: `/reference/java/contacts`
* health: `/healthz/liveness`
* metrics: `/metrics`
* spring: `/actuator`

## Simulating a `pre-prod` deployment

As discussed in previous articles, we expect Enterprise IT orgs to have separate clusters for different production levels and have designed for this by providing `dev`, `pre-prod`, and `prod` Jenkins configurations. Briefly reviewing duties and responsibilities:

* `dev`: build & deploy mainline development (`master`) branch; build, test, and package feature branches for continuous integration
* `pre-prod`: build & deploy release `candidate` and `hotfix` branches; these time-share the pre-prod environment for frugality
* `prod`: perform a measured deploy of a `candidate` or `hotfix` proven image

In the developer workflow, when it is time to start the march to production, developers will merge `master` into `candidate` - we'll do that now to prepare for the pre-prod deploy.

Changing our `dev` Jenkins to look like a `pre-prod` Jenkins is pretty straight-forward, with one catch. Helm has a sticky notion of releases: if you deploy a release to a namespace, it will redeploiy to that same namespace even though you specified another. This means we need to either rename our release, nah - adds confusion, or delete the existing helm release. Let's do that and setup Jenkins to look like pre-prod:

* In a shell: `helm delete --purge ms-ref-java-spring`
* From the Jenkins landing page, click the "Manage Jenkins" link in the left sidebar
* Click the "Configure System" link
* Change Environment variable "TARGET_ENV" to "pre-prod"
* Click on the "Configure" link on the ms-ref-java-spring job
* In "Branch Sources" add "Behaviors", add "Filter by name (with wildcards)"
    * "Include" = "candidate hotfix" 
    * "Exclude" = ""

We can find the service url with:

```
ᐅ minikube service --url ms-ref-java-spring -n pre-prod
http://192.168.64.12:30096
```

And browse to `http://192.168.64.12:30096/reference/java/contacts` to verify the service is working.

## Simulating a `prod` deployment

Our prod environment targets a single branch, so we will clean up as before, delete our multibranch job, and create a new prod job.

On the Jenkins landing page

* Click "Manage Jenkins" and then "Configure System"
* Change "Environment variable" "TARGET_ENV" to "prod"
* Click "Save"

Delete the Multibranch Pipeline for ms-ref-java-spring and then:

* From the Jenkins landing page, click "New Item" in the left sidebar
* In "item name", use "ms-ref-java-spring"
* Select "Pipeline" job type
* Click OK

In the job configuration:

* General
    * Check "Do not allow concurrent builds"
    * Check "Do not allow the pipeline to resume if the master restarts"
* Pipeline
    * Select "Definition" -> "Pipeline script from SCM
    * Select "SCM" -> "Git"
    * Under "Repositories"
        * "Repository URL" -> "https://github.com/stevetarver/ms-ref-java-spring.git"
        * "Credentials" -> "github-jenkins-account"
        * "Branches to build" -> "Branch Specifier (blank for 'any')" -> release
* Click the "Save" button

Now, let's deploy to production. In the developer workflow, when the `candidate` branch is of sufficient quality, it is merged into the `release` branch - I'll do that now. 

**NOTE** The `release` branch provides a Jenkins job target and quick access to the source used to create the docker image, but nothing is actually built from the code in this branch.

After `candidate` is merged into `release`, we can click the "Build Now" link. During the build, the prod part of the pipeline will initialize the build parameter configuration - a short cut for filling it in manually. After this build fails, a new link, "Build with Parameters" will show on the Build Job page - click that.

Select "releaseType": "candidate" and press the Build button.

During the release, the pipeline will:

* Tag the release branch with initial version 1.0.0
* Create a GitHub release using commits since the last release as release notes
* Re-tag the candidate docker image removing "-candidate" from the image name and using tag 1.0.0: `stevetarver/ms-ref-java-spring:1.0.0`
* Deploy the helm chart, using this image, to the prod namespace

We can find the service url as before:

```
ᐅ minikube service --url ms-ref-java-spring -n prod
http://192.168.64.12:30097
```

And list all contacts to prove the service is working by browsing to `http://192.168.64.12:30097/reference/java/contacts`.


## Tips

* To see the minikube k8s dashboard: `minikube dashboard`
* In the minikube dashboard, you can tail Jenkins logs by selecting the "dev" namespace, clicking on "Pods", clicking on the jenkins pod link, then the "LOGS" icon in the menu bar.
* In the minikube dashboard, you can bash into the Jenkins container by selecting the "dev" namespace, clicking on "Pods", clicking on the jenkins pod link, then the "EXEC" icon in the menu bar.
* You can get the Jenkins browser url with:
    ```
    ᐅ minikube -n dev service list
    |-----------|-----------|--------------------------------|
    | NAMESPACE |   NAME    |              URL               |
    |-----------|-----------|--------------------------------|
    | dev       | jenkins-1 | http://192.168.64.9:30091      |
    |           |           | http://192.168.64.9:31397      |
    |-----------|-----------|--------------------------------|
    ```
* You can completely remove the Jenkins deployment with:
    ```
    ᐅ helm list
    NAME     	REVISION	UPDATED                  STATUS   CHART         NAMESPACE
    jenkins-1	1       	Tue Jul  3 12:51:22 2018 DEPLOYED jenkins-1.0.0 dev
    ᐅ helm delete --purge jenkins-1
    release "jenkins-1" deleted
    ```


## Epilog

I have omitted the many Jenkins bugs, constant version and plugin instability, breaking features, etc., from these articles. Jenkins is really a bear to work with and I have added a lot of manual maintenance overhead to insulate developers from this. The whole tedious Jenkins upgrade process with creating matched plugin version lists, migrating the `jenkins_home` to not lose configuration and avoid corruption, etc. is purely prevention for problems we have seen. When I return to build pipelines, I will probably start with experiments in [Spinnaker](https://www.spinnaker.io/), [go CD](https://www.gocd.org/), and [Concourse CI](https://concourse-ci.org/) to try to find a solution to these problems.

**UPDATE 10 JUL 2018** This solution was developed prior to any other Jenkins solution being sufficiently robust. I see that the official  [Jenkins chart](https://github.com/kubernetes/charts/tree/master/stable/jenkins) has become much more mature/robust. If I had to start over, I would use that as a base and augment it appropriately. 
