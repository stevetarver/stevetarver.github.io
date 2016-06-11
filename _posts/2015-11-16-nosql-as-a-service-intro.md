---
layout: post
title:  "NoSQL as a Service (intro)"
date:   2015-11-16
tags: Orchestrate database db DBaaS NoSQL geospatial graph 
---

<img style="float: left;" src="/images/orchestrate-spin-logo.gif">

In the good ol' Enterprise days, when you wanted to POC a new application or service, you would fill out the paperwork and, if you were a favorite son, have a new server to play with in just 5 months.

Then private virtualization came to the Enterprise and you could get a new VM in about a month.

Then, your company committed to valuing Business Agility and paid for a place in the public cloud, providing accounts in AWS, Google, Azure, Rackspace, CenturyLink Cloud, etc. Now, you can have an idea in the morning, provision some VMs in a few minutes, and start playing with that idea immediately.

I'm greedy. What's next? How can I get my POC in front of people even faster?

**TL;DR**: _Introduction to the Orchestrate.io DBaaS_

Data stores are the cornerstone of every application/service ... What if I could just skip database deployment and configuration, have a simplified admin, and just start using it?

That's the promise of a DBaaS like [Orchestrate.io](https://orchestrate.io) which provides much of the common NoSQL functionality over a ReST API. Most importantly, it provides a free pricing tier - perfect for this experiment.

I want to see what one of these services can do, how easy they are to work with, and how it can fit into my developer toolbox. In this article, I am going to scour the docs, pick out what I find interesting, and try to produce a concise, yet meaningful overview. Future articles will explore the API over some common uses.

**Disclaimer**: I am a CenturyLink Cloud employee and CenturyLink acquired Orchestrate.io on 20 APR 2015. 

## What does it do?

Orchestrate organizes JSON objects and search in common use cases

- Basic JSON object storage with Elasticsearch queries
- Time ordered events
- Geospatial data with bounding box and radius queries
- Graph relationships

All interactions with data are through the ReST API; minimal management is available through a dashboard.

## How is it implemented?

Behind the scenes, Orchestrate is multi-tenant database built on a collection of database technologies

- Apache Hadoop
- Apache HBase
- Apache Zookeeper
- ElasticSearch
- MySQL
- Apache Kafka

A complete list of technologies they use is [here](http://orchestrate.uservoice.com/knowledgebase/articles/354377-what-software-and-services-underpin-orchestrate)

Data consistency is determined by the database technology your API is using and is either strongly consistent:

- All key-value operations
- Event
- Graph

or eventually consistent:

- Search
- Geospatial

due to indexing delays required for this type of data. E.g. If you write 1500 documents in a bulk write, it will take a few hundred milliseconds for that data to be available via search.

## Where is the service hosted?

Orchestrate was built on AWS, will continue to support existing customers on AWS, but is expanding through the CenturyLink data centers. Current locations are:

- Amazon US East - Virginia
- Amazon EU West - Ireland
- CenturyLink NY1 - US East (New York)
- CenturyLink VA1 - US East (Sterling VA)
- CenturyLink UC1 - US West (Santa Clara CA)
- CenturyLink GB3 - Great Britain (Slough)
- CenturyLink SG1 - APAC (Singapore)

This [status page](http://status.orchestrate.io/) shows uptime for each data center for day/week/month and allows you to subscribe to updates.

## Did you say free?

When you register with Orchestrate, you are automatically enrolled in the Free Tier which requires no credit card and allows

- 1 Application (database)
- 50K API calls (monthly)
- 50 MB writes (monthly)

If you exceed your API call monthly limit, your account is suspended until you upgrade or the next monthly billing cycle.

Pricing is presented as following application evolution. The free tier allows experimentation and development. When you are ready to go to production, and your volume exceeds the 50K operations/month, or you want to separate your production and development data, you can upgrade to the Developer plan at $49/month. When your application use, or operational needs demand it, you can upgrade to the Professional tier ($500/month). There is an additional Enterprise level for full customization, including options like private cloud deployments, dedicated clusters, warranty and indemnification.

## How is data replicated, backed up, imported, and exported?

During normal operation:

> Data is replicated 3x across secure, distributed HBase storage engines for resilience and high availability. Daily backups to S3 or your preferred storage account are automatic, or available on demand. 

Note that replication is not accross data centers - it is currently 1 DC per app. Multiple DC replication is an outstanding feature request.

There are several ways to import data:

- RDBMS: The JDBC-importer for MySQL, PostgreSQL, MSSQL, and Oracle allows you to  write a select for the RDBMS, point at an orchestrate collection, and fill 'er up.
- NoSQL: The Node.js packages [orchestrate-mongo](https://www.npmjs.com/package/orchestrate-mongo) and [orchestrate-couchdb](https://www.npmjs.com/package/orchestrate-couchdb) can be run as command line or daemon.
- CSV: The Node.js package [orc-csv](https://www.npmjs.com/package/orc-csv) allows CSV imports.

There is also a bash script that uses the bulk write API. If you have > 1 million documents, Orchestrate suggests opening a support ticket for help with the import. There is an outstanding feature request to provide import from an S3 bucket or the dashboard via file upload.

You can migrate data between data centers with a support ticket.

Basic data exports can be done from the dashboard. Once the button is pressed, the data is gzipped and emailed to you. Obvious file size limits apply.

## How do I interact with my data?

Once you create your Application (database), you will rarely use the dashboard; all data interaction is done via ReST API. This includes creating new collections by simply PUTting data in them.

Hard-core shell people you can use cUrl to communicate with Orchestrate - it's just ReST.

There is a Node.js command line tool, [orcli](https://www.npmjs.com/package/orcli), that integrates very well with the shell and simplifies everything. orcli is really handy for interactive and batch db maintenance chores. 

**TIP**: see [jq](https://stedolan.github.io/jq/) for reshaping JSON documents on the command line.

There are several client libraries to simplify ReST use:

- Orchestrate libraries: Node.js, Java, Ruby, Go, Python
- Orchestrate experimental libraries: Erlang
- Community libraries: PHP, .NET, Scala, Haskell

There are also data adapters on [npm](https://www.npmjs.com/search?q=orchestrate) for

- Hapi.js
- Sails.js
- Ember.js

## Other notables?

>All Key/Value items in Orchestrate have a version history, a list of all the changes to the item over time.

>Data in Orchestrate is immutable and the majority of operations are non-destructive. When updating a Key/Value item, Orchestrate creates a new version (instead of overwriting the original), and adds it to the version history.

>This non-destructive behavior enables you to track changes, retrieve previous values of a Key/Value item, restore deleted values, or manage state when multiple actors are changing values in parellel.

All version history is available via API, down to the millisecond. This provides some pretty interesting opportunities, like preventing SQL style "write-unders" by validating that the user's version of the document matches the current version, and reverting to the user when their edited version is out of date.


## Limits

Some obvious limits stand out.

### HA/DR/Regional Presence

If you need presence in more than one data center, you are out of luck. Data does not replicate between data centers so something like MongoDB regional clusters is not possible. 

You could get something like regional sharding by standing up an application in each region, backed by a dedicated database, but any common data would have to make the long haul to the common database. 

It would seem that pricing would become a concern when you have multiple databases associated with one application, but Evident.io claims that 

>the cost of running MongoDB in EC2 was nearly 50% higher than using Orchestrate for our projects.


### Limited Administration

There is very little to the Admin dashboard. A lot of this need is eliminated by design by things like indexing all document keys so you don't have to worry about query index coverage. 

All of your application monitoring will have to be black box from the application/service instead of the more general performance analyzers for more general purpose NoSQL databases like MongoDB.

## Getting Started

1. Visit the [registration page](https://dashboard.orchestrate.io/users/register)
1. Confirm your email address in their registration email.
1. Sign in
1. In the orchestrate dashboard, create your application - give it a name and a home data center

That's it. That's all. Post to a collection using your API key and you have created your first collection.



