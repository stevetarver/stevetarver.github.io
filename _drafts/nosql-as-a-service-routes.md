---
layout: post
title:  "NoSQL as a Service (routes)"
date:   2015-11-16
categories: Orchestrate database db DBaaS NoSQL node.js routes
---

<img style="float: left;" src="/images/orchestrate-spin-logo.gif">


Orchestrate has some pretty interesting features; cobbled together from different database systems and unified through a ReST API. 

A logical first step is to create a microservice around our data.

Let's try that with strongloop

Why strongloop?

* swagger
* deployer
* metrics
* tracing
* profiler

Install loopback: https://strongloop.com/get-started/

We'll hook our Orchestrate calls directly into the routes for simplicity and clarity - and because orchestrate does not have a loopback connector yet.

```
[starver@anjaneya makara 10:58:45]$ slc loopback

     _-----_
    |       |    .--------------------------.
    |--(o)--|    |  Let's create a LoopBack |
   `---------´   |       application!       |
    ( _´U`_ )    '--------------------------'
    /___A___\    
     |  ~  |     
   __'.___.'__   
 ´   `  |° ´ Y ` 

? What's the name of your application? example-orchestrate-strongloop
? Enter name of the directory to contain the project: example-orchestrate-strongloop
   create example-orchestrate-strongloop/
     info change the working directory to example-orchestrate-strongloop
```

Now install the orchestrate driver

```
cd example-orchestrate-strongloop
npm install --save --save-exact orchestrate
```

I don't actually remember what code goes where, so I will cheat a little and let ARC do it for me.

```
slc arc
```

Arc should launch in your default browser - log in with your Strongloop registration credentials.

Now I can jump into the API composer and generate a basic model

In general, I build the first models and connections with Arc and then evolve the code from there.



```
slc run
```


