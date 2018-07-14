---
layout: post
title:  "Service operational concerns"
date:   2018-02-12 11:11:11
tags:   k8s kubernetes helm postman service mesh distributed tracing microservice devops operations
---

**TL;DR**. Describe Kubernetes deployed service's operational duties and responsibilities.

<img style="float: left;" src="/images/logo/k8s.png">

Last month, we looked at milestone facilities required during a Kubernetes migration. This month, we'll look at common concerns for services deployed to that Kubernetes cluster with a focus on providing tools for developers to be effective in this new arena and providing a consistent operational plane to manage it.

From a "whole product concern", business expects engineering to provide reasonable business agility on top of a strong operational story to both deliver products expediciously and avoid an operational drain on resources and morale.

The operational piece of this equation is part cluster management, part service management; the service management part comes largely from implementing the features listed in this article. The business agility piece of the equation, delivering a product/feature from concept to customer, comes from automating, or boilerplating, these features.

This prioritized list is a migration roadmap; roughly oriented from base features to those needed for scale, while accommodating interdependencies. At some point, you want all services to somehow inherit all of this functionality - perhaps through a boilerplate starter app. How do you get to that point? One strategy is to create a reference app that implements each feature with the intent of producing some piece of shared code or template that can be rolled out for wider use.

1. [ ] Logging
1. [ ] Health endpoints
1. [ ] Smoke test
1. [ ] Local cluster support
1. [ ] Helm chart
1. [ ] Deployment pipeline
1. [ ] Facts & Secrets
1. [ ] Integration tests
1. [ ] Load test & Resource tagging
1. [ ] Metrics
1. [ ] Graceful shutdown
1. [ ] Open tracing
1. [ ] Service mesh

## Logging

_**What:**_ A logging standard that each service implements.
_**Reuse:**_ Logging standard, reference implementation, starter project that includes the reference implementation.

Logging is the first item on the list because it provides the fundamental view of what is going on in your service and can aid developers in almost every other chore.
 
Logging provides:

* enough information to reproduce a request as an API call or to debug in the IDE.
* a narrative of decisions made and actions taken while processing the request
* summary metrics like call counts, duration, and response code

This blog has many posts on Logging concerns, concepts, and even a standard. When I wade through those, I think these are the most imporant aspects:

* Have a logging standard
* Define a standard key name dictionary
* Log to stdout in JSON
* Provide a "local" mode focusing on human readability

Logging itself provides for local and in-cluster issue remediation. Adding a logging standard provides the foundation for standardized logging dashboards that can focus on single services or their aggregates. The logging standard defines the fundamental key names that, when in a log aggregator, can be thought of as SQL column names allowing queries that select or aggregate over the key name. Adding a standard key name dictionary; so every service uses the same name for every semantically equivalent log record key, vastly improves your query ability. For example, if many services use a customer account number, and every service uses the key name "customerId" for that value, it is trivial to write a query to see every action affecting that customer's account.

We have to log in JSON for the log aggregator, but these logs are impossible to read. You can really boost developer productivity, and joy, by taking the time to create a logging profile for local development: positional, colored, and focusing on the narrative.

## Health endpoints

_**What:**_ A standard approach to service health endpoints.
_**Reuse:**_ Templates for the health endpoint implementation.

Health endpoints are second on the list because the form is so easy to implement and having a service endpoint facilitates the next item in the list.

Kubernetes' fundamental notion of service health comes from the readiness and liveness probes, described [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/).

> The kubelet uses liveness probes to know when to restart a Container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a Container in such a state can help to make the application more available despite bugs.
>
> The kubelet uses readiness probes to know when a Container is ready to start accepting traffic. A Pod is considered ready when all of its Containers are ready. One use of this signal is to control which Pods are used as backends for Services. When a Pod is not ready, it is removed from Service load balancers.

After mining docs, we can add a little detail to the implementation and our own customizations. Note: I group health probes functionally, under `/healthz`, to avoid URI sprawl. 

The `/healthz/readiness` endpoint returns `200 OK` when the service is able has completed startup tasks and is respond to requests and `503 Service Unavailable` otherwise. A service should not report that it is ready until it has loaded all configuration, executed potentially long running startup tasks, and can connect to all upstream components.  A service can be 'live', but not 'ready' - in the case of an upstream partner going off-line for example.

The `/healthz/liveness` endpoint returns `200 OK` when the service is able to respond to requests and `503 Service Unavailable` otherwise. It can be difficult to tell if a service is broken from within the service; a light weight, synthetic transaction to its backend may be the simplest way to check. Strategies will vary by language, framework, and domain.

The readiness probe needs only to prove connectivity to its upstream partners; its upstream partners are responsible for reporting their own health. If each service checked upstream connectivity through a normal API call, one could generate a lot of load just handling health probes. To reduce this load, we add the notion of a service ping.

The `/healthz/ping` endpoint should always return `200 OK`. This provides a very light weight communication check between services.

**NOTE** k8s will use http status codes to determine liveness and readiness, but your PagerDuty friends will appreciate you returning the checks that were performed and the results.

### Reference implementations

* [python](https://github.com/stevetarver/ms-ref-python-falcon/blob/master/app/controller/health.py)
* [java](https://github.com/stevetarver/ms-ref-java-spring/blob/master/src/main/java/com/makara/ms_ref_java_spring/controller/HealthzController.java)
* [groovy](https://github.com/stevetarver/ms-ref-groovy-spring/blob/master/src/main/groovy/com/makara/ms_ref_groovy_spring/controller/HealthzController.groovy)


## Smoke test

_**What:**_ A black box test of major service functionality.
_**Reuse:**_ The test runner and idiom for waiting for services to become available.

A smoke test is item three on the list because we now have a service endpoint to test and the smoke test will be the initial integration test used in the deployment features up next.

The smoke test is a fast, black-box test, proving major service functionality. It is the basis for the initial integration test, used to verify production deployments, and used by operations during issue remediation.

My favorite tool for service testing is [Postman](https://www.getpostman.com/), and the command line runner, [Newman](https://www.getpostman.com/docs/v6/postman/collection_runs/command_line_integration_with_newman), for automating those tests. Standardizing on a single testing tool provides operations with a common interface for service interactions - whether these are automated tests or used to debug production issues.

At this point in the reference service development, we only have `/healthz/*` endpoints to test. That's OK because our focus is to build the infrastructure to make this easy to evolve. Let's take a look at how I set up the smoke test in one of my reference apps.

In [ms-ref-java-spring/integration-test](https://github.com/stevetarver/ms-ref-java-spring/tree/master/integration-test), I have a postman folder containing tests and environments and a bash `run.sh` script to run any combination of the two. The tests are ordered api calls and metadata and the environments are local variables specific to that target, like the api url. I have a standard naming convention for tests: `test.*.json` and environments: `env.*.json`. Using the `run.sh`, I can:

* run an integration test locally against the service running in the IDE and produce a report
    `./run.sh -r -t integration -e local.native`
* run a load test locally against the service running in Minikube
    `./run.sh -n 10 -t integration -e local.minikube`
* run a smoke test during build pipeline deployment to production
    `./run.sh -q -b -t smoke -e prod.ops`

Both developers and operators should be proficient in Postman, to create great initial tests, to create ad hoc tests for debugging, and to understand behaviors and reports. Standardizing the project structure and `run.sh` script lets any developer or operator walk into a new repo and feel comfortable.

As your smoke tests evolve beyond health endpoint testing, budget time to create test accounts in all facilities so you can isolate test activity from real client activity.

Somewhere a little further down the road, you may consider having a central test runner for all services that: continually runs the tests, stores the results, and presents results to users. This can be an invaluable production canary. It is then easy to expose an API that will allow initiating a test from [Slack](https://slack.com/) or view any results from Slack.

## Local cluster support

_**What:**_ Simple tooling to locally build & run a docker image, start all components, and perform a local deploy.
_**Reuse:**_ Templates for `Dockerfile`, `docker-compose.yaml`, and Minikube deployment.

Item four on the list is the ability to package and run your service which only starts to make sense when we can interact with it and we can automate that interaction. Local cluster support development is tightly coupled to, and will be co-developed with, the helm chart and the deployment process.

If you are migrating to Kubernetes, it is likely that most of your staff has little knowledge of containers, distributed systems, and Kubernetes. That's OK. They don't really need much. The challenge is determining what knowledge you need from the vast documentation available. Having local container and cluster support will provide a local playground, where you can identify what to teach, and where they can experiment and learn.

During this step, we want to create a standard set of tools to:

* Build and run a Docker image
* Start service components with [Docker Compose](https://docs.docker.com/compose/)
* Deploy service components to [Minikube](https://github.com/kubernetes/minikube)

Why Docker tools? Each developer should be able to confidently change their Dockerfile and evaluate it prior to commit. If this exists, a developer can do things like update all dependencies and run the integration test against a local Docker container to identify breaking changes. Otherwise, they would update dependencies, let the build system push those changes to the cluster, fail the integration test, and rollback. Then they would have to revert the change and start the deploy process again. The first approach takes 15 minutes. The second takes an hour and impacts everyone else on the team. There are many other similar examples.

Our group allows developers to implement services in any language they choose provided they implement the standard operational support. After living with this freedom for a year, our developers naturally settled on a small set of languages. Each of the Dockerfiles for services in a language family look remarkably similar. This means that the Dockerfile you create for your reference service can easily be reused by manual copy, or creating a boilerplate starter app that includes it.

Why Docker Compose? It takes Kubernetes out of the picture and lets developers focus on service interactions. During project startup, if you take the time to create mock facilities, like a [database seeded with realistic schemas and data](https://github.com/stevetarver/sample-data), and tools to easily keep them up to date, your developers can debug many service problems locally. This is also a good isolation point from the cluster - it can help the developer divide and conquer the stack - differentiate a service problem from a cluster problem. The good news is that once a Dockerfile and your mocks are created, adding a `docker-compose.yaml` is trivial.

Why Minikube? Kubernetes is complex; there are different components involved and several nuances to service operation that simply don't exist at the Docker Compose level. As you spend more time with your cluster, your deployed workloads will become more sophisticated, and Minikube provides a perfect proving ground. Also, when it is time for a Kubernetes version upgrade, Minikube can provide a test area for that new version.

Also, Minikube is a great proving ground when developing your Helm chart and the deployment pipeline. You can isolate that activity to a single laptop to prevent cluster disruption and shorten the code-deploy-test loop.

If you build these features into your reference app, you have boilerplate implementations for all services that follow. It will probably help to have training on each of these facilities. Perhaps devote an hour to a Dockerfile deep dive covering all statements and decision criteria behind each. Another hour on the connective tissue between components in a `docker-compose.yaml`. Finally, perhaps a half day to get Minikube setup and deploying the reference app and learning the fundamentals. Having the reference app's [Helm](https://github.com/kubernetes/helm) chart available free's up discussion to focus on Minikube.

This is another example of a standard implementation, even if it is copied to each repo, allows developers to walk from one project to the next and feel somewhat at home. All of the operational concerns and implementations are the same, only the business domain code changes.

## Helm chart

_**What:**_ Templatized Kubernetes manifests used to deploy to the cluster.
_**Reuse:**_ Standard form for Helm charts

Now that we can produce a Docker image and have Minikube installed, we can start developing our approach to deploying services.

Workloads are deployed to Kubernetes through yaml manifests that describe all aspects of the containers, ingresses, security policy, facts & secrets, monitoring rules, etc. When we start thinking about automating deployments to several clusters, some questions arise:

* How do we change manifests to suit the target cluster?
* How do we rollback a deployment to a known good state?

[Helm](https://github.com/kubernetes/helm) provides a good balance of simplicity and sophistication; it is easy to start with and can grow to accommodate most needs. Helm consists of a command line client that interacts with a cluster deployed server (tiller) to provide install, upgrade, and delete for a set of manifests. Its greatest strength, however, comes with the addition of 'go' templating to manifests. Templating provides variable substitution, grooming, conditional logic, and control flow.

At this point in the reference service development, we only need a `deployment.yaml` and a `service.yaml` to fully describe our service. There are many examples on the net, including `/helm` directories in my `ms-ref-*-*` repos.

As you experiment with Helm and the Kubernetes manifests, you will develop strategies that suit your clusters which you should codify in your helm charts. You can allow copy & paste from the reference service or bundle the helm chart in a boilerplate starter app. Again, the goal is that when a new operator or developer looks at a helm chart, much of it is familiar.

## Deployment pipeline

_**What:**_ A method for building source code and deploying it to the cluster.
_**Reuse:**_ Build and deploy infrastructure and a common pipeline.

Now that we have Minikube and a helm chart, we can start developing the deployment pipeline.

This is a surprisingly big topic; it will consume a lot of time and effort. Check out these previous posts for excruciating detail, especially if you will be using Jenkins:

* [An Ephemeral Jenkins](http://stevetarver.github.io/2017/07/08/ephemeral-jenkins.html)
* [Terms & Concepts](http://stevetarver.github.io/2017/08/14/design-ci-cd-cd-pt1.html)
* [Designing the build pipeline](http://stevetarver.github.io/2017/09/09/pipeline-design-pt2.html)
* [Interactions with GitHub](http://stevetarver.github.io/2017/10/11/pipeline-design-pt3.html)
* [A custom Jenkins pipeline](http://stevetarver.github.io/2017/11/14/pipeline-design-pt4.html)
* [A local Jenkins pipeline](http://stevetarver.github.io/2017/12/19/pipeline-design-pt5.html)

In all those thousands of words, I think the key points are:

* Develop tooling to easily setup your build & deploy system in Minikube.
* Develop a standard pipeline that all services merely configure.
* Develop a version upgrade strategy.
* Develop a tragic loss strategy.

The Minikube setup step provides a short code-deploy-test loop that is critical in early implementation. As more people use the cluster, it provides a safe, non-disruptive testing ground for new features, refactoring, and version upgrades.

You should strongly consider developing an opinionated build and deploy pipeline that all services use. This provides the freedom to encode organizational quality gates, auditing, reporting, notification, etc. Consider the point where you have 20 services deployed and you have a new audit requirement. With a standard pipeline, you can implement and test that in an hour and every service build will execute it. Without a standard pipeline, you will have to wade through 20 services to find the right implementation point and each may have potentially different bugs.

A version upgrade strategy addresses moving all infrastructure configuration to a new instance, with a new product version, with minimal team disruption. This is critical for Jenkins who is notorious for introducing bugs in new version, plugin incompatibilities, and difficult rollbacks on failure.

Tragic losses happen and being prepared for them turns a very big issue affecting all staff, to a non-issue. This strategy should rapidly build and configure infrastructure. The more you can automate, the better.

The pipeline will be the primary developer interface to the cluster. If done well, developers should not really notice that they are using it. They commit to GitHub and some minutes later, that code is magically working in the cluster. It is also a place to automate away organizational concerns to reduce mundane developer load.

When you reach suitable quality in Minikube, you can move your infrastructure to your cluster. Be sure to budget some time to address issues not exposed during Minikube development.

## Facts & Secrets

_**What:**_ A standard approach to providing facts & secrets to cluster services
_**Reuse:**_ Facts & secrets infrastructure and standard use.

Now that we can deploy the reference service to Minikube, and our cluster, we can start thinking about some standard way to provide facts and secrets to that service. This makes sense as step seven because the deployment pipeline is fresh in our minds and we may use that, and we may need secrets to connect to upstream partners.

Facts are service configuration. Some facts are included in your helm chart; those you want to modify at runtime should be stored in your fact repository. 

Secrets are sensitive facts, account names and passwords chief among them. Secrets deserve a dedicated store and should never be included in your helm chart. Even if you encrypt secrets to safely store them in your source repository, it is likely that someone will decrypt them and check those secrets into the source repository.

The simplest fact repository is a Kubernetes ConfigMap. Simple, but also several benefits:

* ConfigMaps are included in the service helm chart and therefore easy to use in local Minikube deploys.
* Helm charts are stored in version control and therefore versioned and audited. If a bad fact is committed, you can easily restore to a known-good version and track down the culprit.
* Since they are not stored in the build pipeline, they are not lost when the build infrastructure fails, and the build infrastructure is easier to restore on tragic loss because there is less configuration to apply.

Although there are some Jenkins plugins that simplify storing facts in Jenkins and integrating them into deploys, I think that facts should not be stored in Jenkins:

* Anyone can change them, there is no practical auditing.
* There is no versioned record of the configuration, no way to roll back breaking changes.
* The Jenkins editing facilities are not robust.
* You can't easily audit fact use - do things like verify that all facts exist in all build pipelines, or validate appropriate use of the fact.
* If your build pipeline suffers a tragic loss, like persistent volume claim failure and backup corruption, you have no record of your facts. It also makes recovering the build infrastructure more tedious and time consuming.

There are more robust fact stores - I am a big fan of [Hashicorp's Consul](https://www.consul.io/). Most languages and frameworks have a Consul integration that simplifies rolling updates. It also supports organizational goals like auditing. When you implement a Consul solution, ensure that you create tooling to standup a local consul for disconnected developer use, and, provide central configurations for local developer use. This maintains developer productivity in their local environment.

Secrets must be either merged into the helm chart during deploy or available to the service at runtime. If you are merging secrets into the deploy, the build pipeline will not be a sufficient store - it can die. You need enterprise grade secure storage that is backed up. Then you will need migration from the durable store to the build pipeline. 

Storing secrets in Jenkins is a simple start, but as service count grows, you will want a more automated approach. You will see the value in this effort with your first Jenkins loss. The standard seems to be [Hashicorp's Vault](https://www.vaultproject.io/) - I'm a big fan of this product as well. As with Consul, when you start implementing this solution, take care of your developer's local environment.

## Integration test

_**What:**_ An exhaustive set of black-box service tests.
_**Reuse:**_ Test running scripts, some test idioms.

Now that we can easily include account names and passwords in our deploy, we can start adding functionality to the reference service and expand our lightweight smoke test into our comprehensive integration test.

The integration test should exercise all product functionality. You may include regression tests or leave those as a separate test suite. You should be able to be run this test locally, against the service running in the IDE to aid debugging, or in a local docker image through Docker Compose, or against Minikube, as well as any Kubernetes cluster.

Monoliths can verify most functional and integration concerns with unit tests - everything is in the same code base. When you switch to a microservice architecture, unit tests no longer dominate your testing arsenal. It is much more important to verify service operation among its peers than in isolation. It is also important to give operations the tools it needs to evaluate the service during production issues. You should adjust your "test writing budget" to put emphasis on integration tests over unit tests.

Once defined, the integration test will be used to validate pre-production services and should be able to evaluate production services for proper operation. It is appropriate to spend significant effort on this test suite: bugs found sooner are cheaper to fix, and reducing production issue remediation time is a boon in many ways.

I like Postman/newman for all black-box testing. This means you can clone your smoke test to provide the first integration test and use the common `run.sh` test runner to run either in any environment.

## Load test & resource tagging

_**What:**_ Tooling to load test a service and produce Kubernetes manifest resource limits.
_**Reuse:**_ Strategy and load test scripts.

Now that we have an integration test, we can apply it to our service and quantify how it does under load.

The simplest load test could be start your service with Docker Compose and run the integration test 10 times in each of 10 shells on your laptop. While the test is running, run `docker stats` to watch CPU and memory use.

Once you have your base metrics, you can multiply by some design margin (1.5) to produce CPU and memory resource tags described [her](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/).

As your services evolve, you may want to create dedicated load tests that focus on a read load, a read-write mix, or some production discovered profile. When metrics and monitoring are available, you can run your service in the cluster and create dedicated dashboards to show the service under load.

Beyond resource tagging, load tests can be used to discover memory leaks, race conditions, and how the runtime performs underload to tweek garbage collection, etc.

## Metrics

_**What:**_ Export service metrics and provide a central facility to collect, query, and view them.
_**Reuse:**_ Monitoring infrastructure, metrics configuration.

At this point, we have covered enough operational concerns to run and manage a small scale cluster. Operations is still limited by metrics collected from logs and the dashboards you can create from them. Publishing well considered metrics and collecting them in a central facility will provide a much better view of the services in aggregate, as well as in isolation.

There are three basic phases to metrics publication:

* what is easy to get
* what is required to answer operational questions
* what is required to answer domain questions

All monitoring solutions, managed, on-prem, or open source, will provide some kind of agent to collect the metrics from the service AND expose metrics that are easy to collect. This is a really good place to start: it provides some metrics to work with to get started dashboarding and alerting. During initial dashboard creation, you will have "ah ha" moments where you discover one piece of information that will open up many answers. Time to enhance the service metrics published. As you learn more about your service domain, you may reach a similar "ah ha" moment, and add some domain specific metrics.

If your cluster already has a monitoring system, it makes a lot of sense to hook into that. If not, you will want to ensure your cluster is covered by the monitoring system you build.

After some time with the dashboards, you will be hit with the profound impact of standardized names, and similar to logging, create a standard metrics key name dictionary. If you are a polyglot language shop, you will probably have to rename metrics in the client to reach naming consensus.

Monitoring is closely tied to alerting - all monitoring solutions should include an alerting package that can tie into your:

* chat rooms - items that need someone's attention during business hours
* ticketing - an odd situation that needs to be investigated
* pager - service or cluster fire

Once the monitoring system and its dashboards and alerting are in place, you have a decently appointed cluster and can open the gates a bit wider to let in more customers.

## Graceful shutdown

_**What:**_ A system to ensure no message is dropped.
_**Reuse:**_ Graceful shutdown idiom.

Now that we have a small, fairly robust, cluster, we should turn our attention back to production resiliance and reliability - ensure that no important messages are dropped. A Kubernetes cluster is volatile; reschedules happen all the time. We need to ensure that a reschedule does not abandon client requests.

When Kubernetes needs to reschedule a pod to another node, it sotps sending requests, sends the container a `SIGTERM`, waits 30 seconds, and then sends a `SIGKILL`. Our service needs to be able to clear all requests between the `SIGTERM` and the `SIGKILL`.

A first stab at this, if your service uses a thread-per-call model, is to limit a single pod's request worker thread pool count and use horizontal pod autoscaling to adjust pod count for load.

But what about asynchronous services and those that fetch work from a messaging system like [NATS](https://nats.io/)? You will still want to limit the number of concurrent messages in flight and provide an out-of-band connection to the client to tell it to stop fetching work.

## Open tracing

_**What:**_ A system to record service communications.
_**Reuse:**_ The infrastructure and idioms.

At this point, our cluster is growing and understanding how services interact with each other is becoming a larger concern. Perhaps a request timed out but you don't know who in the service chain is at fault. Perhaps you don't understand which service is creating load for other services. This operational view of service interactions can drammatically reduce issue remediation times and is worth the effort before, or when, you start to feel these growing pains.

Although [Twitter's Zipkin](https://zipkin.io/) pioneered the field, [Jaeger's Open Tracing](https://www.jaegertracing.io/) feels like it is becoming the standard. The system consists of an agent, a collection facility and a UI.

By simply installing the agent in your service, you will be able to show service topology and look at timing. But don't stop there. You can add spans for external service calls as well. This is invaluable for identifying things like increasing database call duration because of a missing index, etc.

## Service mesh

_**What:**_ A system for programmatically controlling network traffic.
_**Reuse:**_ Infrastructure and helm chart templates.

At this point, you have a robust cluster and understanding service interactions has become critical. You also want uniform implementation of distributed idioms like circuit breaker, retry budgets, and more control over request routing to provide for canary deploys, A/B testing, etc.

I must confess, I have only seen demos of [Istio](https://istio.io/) + [envoy](https://www.envoyproxy.io/) but it is definately on my wish list. Rather than repeat things I've read, here's [18 minutes of Kelsey talking through the promise of Istio](https://www.youtube.com/watch?v=6BYq6hNhceI).

## Epilog

What began as a checklist, ended up a manifesto. If you are trying to wrap your head around getting your organization on a Kubernetes platform, this and the previous article should provide fertile ground for planning.
