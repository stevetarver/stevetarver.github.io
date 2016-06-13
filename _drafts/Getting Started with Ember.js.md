# Getting Started with Ember.js


## Ember elements

Ember.js contains the following types of things

- routes
- controllers
- components
- models
- templates

and they are related like this:

**TODO**: diagram of relationships


## Naming from db to client

ref [json:api format](http://jsonapi.org/format/)

GOAL: Build a client that is closely related to the underlying data.

A database has notions of tables, columns, and relationships. You can bubble up that information in Strongloop as classes, attributes, and relations. json:api is presenting that information for the default Ember.js ReST adapter and serializer. ember-data let's you specify that information, yet again through models attributes and `DS.*` types.

You can see that the notion of an ORM is present in the server API and relations are conveyed to the client via json:api so Ember.js can define model relations and map response data to models. 

This is VERY COOL because Ember.js has enough information about our models and our incoming data that it can make a lot of assumptions and eliminate a lot of code. This replicating relationships in each layer (db, server, client) seems like a lot of extra work, but when you see how much code is eliminated from your projects, you will see the value.

There is one VERY BIG gotcha, and that is naming. It is actually very small, but if you have this problem, it will eat up a lot of time figuring out, and you will forget the solution between occurrences. So let's identify the simple rules in one place that we can refer to.

Basically, json:api wants to be universally 

The Problem: In code, multiple word class, instance, or attribute names are usually represented as proper-case or camel-case. 

First, let's look at the json:api structure so we can talk about where

In StrongLoop, when you reverse engineer a database, '-', '_' will be removed, words joined in camel-case. 

```
action_logs => ActionLogs
actionLogs  => ActionLogs
```

json:api wants all of its attributes and resource names 


## Pluralization

## Reserved Words


## Building the data pipeline

Steps from db to client: Strongloop Arc reveng, name changes, ember-data naming.


## Using json:api

Create a mapping layer on top of your database that exposes json:api. Ember.js can eliminate A LOT OF CODE if you take the time to do this.

Strongloop is a great solution for this because

- it has a json:api add-on to generate links and side load data
- it has Swagger built in so you can easily see what your api is producing

Steps:

- Use StrongLoop ARC to reverse engineer your database
- Hide API methods you don't want exposed
- Add relationships `slc loopback:relation`. This works well until your table count becomes large; then it takes a while to load. Generate a relation of each type to see how they work. Then you can manually add them much faster.

### json:api naming rules

Optimally, you design your Ember.js app first, using `ember-cli-mirage` to mock data. Then you know what table names and relations should look like.

If working from an existing database, your database will likely have names that don't map cleanly. You can fix that up in the `<table>.json` files so that you are producing valid json:api names.

json:api defines standards for converting from proper-case, camel-case, underscore-case to hyphenized names, but these rules are difficult to remember for links, relationships, included URLs, etc. It is far simpler to let class names start with an upper-case letter and down case everything else and accept standard pluralization. The gain readability for camel-case, or customizing the pluralization from 'persons' to people is not worth the time lost in remembering the rules.

`attributes` and `type` are reserved so you must rename them.

## Debugging the client side

### Ember Inspector

### Logging and breakpoints

ref: [Development Helpers](https://guides.emberjs.com/v2.4.0/templates/development-helpers/)

You can log to the console in your html with

```
{{log 'Name is:' name}}
```

And break into the debugger with

```
{{debugger}}
```

