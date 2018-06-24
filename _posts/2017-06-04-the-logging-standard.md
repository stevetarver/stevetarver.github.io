---
layout: post
title:  "The logging standard"
date:   2017-06-04 11:11:11
tags:   log logging monitoring standard

---

**TL;DR** Consolidation of previous logging articles into a more concise, implementable standard.

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/logging.png">

I've written a lot about logging because it provides our fundamental notion of how our service is doing in production: how to remediate issues, how the service is doing at a platform level, and rudimentary service metrics in the absence of a more sophisticated monitoring system. 

After living with those decisions for a while, it's time to create a minimal, yet robust and portable logging standard for our Kubernetes platform. Background for this approach can be found in:

* [Logging as a First Class Citizen](http://stevetarver.github.io/2016/01/09/logging-best-practices.html)
* [A whole product concern logging implementation](http://stevetarver.github.io/2016/04/20/whole-product-logging.html)
* [A logging aspect in java](http://stevetarver.github.io/2016/06/12/java-logging-aspect.html)
* [Structured logging in Python](http://stevetarver.github.io/2017/05/10/python-falcon-logging.html)

But, before we define the standard, let's talk through the overlap between logging and monitoring.

## Differentiating Logging from Monitoring

In the past, logging has been a fundamental service metrics collector - is it still the right tool for the job? 

Modern cloud architectures provide services on top of a volatile fleet of servers. Modern monitoring systems are able to automatically discover and monitor services and servers, and consequently, some metrics traditionally provided by logging are more simply captured by monitoring components and more conveniently grouped with associated server metrics. More specifically, monitoring tools focus on numbers, consolidate your numbers in one place, have data storage and a query language built for numbers, simplify visualizations built for numbers, include facilities like alerting triggered by simple rules. Furthermore, monitoring strategies, concepts, and implementations are sharable across the industry, because numbers and metrics are common across the industry - they aren't polluted by one company's domain languge include in logging text or key names.

Logging and monitoring are fundamentally different beasts: logging should be optimized for text and key-value pairs and monitoring should be optimized for numbers. So, let's back up a bit and compare the fundamental nature of logging and monitoring.

* Logging provides a diary of service activity, where monitoring quantifies service behavior. 
* Logging is good at telling you how a customer's specific request failed, where monitoring is good at telling the percantage of that request type returned a 500.
* Operators use logging to solve specific customer problems, and use monitoring to solve system problems.
* Logs are retained for a long period (60 days?) for SLA obligations and long-running problem research. Monitoring data is kept for a short period (10 days) for solving immediate problems.

Creating logging and monitoring strategies are fundamentally different as well. Both start with questions but logging questions focus on what was requested, why choices were made, and what was the result. Monitoring questions revolve around tasks in aggregate, like how many requests were made, how many failures happened, what types of failures, as well as resource questions like resource utilization.

Since we will provide both logging and monitoring facilities to our Kubernetes clients, we will focus each on their strong points and try to minimize overlap.

Now, let's look at how logging works and how clients should use it to provide a consistent operational plane.

## Log entry flow

In our Kubernetes environment:

1. Service code logs well-formed json using a standard keyword dictionary to stdout
2. A daemon set gathers all service log entries
3. The daemon set grooms and forwards log entries to the log aggregation system

## When to log

* Every API request receipt, including parameters so the API call can be replicated during debugging
* Every significant branching decision, including data used to make that decision so execution can be traced logically through the code
* Every call to an external service including call and return parameters if that service is not already logging them
* Unexpected data and situations that require further analysis
* Every API request response, normal and exceptional, including response code, call duration, and exception messages so you can easily identify long running calls and error frequency by type

## What to log

Each log entry should include

* The 'x-request-id` header. Allows a single keyword search to identify all log entries made while attempting this task
* The distributed tracing id header. Allows finding the Jaeger trace from any log entry.
* Required boilerplate key-value pairs (described below)
* A narrative message
* Optional key-value pairs to clarify branching decisions or provide a group-by clause for ELK queries.

When viewing log messages associated with a single task, those messages should read like a diary of all significant actions taken to accomplish that task. E.g.

```
Contact change requested for bob.martin
Account bmc.helpdesk is authorized to make this change
Contact change request is valid
Contact change request is different than existing contact record
Contact record updated
Contact email has changed, sending email address verification to bob@bobmartin.com
```

Note that the narrative is not polluted by object dumps, extraneous data - that is included in log record key-values pairs and easily excluded from the narrative.

Key-value pairs

* Improve the readability of narrative messages by moving excessive detail to other parts of the log entry
* Are written as JSON so they are easily copied into other tools to reproduce api calls
* Act as columns in a SQL group by clause, allowing you to aggregate calls by these columns for metrics in ELK queries

Sensitive information like passwords, SSNs, etc., should be masked in log entries as part of the logging implementation so they are not present in the docker container logs.

## Field definitions

**NOTES**: 

* All boilerplate logging keys have a "log" prefix to avoid key name collisions with ad hoc key-value pairs used in service code.
* Each log entry should contain a "logMessage" field.
* Duration key names should have a suffix that indicate precision: durationMicros, durationMillis, durationSecs.

### Identify the Request in all collaborating services

**GOAL**: Identify the client request that triggered log entry so operators can easily correlate a ticket to a set of log entries.

A single unique identifier is added to a client request whether generated from the UI or an API entry point. That id follows the task through every service call used to complete the task - if an id does not exist, generate it, if it does, propagate it to peer services. Inclusion of this id in every log message allows us to identify the request from any piece of information a customer ticket provides through novel ad-hoc ELK queries, and then see all log entries related to that request through a simple ELK keyword search.

ReST APIs will include this as an `x-request-id`. When one service calls another, this header should be forwarded. When messages are queued, the `x-request-id` should be recorded in metadata so the message consumer can use it in their log messages. 

This key must be included in every log entry.

| Key | Type | Description |
|-----|------|-------------|
| **logRequestId** | string | `x-request-id` header, generate a UUID4 if missing |

### Identify the service deployment

**GOAL**: Identify the Kubernetes component that made the log entry so operators can quickly check monitoring for that component.

The following entries provide a coordinate system to locate the instance of code that created the log entry. These entries should be configured as boilerplate entries for your logging system and included in each log entry. Separate key-value pairs are used for convenient grouping by each level in the hierarchy.

These keys must be included in every log entry.

| Key | Type | Description |
|-----|------|-------------|
| **location**|         string | a three letter datacenter identifier|
| **logServiceType** |     string | a unique name for the service category. e.g. "network", "vmware", "blueprints", "billing". This should be the same for all services in this domain |
| **logServiceName** |     string | a unique name for this service. e.g. "nsxApi", "junosOrchestration"|
| **logServiceVersion** |  string | this service's docker image tag|
| **logServiceInstance** | string | a unique identifier for the code instance implementing the serviceType and serviceName. e.g. a Kubernetes pod id. |

### Identify the source code location

**GOAL**: Identify the source code repository so operators can quickly browse source to understand what the code attempted to do, differentiate infrastructure from code issues.

The following entries help locate the log point within the process, are included in each log entry, and in most cases can also be a boilerplate entry:

These keys should be included in every log entry, except logCategory which is only specified when used.

| Key | Type | Description |
|-----|------|-------------|
| **logGitHubRepoName** | string | identifies the GitHub repo for this code |
| **logThreadId** | string | a convenient grouping for task execution (in thread per api call) or functional area (thread pool). Experience shows this grouping can reveal an otherwise hidden class of problems.|
| **logClassName** | string | identifies the class that made the entry |
| **logMethodName** | string | identifies the class method, or function, that made the entry |
| **logCategory** | string | an optional context to better understand the log entry. Currently used with Telemetry e.g. apiRequest, apiResponse, apiException. These should be standard across all services|

### Identify when the log entry was made

**GOAL**: Provide a timestamp in the form Analytics requires that is easily parseable by ELK.

ELK is picky about timestamp format and in the [ELK as a Service Onboarding page on x.clt.io](https://x.ctl.io/teams/0e0dff97f710efa3/wikis/0f10a5d9b1108a81), Analytics requires the following format listed.

| Key | Type | Description |
|-----|------|-------------|
| **@timestamp** | string | ISO8601 format with millisecond accuracy: `2017-12-10T11:10:38.192Z` |

**NOTE**: ELK requires the field name to be `@timestamp` to be recognized as the official log entry timestamp.

### Identify log entry severity

**GOAL**: Provide indication of log entry importance so operators can focus on those indicating problems.

Some logging systems offer an abundance of log levels. When dealing with many instances of many services implemented in several languages, we prefer a minimal, semantically meaningful set:

* **DEBUG** is for excessive detail targeted at resolving complex, possibly opaque areas
* **INFO** is for API boundary entry and exit. log the inbound parameters and headers (Referer, User-Agent, X-Forwarded-For, as well as meaningful custom headers) so if there is a bug, you can easily reproduce it. On exit, log the call duration and status. INFO is also used for narrative entries that should read like a book describing all significant actions or branching decicisions taken during the execution of a task. You should also log calls to other modules.
* **WARN** for conditions that should be looked at but did not cause task failure
* **ERROR** for failed tasks

| Key | Type | Description |
|-----|------|-------------|
| **logLevel** | string | One of DEBUG, INFO, WARN, ERROR |

**Note**: Default log level for all services is INFO.

### Telemetry

**GOAL**: Record inbound parameters, status code and duration so operators can replicate problem calls and we can generate failure and timing dashboards to indicate system problems.

Telemetry tells us about the request, in contrast to describing request processing. In addition to standard entries described above, on request receipt:

| Key | Type | Description |
|-----|------|-------------|
| **logCategory** | string | "apiRequest"|
| **logMessage**  | string | "{resource} requested" |
| **reqOrigin**   | string | caller - best effort to get actual caller |
| **reqVerb**     | string | uppercase http verb name, e.g. GET, POST, etc. |
| **reqPath**     | string | path portion of the request URL (not including query string) |
| **reqQuery**    | string | query string portion of the request URL, without the preceding '?' character |
| **reqBody**     | json   | the request body |

In addition to standard entries described above, on request response:

| Key | Type | Description |
|-----|------|-------------|
| **logCategory**       | string | "apiResponse"|
| **logMessage**        | string | "{resource} request completed" |
| **reqStatusCode**     | int    | http status code as a number|
| **reqDurationMicros** | int    | number of microseconds elapsed during request execution |

## What the framework should log for you

Many languages and ReST frameworks have support for cross-cutting concerns or middlewares. These facilities can eliminate boilerplate logging like API request and response and ensure the entries are always made.


## Implementations

The Microservice Reference implementations ([stevetarver/ms-ref-* repos](https://github.com/stevetarver?utf8=%E2%9C%93&tab=repositories&q=ms-ref&type=&language=)) provide implementations and example use of this standard.

* java
* python
* go
* groovy

