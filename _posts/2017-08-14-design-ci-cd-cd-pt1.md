---
layout: post
title:  "Desigining CI-CD-CD Pt 1: Terms & Concepts"
date:   2017-08-14 11:11:11
tags:   ci cd build pipeline jenkins automation

---

**TL;DR** Define CI-CD-CD, and duties and responsibilities of stages.

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/jenkins.png">

Continuous Integration has long been a standard practice for Enterprise IT, Continuous Deployment has long made us feel inadequate because we don't do it, and the terms Continuous Integration, Delivery, and Deployment have long been misused and misunderstood. Today, I say "No More!" Let's understand each, their potential use, and make a conscious decision to use or avoid them.

## Clarifying terms

These terms are confusing. Continuous "Integration" could mean that you are integrating a product into a production environment, "delivery" could mean that you are delivering to a production environment, and "deployment" could mean that your are deploying to a production environment. We have to actually read industry literature to figure out that what the industry means by each, and then there is still dissent.

Let's try to build really short definitions to keep them straight, then discuss nuances:

1. **Continuous Integration**: build source on commit - source code is continuously integrated
2. **Continuous Delivery**: build deployment artifact on source build - product is continuously ready to deliver
3. **Continuous Deployment**: deploy product to production - product (value) is continuously deployed to customers

Think of each facility as a product investment that reduces deployment risk while increasing business agility - the abilty to rapidly deliver features. Automating each facility has a cost and pays a return. Furthermore, each facility has many facets to opt-in to based on what benefits you want to capture and how much investment your are able to make. Devels and Operators will usually want to implement everything because it simplifies daily life, while Business and Product people will usually only see overhead and try to limit time spent on these quality-focused items.

Let's dive a little deeper into definition to ensure we are all on the same page.

### Continuous Integration

Products are usually built by teams of developers. Even if you are religeously pairing, there are frequently multiple concurrent lines of development - different features in flight at the same time. Modern developers frequently develop in separate branches to insulate their work from other changes. A whole lot of change is going on, and those independent changes cause the source to diverge. 

We know that the sooner a bug is identified, the cheaper it is to fix: it is fresher in the developer's mind, and has not caused other bugs. This is the point of continuous integration: to pull together all source, build it, and test it, as early and frequently as possible to reduce the cost of fixing integration errors.

In modern build strategies, it is also common to perform unit tests, functional tests, and integration tests defined within a single code base. There are differing levels of sophistication - perhaps you go beyond mocking and standup a database with valid schema and data. The general limit is that you are not integrating multiple code bases and usually not communicating off the build server. Of course these are soft limits, each group builds the tools that make sense.

### Continuous Delivery

Each product is composed of one or more build artifacts and some packaging - perhaps a Java jar with some configuration bundled in a Docker image. Continuous Delivery is the process of pulling those items together and producing the deployment artifact. It usually proves this artifact by deploying it to a pre-production facility and conducting manual or automated testing.

Frequently, this pre-prod facility gives other groups an opportunity to perform some validation or to start integrating with new or modified apis.

### Continuous Deployment

Continuous Deployment is usually implemented in the form of a pipeline triggered by the presence of a new deployment artifact. The pipeline walks the artifact through a guarded deployment, performs some validation testing, and on success, and perhaps a manual approval, releases the product to production.

Few companies are ready to continually release to production or can afford this level of investment. Especially in the Enterprise IT world, continuous deployment includes the notion that each successful build is ready to deploy to production, in contrast to actually doing it.

### Enough with these stupid terms!

While composing the paragraphs above, I recounted the times I have given a talk, or training, and had to explain these terms, clarify these terms, or correct people on these terms so we were all using the same terms to mean the same things.

It is absurd that I have to tell a woman, or man, working in the field, what to call their daily chores. I am officially opting out, not gonna use these terms, and you are invited to give me a slap if I do!

From now on, I am calling these combined facilities a **Pipeline** composed of a **Build Phase**, a **Deployment Phase**, and a **Release Phase** with these simple definitions:

Pipeline phases:

* **Build**: build and locally test the source code, package and archive the deployable unit
* **Deploy**: deploy a product to some pre-prod facility and integration test it 
* **Release**: make the product available in production


## Designing your pipeline

Pipeline phases are composed of stages. Although each product will go through each phase on its way to production, it may opt out of some of the stages for various reasons. Let's take a minute to walk through all the possible stages and what activities happen in each - these will become the building blocks for designing pipelines.

* Build phase
    * compile
    * test
    * package
    * archive
* Deploy phase
    * select
    * deploy
    * integration test
    * rollback
* Release phase
    * select
    * deploy
    * smoke test
    * approve
    * release
    * rollback

### Cross-cutting concerns

These two facilities should be available to all stages.

**Team notifications**:

You should spend a little time configuring your pipeline for easy integration into your team's communication tool (Slack, HipChat, etc.) to keep everyone aware of automation activities and when the pipeline needs a little love. In general, your pipeline will notify:

* On every stage failure, notify the owning team and the person that broke the build
* On every phase success, notify the team
* On every deploy, notify interested parties: the owning team, the business side, client partners

**Auditing**:

All compliance certifications include the notion of: tracking who changed a product, who moved that change into production. If you have compliance concerns or certification is in your future, you should invest a little in an audit facility. Even if you don't, you will need this information to fix broken processes when things go wrong.

What should be audited?

* Source code changes are audited through the revision control system
* Automated product builds are triggered through revision control system changes and thus audited by the revision control system
* Manual product builds must be audited through your build system
* Automated product deploys are triggered through automated build testing successes and thus audited by the revision control system 
* Manual product deploys must be audited through your deploy system
* Automated releases are triggered through automated product deploy testing successes and thus audited by the revision control system 
* Manual releases must have audit log entries for:
    * Intention to deploy
    * Deployment unit selection
    * Approval to continue after first increment deploy (canary test)
    * Approval after complete production deploy

### Build - compile

_Primary activity:_ produce an execuatable image from source code

This could be used as a first indicator of integration failure - answers the question "Will the current set of changes compile?" Although it adds a sense of completeness to the pipeline diagrams, I have started omitting it. Usually, we will have an instrumented build in the test stage and an optimized build in the package stage - a compile stage adds more build server load and execution time with no real benefit.

### Build - test

_Primary activity:_ execute all local testing

This should be one of the two longest running stages; your testing should be exhaustive. Our goal is to produce high quality code as rapidly as possible. This implies automating everything and catching errors as soon as possible in the SDLC. The more testing we can do, the more bugs we will catch and the earlier we will catch them. This becomes more important as the codebase grows and the code becomes more complex.

This is also a great place to enable checking of, or enforce, corporate quality standards by publishing testing reports or failing builds that don't meet certain criteria.

_Types of testing:_

* Unit, functional, local integration testing
* Source linting
* Static code analysis using [various tools](https://github.com/mre/awesome-static-analysis)
* Instrumented build for code coverage and/or performance evaluation
* Source security scanning
* Dependency security scanning

### Build - package

_Primary activity:_ build a deployment unit from source

_Other activities:_

* package validation
* package security scanning
* package size growth trending/monitoring

### Build - archive

_Primary activity:_ store the deployment unit in the central repository

_Other activities:_

* central repository image security scanning

### Deploy - select

_Primary activity:_ identify a deployment unit and target environment

Deployments to pre-production platforms may be triggered by the presence of a new deployment unit in the archive, a successful build, or be manually initiated. We need a selection mechanism for manual deployments.

### Deploy - deploy

_Primary activity:_ install the deployment unit in the target environment

_Other activities:_

* include environment specific facts (configuration)
* include environment specific secrets


### Deploy - integration test

_Primary activity:_ execute all product testing in a production-like environment

This should be the other of the two longest stages; your testing should be exhaustive. For microservices, integration testing is freqently more important than local testing because it is much more likely to break a contract or discover that some client depended on a side effect - the developer can't test for this on his laptop.

Having pre-production environments that mimic production is key to success here. If there are differences, it is highly likely that you will eventually ship a bug that was missed due to these differences. It is worth investing in automation that can refresh these environments from production periodically.

_Types of testing:_

* black box api invocation
* automated gui testing
* manual gui testing
* performance and load testing
* production-like monitoring and alerting
* randomized testing
* [InSpec](https://www.inspec.io/) compliance as code validation

### Deploy - rollback

_Primary activity:_ return the pre-production environment to a known good state

Investing in an automated rollback stage minimizes platform downtime and impact to teams that depend on a working pre-production platform. If deployments are designed well, this could be as simple as a re-deploy of the previous deployment artifact.

### Release - select

_Primary activity:_ identify a deployment unit and target environment

A selection may be triggered by a pre-production deployment that passes all tests, but more commonly is manually initiated through a manual product version selection.

_Other activities:_

* public announcement of maintenance period start
* internal announcement of maintenance period start
* prepare monitoring systems for expected service bounces

### Release - deploy

_Primary activity:_ install the deployment unit in the target environment using an appropriate deployment strategy

A production deployment can be as simple as placing all deployment units in production or a complex canary, red-black, or other strategy because an organization wants to avoid downtime and customer impact. This stage will be the most heavily customized for an organization, and may be different between different groups due to compliance concerns, etc. 

_Other activities:_

* include environment specific facts (configuration)
* include environment specific secrets
* [InSpec](https://www.inspec.io/) compliance as code validation
* internal announcement of new product availability
* internal announcement for interested parties to commence testing


### Release - smoke test

_Primary activity:_ execute basic testing on the production environment to ensure proper operation

Ideally, the entire platform should have a validation test that runs periodically to ensure all major functionality is available. If customer satisfaction is important, this is worth its weight in gold and will likely, continually, shake cockroaches from their nests - identifying opportunities for monitoring and alerting improvements. Additionally, each subsystem should have a similar type of test.

In addition to the smoke test, you can benefit from using [InSpec](https://www.inspec.io/) to verify that your deployment system did what you intended for it to do.

### Release - approve

_Primary activity:_ manually approve production deployment

Most compliance includes the notion of gatekeepers to production changes. Having an approval step is a great idea to keep people from becoming too informal and, if any type of compliance is in your future.

### Release - release

_Primary activity:_ complete product installation in production

This stage is included any time there is a multi-step deploy process and completes the production deployment.

_Other activities:_

* [InSpec](https://www.inspec.io/) compliance as code validation
* final production testing
* internal announcement of maintenance period end
* public announcement of maintenance period end
* release announcements to public
* publish release notes

### Release - rollback

_Primary activity:_ return the production environment to a known good state

At any point in the release phase, after the deploy stage, this stage restores  the product to a known good state. This includes recovering for bugs discovered a week after release that are high enough severity and cannot be fixed with a less dramatic operation.


## Epilog

Now we have a fairly extensive list of pipeline building blocks, we can start thinking about how to piece them together to make a custom system for our group or organization.
