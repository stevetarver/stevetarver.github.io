---
layout: post
title:  "Planning a k8s migration"
date:   2018-01-22 11:11:11
tags:   k8s kubernetes
---

**TL;DR** Prioritized features and concerns for an initial Kubernetes migration.

<img style="float: left;" src="/images/logo/k8s.png">

Last January, my partner and I stood up a new Kubernetes 1.4 for a new workgroup. There have been several iterations as we struggled with how to translate Enterprise IT concepts into a Cloud environment. Beyond the challenges of having multiple production levels and 17 productions data centers that span the globe, our biggest challenge was educating staff and motivating them for the move: adopting the changing toolset and changing processes. We learned that one of the key challenges was accurately estimating the staff's maximum rate of change - how much change they could accomodate without rebellion.

Here are some thoughts about this process in the hopes that it can ease your journey.

## A dedicated migration group

The first decision is "Who will lead this migration?" Picking the right team is critical to success.

As of KubeCon 2017 (Dec. Austin), Kubernetes has won the orchestration platform wars - it is the standard distributed workload platform. During its rapid rise, early adopters blazed trails and developed strategies that we can leverage. Justin Dean (SVP Ticketmaster) presents one key idea in his [Tectonic Summit talk](https://www.youtube.com/watch?v=wqXVKneP0Hg): Kubernetes is complex, everyone does not need to know the gory details, select a small group to become your Kubernetes experts.

Whether you are standing up an on-prem cluster, or using a managed cluster from one of the big cloud providers, you will want a small, highly capable group that knows the ins-and-outs of Kubernetes. How do you pick such people? Here are some key traits:

* Proven self-learners and self-starters who love new technology
* Enthusiastic about sharing new ideas through presentations and demos
* Operational AND Developer skills
* Sophisticated design aesthetic - able to quickly match constraints with a solution

This group of self-motivated operators and developers will be designing solutions and tooling and promoting them throughout the org. Every individual does not need excellence in all skills above, but all skills must be covered.

Once the group is formed, use the [Certified Kubernetes Administrator program](https://training.linuxfoundation.org/linux-courses/system-administration-training/kubernetes-fundamentals) as your Kubernetes University. It will provide both basic training and exposure to implementation alternatives that will be essential to assembling the right choices for your org.

## Understand your clients

A Kubernetes migration doesn't happen in a vacuum: it can be ordained from on high, a skunk-works project, something in between, and usually, some combination. Whatever the effort origin, there will be many in the org that are happy with the way things are and resent the change. You have officially entered the realm of "Propogating change through an organization" and need some mastery of those skills or the migration will be long and painful.

Mike Cohn's book [Succeeding with Agile](https://www.amazon.com/Succeeding-Agile-Development-Addison-Wesley-Signature-ebook/dp/B002TIOYWQ/ref=mt_kindle?_encoding=UTF8&me=&qid=1531247930) has sections on propogating Agile through an Enterprise and are a great strategy primer. I highly suggest buying the book, just for those sections. 

Cohn's book divides people into:

* originators: 25% - change drivers
* pragmatists: 50% - generally practical, agreeable, and flexible
* conservers: 25% - change averse; prefer status quo

Effective communication requires different strategies for each group. Originators are already on-board but may lose focus when the next shiny toy is announced. Most pragmatists are relatively easy sells with well designed tooling (described later). Conservers are the most difficult: even after addressing their concerns, they will resist because it is "change". Cohn lists the basic reasons for resisting change as:

* Employees
    * Lack of awareness
    * Fear of the unknonw
    * Lack of job security
    * Lack of sponsorship
* Managers
    * Fear of losing control and authority
    * Lack of time
    * Comfort with the status quo
    * No answer to "What's in it for me?"
    * No involvement in solution design

And suggests:

> In attempting to anticipate where resistance will arise, it can be helpful to consider the answers to questions such as these:
> 
> * Who will lose something (power, prestige, clout, or so on) if the transition to Scrum is successful?
> * What coalitions are likely to form to oppose the transition? 
> 
> By identifying individuals who will lose from the change and coalitions that will form to oppose it, you will know where to target initial efforts at reducing resistance.
>
> Cohn, Mike (2009-10-20). Succeeding with Agile: Software Development Using Scrum (p. 98). Pearson Education. Kindle Edition. 

Motivating change is hard, but all time devoted to it will pay dividends throughout the migration.

## Understand your constraints

When you start designing infrastructure and tooling, you should accommodate organizational constraints and group desires from version 1.0. Staff has evolved solutions to all of these concerns; your new facilities will be much better received if they solve these problems as well.

The migrating team should be mostly senior staff who readily list these constraints, but it is worth taking the time to document them to refer to during design, to communicate during presentation, and of course, for auditors.

A group mind-dump is a great place to start, producing a bulleted list covering these organizational constraints:

* How many environments will we support? Dev, pre-prod, prod?
* How much automation is acceptable? Does each code change result in a production change, or will deploys be more graduatual.
* What integrations with other systems are required? Ticketing, notifications, alerting, agile boards, etc.
* What compliance concerns must be addressed? Data sovereignty, encryption, isolation, etc.
* What audit facilities are required? Production changes, source changes, code deploys...
* Other?

You can follow a similar process for "group desires" - what your clients will want out of the new system:

* Simple migration of existing services
* Boilerplate implementations of common facilities
* Training on the new systems
* Other?

And finally, every migration is a great time to fix problems you have been living with. What things can you improve in early or later versions:

* build system
* logging
* monitoring
* alerting
* notifications
* dashboards
* general observability
* other?

## An opportunity to cure engineering ills

When designing new facilities, choose quality over robust feature sets. It is easy to return to a facility to add a new feature but a morale drain to be interrupted by facility flaws that need immediate attention. If others are using these facilities, it is absolutely key that they work flawlessly. Remember that your staff has been using the same tooling for a while; they do not notice any of the current smells. They will not see the effort you poured into the implementation, only that it performed poorly compared with their standard, which erodes confidence in the new system.

We adopted this principle from the beginning, and we still had flaws, but just enough to let us know that devoting sufficient time to getting something right was worth the effort and delay. Our clients accepted minor flaws, everyone writes bugs, but some of the bigger ones created concern over the new platform. 

We also treated each system like a microservice - understanding, documenting, and defining the process including functional boundaries. This allowed completely rewriting/replacing one component instead of landing in the position of a tightly coupled monolith that can never be improved because it is too costly (too big) to change; to complex to modify.

## Produce milestones regularly

Much of the migrating group's initial work will happen in isolation. During these periods you need to continue communicating your progress to instill confidence in the project and keep it fresh in everyone's mind. 

Following the agile notion of short sprints with a end-of-sprint demo worked very well. Even if there was nothing new for staff to use, seeing incremental progress, especially when including improvements to existing systems, kept excitement flowing around the new project.

## Facility development

We can put a pin in communication and high level planning for now, although you should revisit it frequently - it will be your sole measure of progress in early stages.

Now that the group is formed, has a little knowledge of Kubernetes, it's time to start building things. But what? And in what order?

Here is a prioritized list:

* **Logging**: a standard, and a boilerplate implementation. How else will you debug services you deploy?
* **Build & deploy pipeline**: a way to easily move workloads into your clusters.
* **Ingress**: a way to communicate with services in your cluster
* **Monitoring**: services export metrics which consumed by an central facility
* **Durable logging**: logs are forwarded to a central facility, dashboards are created
* **Monitoring dashboards**: dashboards are created on top of service and cluster metrics to highlight problems
* **Alerting**: alerts are sent to appropriate systems to indicate service/cluster faults
* **Service resiliency**: revisit service configuration for resource use, auto-scaling, ensure no dropped requests
* **Distributed tracing**: a way to understand microservice interactions, communication latencies, mesh problems
* **Service mesh**: a way to control request flow, automatic implementation of scaling, circuit breakers, other flow control

**NOTE**: The first two items have been thoroughly discussed in earlier articles in this blog.

## The first project

Some recommend migrating a small piece of functionality as a first project. Although it produces immediate, tangible success, I think this ignores all of the enterprise concerns above. The Kubernetes migration is a marathon and incrementally building rock-solid implementations pays off in building platform confidence as well as providing incremental functionality. I recommend building a reference application that you can evolve along side your infrastructure facilities - a proving ground.

Developing a reference service as your first project allows you to start small, keep unknowns to a manageable count, focus on quality, and document each facility you add. You can use this project in frequent demos to communicate how things work in the new world and your progress. This is important in managing change: you should reveal change gradually. If not demoed frequently, staff will be overloaded by all the new concepts that have accumulated - like trying to eat an elephant.

Now, let's visit each of the facilities listed above for a little more detail.

### Logging

I list logging as step one: your first deploys will have problems and you will likely need logging to help you figure out why.

The first iteration simply logs to stdout which will be available on the node, through kubectl, and in the Kubernetes dashboard. 

The logging standard should:

* require log records to be in json format
* specify standard entries for all services
* require the `x-request-id` in each log record

`x-request-id` is the standard name for a tracing header bundled with a request. The logging implementation should extract that id or create one if it does not exist, include it with every log entry, and the service should include it in all upstream calls. This allows locating a single log statement from a customer ticket, and then a search through the log system you will eventually deploy to find all log entries made by any service while accomplishing the service task.

Stuck on what that standard should be, or how to implment it? Check out my earlier blog posts and use the standard or implementations as a starting point.


### Build & deploy pipeline

You need a solid system to move code from source control to each of your clusters. This is the first interaction with the new system that most staff will have, so it is important to get right. It should be intuitive and easy to use, not have false build errors, and provide for easy debugging when errors occur.

A key, and possibly new, concept that you should embrace is "Build pipeline as code". Kubernetes clusters go down. Pods are rescheduled. You need to build resilency into every aspect: having all build and deploy configuration as code is another key to success. When designing your build & deploy pipeline, ask the question: How do I recover in the face of tragic loss.

Stuck on where to start? I also have several articles describing the SDLC, an enterprise build & deploy strategy, and an opinionated Jenkins pipeline that implements these ideas as well as reference apps that use it.

### Ingress

Once you have a service deployed, you probably want to communicate with it. Kubernetes ingress provides this link to your service from the outer world.

There are two distinct concerns: normal client access and operator access. Normal clients may connect from the public internet or a coporate intranet. They will consume the service provided and should have no other access. Operators, on the other hand, should have access to liveness probes and metrics for issue remediation and building automations and notification tools.

Let's talk ReST URIs at a high level for a minute. As you add to your ReST service inventory, naming becomes more and more important. Without an organizing structure, ReST endpoints become a cacophony. Also, requiring clients to connect to an increasing fleet of frontends can become confusing and frustrating. The first strategy in dealing with this is to mount your service URI by the first URI element, not the implementing package. The second is to have a single frontend that matches on URI prefix and routes requests to the appropriate host. Finally, you should consider adopting functional categorization for the first URI elements. Suppose that one group produced a new billing report - how should that report be accessed. If you required all billing reports to have a URI prefix `/billing/reports`, all billing reports would be naturally grouped and URI paths would be intuitive to clients. Billing reports could be implemented by many groups, and deployed on many hosts, and be consolidated by the frontend load balancer.

That scheme lays the groundwork for service operator access. If you adopt functional URI elements then you can put liveness probes, metrics, etc., outside that scheme and restrict access appropriately. E.g. `/healthz`, `/metrics`.

### Monitoring

Logging is about narrative and best suited to remediating individual issues. Monitoring is about numbers and best suited to understanding individual services or cluster level performance. While some metrics can be accumulated and dashboarded in logging systems, when you move into reschedulable microservices, you need a deeper view of individual pod performance.

There are now many managed monitoring services like [DataDog](https://www.datadoghq.com/) and [New Relic](https://newrelic.com/) and if you can afford it, you should use them because monitoring is complex. If budget is tight, I recommend [Prometheus](https://prometheus.io/).

The first step is to export metrics from your reference application. Each monitoring service should have a simple way to export metrics from the jvm, a wsgi service, or golang to cover basic service performance. Each service implementor will need to understand their domain well enough to add additional metrics that help them understand service performance or problems. These metrics can be added at any time; usually after a difficult incident where one key metric would have significantly reduced remediation time.

If using Prometheus, the second step is to deploy the metrics collection facility. You should start small, with a simple helm chart deployment, that simply collects and durably stores the metrics. This will provide a base to experiment with; dashboarding and alerting are built in, but significant enough effort to warrant their own steps. It also provides a great set of Kubernetes cluster dashboards allowing you to monitor and identify problems with the cluster. You should allocate some time to understand the dashboards and reported metrics. Until we did this, we had no idea that one service was rescheduled hundreds of times a day.

When creating this deployment, ensure that it is easily extensible. You will likely have a few Prometheus deployments, that may foll up to a federated server.

### Durable logging

The first stage of logging was to simply standardize and log to stdout. Durable logging sends log records to an aggregator: a managed service like [logly](https://www.loggly.com/) or [Splunk](https://www.splunk.com/), an existing aggregator, or a new dedicated facility like [graylog](https://www.graylog.org/) or [Elastic Stack](https://www.elastic.co/products). 

Each of these facilities provide robust search through all log records and durability beyond what a reschedulable microservice can promise. With this facility, you can effectively remediate issues, compare successful to unsuccessful requests, create dashboards to start understanding request load and how call duration varies with load, etc.

This milestone is the first point you should consider letting other services into the cluster. Operational chores are still a bit rough, but you can manage if the load is low and the pressure high.

### Monitoring dashboards

Our initial iteration merely deployed the monitoring system and with that, you could create simple charts of a few metrics. The real power comes with dashboarding where you can develop the mythical Single Pane of Glass - a cluster red/green board that indicates overall cluster and service health.

Initial charts will be pretty basic, jvm memory pressures, disk pressure, etc. A well done dashboard will allow you to compare values between many services and then drill down into a single hotspot.

You should allocate time to revisit this initial implementation - your thoughts on good dashboards will evolve considerably.

### Alerting

Now that you can discover hotspots with monitoring or perhaps service problems through logging, you need a way for the cluster to cry for help. You will likely want a phased approach: [Slack](https://slack.com/) notifications for non-threatening business hours problems, ticket generation for one-off that need to be investigated, and [pagerduty](https://www.pagerduty.com/) notifications for cluster fires.

If using Prometheus, you have a lot of granularity in rule definition. Basically, any metric threshold can trigger an alert. You can define cluster level rules, general rules that apply to a jvm or service quality, or include service specific rules in that service's helm chart.

This is another area that will be revisited, probably when investigating an incident root cause.

At this point, you can consider widespread migrations to the k8s cluster. Basic operational support is in place; further additions make operations and self-healing easier.

### Service resiliency

After the basic operational support is provided, and we have accumulated some run time on Kubernetes, we have the skills to start thinking through service resiliency.

First stop: **How do I tell if my node is sick?** The pod spec in your deployment manifest provides for [CPU](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/) and [Memory](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/) resource tagging. These tags help Kubernetes decide which node to deploy a pod to, as well as know when the process is spinning or has a memory leak.

To generate realistic limits, each service needs a load test. The simplest implementation is to create a [Postman](https://www.getpostman.com/) request collection that fully exercises the service, run the service in a docker container, run several instances of [Newman](https://www.getpostman.com/docs/v6/postman/collection_runs/command_line_integration_with_newman) (the Postman command line runner), use `docker stats` to evaluate the load, and develop your limits from the results. 

This is enough to place initial tags on the service although you will want to monitor reschedules in the cluster to see if limits are creating a problem. You should also consider creating a more robust facility so this can be done easily for each service and catch changes as functionality is added.

Next stop: **No dropped messages**. The Kubernetes cluster is volatile; reschedules happen all the time. How can you ensure that no requests are dropped? By default, when Kubernetes wants to reschedule a pod, it stops delivering requests to the pod and gives a 30 second warning in the form of a `SIGTERM`. If the container has not stopped after 30 seconds, it sends a `SIGKILL`.

The simplest answer is to ensure that no service accepts more requests than it can handle in, say, 20 seconds. You can do this in a thread-per-call implementation (e.g. SpringBoot, WSGI, etc.) by limiting the request worker thread pool. You need to know the average call duration for the service, which you can find through the monitoring or logging system. But as you think through the solution, you start to see flaws. Perhaps call durations are short for getting a single value and very large for getting an unbounded list - no way to tell what requests are in the queue. Or your service relies on upstream partners and they may be having problems or there may be a network partition. Or perhaps your service fetches messages from NATS or RabbitMQ instead of receiving requests - it will keep fetching requests. Limits aside, if you do nothing else, you should bound your request thread pool.

The better answer is to ensure all services have a "proper" microservice design. Proper? We know that microservice design involves breaking apart a monolith, but there is a lot of confusion in where to draw boundaries. Especially in this light, I think message boundaries are a great division - any place a message can be dropped is a microservice boundry. In my domains, this means all reads are from a dedicated "high speed read" service. Longer running service tasks are broken into idempotent pieces and have a messaging buffer between microservices, and provide an appropriate reponse mechanism. Non-essential, non-public services, where a client retry does not have a business impact can use less intense designs. Any more detail becomes a book sized answer.

For services that gather work from message brokers and the like, there needs to be some method to communicate the `SIGTERM` to the NATS or RabbitMQ client so it knows to stop processing.

Finally, it would be really nice for the service to just stop when it was told and had finished its work day. Perhaps dump requests to log if there were any it could not complete.

Next stop: **Minimal resource use**. I have a mantra for microservice language choice: python to prototype, go to make it small and fast, and rust to make it correct. The problem is that most enterprises employ jvm developers and optimizing the jvm is tough. Memory use is tricky to tune because of garbage collection: too short a period can impact average call duration, too long a period can cause memory bloat and potentially a reschedule. I haven't worked through this challenge yet, but there are some nice articles showing up, like [this](https://dzone.com/articles/how-to-decrease-jvm-memory-consumption-in-docker-u), walking you through the process.

### Distributed tracing

When you switch from a monolith to a microservice architecture, you are trading one set of problems for another. One of those problems is understanding how the mesh of services is collaborating. If many services collaborate on a single task, and that task timed out, which service was actually at fault? What is the service topology: what services connect to what other services and is that changing over time? What tasks are services spending the most time performing?

A relatively new CNCF project, [Jaeger](https://www.jaegertracing.io/), has arrived to dramatically improve operator joy in this age of microservices. After adding a small client to your service, Jaeger will collect information about all requests and tie them together in a slick UI. Services can also create spans for connecting to databases or other blocking or long running work which becomes invaluable when investigating intermittent problems.

Like logging, this tool's best use is understanding individual issues, yet has some handy features like generating service connection topologies to help you understand you cluster just a little better.

### Service mesh

To cap off our hierarchy of cluster needs, we have the Service Mesh. The current king is [Istio](https://istio.io/) + [envoy](https://www.envoyproxy.io/). This pair implements traffic management, service identity and security policies, and provides service telemetry. With it, you can:

* perform true canary deploys - where the canary is only accessible to operational staff
* easily provide TLS everywhere
* easily implement distributed service idioms like circuit breakers, retry budgets
* see Jaeger like telemetry in aggregate

## Epilog

That looks like a lot, and it is. Depending on migrating team size and competing responsibilities, this will take a year or more to implement. Doing it right is worth the time because you are creating the foundation that your org will use for the next decade.












