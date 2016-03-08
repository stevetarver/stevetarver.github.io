---
layout: post
title:  "Logging as a First Class Citizen"
date:   2016-01-09 13:22:05
categories: logging
---

When I started getting serious about programming, I was indoctrinated into leading edge development methods for X Windows and C like PDD: Printf Driven Development. While you were trying to figure out how your code worked, you would litter the code with printf statements, run it, read the log, revise the code. By convention, you left the printf's in place to simplify future feature additions and so you could find and resolve bugs if they were ever reported.

X Windows? C? Yeah, the world has changed a lot in 25 years. New languages, platforms, and even modern incarnations of ancient languages have amazing tooling, debugging, robust logging frameworks, and support BDD and TDD. In the Enterprise, BDD and TDD are pervasive and change the emphasis from "get 'r done" to improving how you can reason about your code. Moving to the cloud implies moving to small DevOps oriented teams and a whole product concern. BDD, TDD are essential methods for improving your ability to reason about your code, and their artifacts are used to prove initial code and that code's evolution. 

What role does logging play in this new environment?

**TL;DR** How to log with a Whole Product Concern


## What's up in production?

Our testing frameworks help us develop code and prove it on every commit via CI, and then prior to each release with CD. Debuggers reveal the inner workings of our code in unexpected situations. This eliminates some of the traditional needs for logging so the question remains: what is logging's role in a DevOps focused environment?

Enterprise wisdom tells us that 90% of the life of a product happens after initial release. While code is in a production environment, logging will be one of four ways we can evaluate the health of the product:

- Application/Service historical trending and application diaries produced in logs and aggregated by infrastructure like [Splunk](http://www.splunk.com/), [Graylog](https://www.graylog.org/), and the [ELK stack](https://www.elastic.co/products).
- Application/Service metrics generated with frameworks like [Spring Boot Actuator](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/) and [Strongloop](https://strongloop.com/), more metrics produced by cluster managers like [Kubernetes](http://kubernetes.io/), and all of that collected and displayed with products like [Prometheus](https://prometheus.io/).
- Execution environment monitoring with products like [Prometheus](https://prometheus.io/).
- Interprocess request tracing with products like [Zipkin](http://zipkin.io/)

There is some necessary overlap between `/metrics` and logging. `/metrics` provide a snapshot of your code with counters and guages, collected on some short period, and Prometheus provides a way to organize those snapshots in meaningful views. Log based metrics provide a more wholistic view of more granular information in a way that is easily aggregated and analyzed. 

For example, in `/metrics`, you may capture average call duration but how do you see only outliers; the one call that was 10x average, calculate 90th percentile after the fact. Providing this kind of post-mortem, ad hoc query, producing actionable analysis justifies the double data recording in `/metrics` and logging. If you are overly concerned with, or culpable for, product quality, you will learn this value.

When choosing one of several logging frameworks for your environment, the key factor is how easily you can produce log entries that are efficiently and effectively ingested by your logging aggregator.

Once that choice is made, the focus turns to what should be logged.

## Narrowing logging's purpose

Given the wealth of information aleady available from other systems, how can logging improve our knowledge of our production system? Logging's unique contribution addresses two operational concerns:

- Historical trending
- Issue remediation
Historical trending shows API call counts, durations, and response status. These three simple metrics can answer questions like

- Capacity planning: How do API call counts vary over time
  - How does application use vary during the day, week, month, quarter, seasonally  - What rate is my application use increasing
  - Do I need to scale for periodic peak use
  - When do I need to scale as application use increases
- Load based problem analysis
  - Do some problems only occur under load
  - What services are my weakest links and need some love
  - What interdependencies exist that contribute to load based problems- Performance analysis
  - Does call duration increase under load
  - What load should benchmarks and load testing simulate
  - What code should change to improve UX

Narrative log entries read like a diary of actions taken to accomplish a task. These are vital to reproducing production issues in a development environment and rapid remediation. Some key goals for these types of entries are:

- Log API request params so you can replicate an API call locally
- Log significant branching decisions and values used to make those decisions
- Log out of process calls. If that process is not monitored by your log aggregator, request params, call duration, and response status should be logged to provide a more complete task view.

## What should I be logging?

Logging should be minimal, yet meet the above goals. 

If you are paying for a logging service like [Loggly](https://www.loggly.com/), or infrastructure like Splunk, those costs are based on how much you log. Minimal logging can save you some money.

No matter what log aggregator you use, there is a cap on how much disk you can use. Minimal logging lets you keep logs for a longer time. Want a year's worth of historical trending? Minimize your logs.

Then there is the question of clarity. When you are under the gun to resolve a production issue, you don't want to have to wade through a lot of clutter to find what is important.
Distilling logging goals above, this is what you should be logging:

- On API entry, log request and header parameters
- On API exit, log call duration and status
- Log every branching decision and the data used to make that decision
- Log calls to other service and capture parameters, call duration, and status if that service does not log to your aggregator
- Log other data not captured in `/metrics` that may result in actionable analysis

See below for framework generated log data - essential tagging for queries that you won't have to add to your log messages.
## Log with the aggregator in mind

All aggregators require JSON formatted data to effectively ingest log messages as fields. Some may claim they can parse XML but that never works well. Some will talk about extracting values from narrative messages, and while true, it is so inefficient as to be unusable with larger datasets. What happens? Spinning wheel of death when you apply a regex to a month of logs. 

Tell you more about inefficiently encoded data? On Splunk, you define the number of concurrent jobs permitted, defaults to 8. If you designed a dashboard based on regex extracted values, and you have a production problem, and 8 managers open the dashboard, you just locked every engineer out of doing real work with Splunk. And not just your team; every team that uses that Splunk repo. Sure you could increase the concurrent job count, but that just defers the inevitable. The point is that you are doing something so ineffeciently that you WILL choke the system. Not bashing Splunk here - it is my favorite log aggregator so far. So, on the importance of JSON encoded log messages: Got it? JSON formatted data GOOD. Extracting values from strings BAD.

After you have all logging statements as key-value pairs, there is still a little more work to do. If you think back to the kinds of questions we want to answer, how would you write a query for "Show call counts for all API methods by name"? This is a great basic query. Your log aggregator can produce a pie chart from it showing heaviest use and hinting at who should be optimized first. It can also provide time rollups and show how comparative values vary over time. In Splunk, with well formatted data, I can write an eight word query and produce this dashboard in 6 clicks.

<img style="float: center;" src="/images/api_call_count_by_method.png">


The key to writing this query is: how do I identify API method names and how do I identify API entry points. 

1. API method names should be logged by your logging framework
1. Use tags to allow you to 'group by'In the past, I have used a tag `executionPoint=request` to mark an API entry point and `executionPoint=response` to mark an exit. This small addition allows the log aggregator to exclude all irrelevant log entries and drammatically improves performance.

## What should the framework be logging?

Just for completeness, let's explore what your logging framework should be logging for you and why it is important.

Log4j is pretty popular in Java land, so let's use it as an example. Log4J wants you to provide a logging pattern

`%d{yyyy-MM-dd'T'HH:mm:ss.SSSZZZ} level=%-5p, thread_id=%t, cat=%c, class=%C, method=%M, %m`

Note that these are all comma delimited, key-value pairs except the first and last. 

The first entry is special, it is an ISO8601 compliant date-time format including timezone. Using this format allows you to effectively integrate logs across the globe and your log aggregator bears the responsibility of rendering that time in the local zone and format.

The `level` entry should include things like DEBUG, INFO, WARN, ERROR and allows you to turn up or down logging granularity to support issue remediation. Every environment should log at INFO level: these are your narrative diary entries. The WARN and ERROR tags make it easy to find suspicious entries, things that need attention. DEBUG entries are for emergencies; when you can't reproduce the problem in DEV and can't understand what is going on in PRODUCTION.

The `thread_id` entry can reveal a different side of the story in thread based systems like Java. With it, I can locate the thread id with data provided in a bug report and then show all actions that occurred on that thread. 

The `cat` or category entry allows you to provide vertically or horizontally sliced views of your code; assign a tag to files or loggers. For example, you could tag all of your data access objects as `dao` and easily generate queries that focus on only those entries. In the Java world, I have never seen anyone take advantage of this; it is always identical to `class`. 

`class` and `method` are a coordinate system allowing you to quickly locate the logged line of code. They also allow efficient queries when you want a horizontal slice (across all threads) of all method invocations.

Concrete example:

```
2014-09-04T19:18:55.656+0000 level=INFO, thread_id='http-51101-6', cat=RestResource, class=com.github.stevetarver.rest.impl.ContactResourceImpl, method=getContact executionPoint=request, id=0050568A-011D-11E3-E37D-C03336F01F4C, message='Getting contact 0050568A-011D-11E3-E37D-C03336F01F4C"
```

All log aggregators, Splunk, Graylog2, ELK, as well as logging services will parse logs more efficiently, effectively, and meaningfully if you log in JSON: your logging framework needs to emit JSON. At a minimum, it needs to log in the format above - almost-json.


## What do log entries look like in code?

Guidance: Log in JSON, log parameters first, and then provide narrative messages.

Create a method that generates an appropriate string representation for every object. In JavaScript, this is pretty straight-forward. In Java, the easiest solution is probably to use `Guava` in your toString() override. In go, go-kit provides a perfect implemenation for logging.

Concrete Java example

```java
LOG.info("'executionPoint': 'request', contact: " + contact + ", message: 'Creating new contact'");
```

## 12 factor guidance

I have a lotta love for Twelve-Factor Apps. I have experienced all the pain points they describe and get all the guidance they provide. Here is what Heroku says about [Logging](http://12factor.net/logs).

Briefly:

- log to stdout and let the execution framework send it to the log aggregator

And this is what Steve says:

- log to persistent storage for reliability - kafka, rabbitmq, etc. Your log aggregator reads from your persistent store.
- use a log guid to track operations from app to middleware and then among collaborating microservices
- use an environment trace monitor like zipkin if you can


## Mask sensitive information

Got passwords, SSNs, other sensitive data? Create a logstash filter in your execution environment to strip these out before recording in your log aggregator.


## More on levels

Some systems have an abundance of logging levels. At some point this made sense, but in modern 12-factor apps, not so much. IMHO, this is what you need.


- DEBUG is for excessive detail targeted at resolving complex, possibly opaque areas
- INFO is for API boundary entry and exit. log the inbound parameters and headers (Referer, User-Agent, X-Forwarded-For, as well as meaningful custom headers) so if there is a bug, you can easily reproduce it. On exit, log the call duration and status. INFO is also used for narrative entries that should read like a book describing all significant actions or branching decicisions taken during the execution of a task. You should also log calls to other modules.
- WARN for conditions that should be looked at but did not cause task failure
- ERROR for failed tasks



## Security

Log renderers are not immune from security breaches, forged logs, and XSS attacks. Here are some things to consider:

- CRLF injection via user supplied data: protect by ouput encoding using `ESAPI.encoder().encodeForHTML(<string>)`
- Log Forging Attack: If it can be proved that a log has been altered, then the entire log file is suspect and cannot be trusted. If a user can supply data that adds log entries, through injected CRLF, your log as a record has been invalidated.
- XSS attacks: If a user can embed code in your logs, and your log renderer presents that code, what happens when a user clicks on a generated link?
