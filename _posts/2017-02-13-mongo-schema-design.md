---
layout: post
title:  "NoSQL schema design"
date:   2017-02-13 13:25:16
tags:   mongo nosql schema design

---

**TL;DR** NoSQL schema design for common application access patterns

<img style="float: left;" src="/images/logo/mongo.jpeg">

I loved the Mongo University's M101 course I took a couple of years ago. They have added many courses that look interesting, DBA, clustering, security, etc., so I am taking M101 again as a refresher. Week 4 focuses on application access pattern based schema design. It is such a good overview that I wanted to have those ideas handy - these are notes from that week.

Have to put a plug in for Mongo - they do such a good job with their courses. Beyond learning about Mongo, these are good introductions to NoSQL database concepts in general. As a bonus, they teach assuming you are already familiar with relational concepts and provide a fair amount of contrast between the two. Armed with a knowledge of both, you can compare the benefits and challenges of each which helps picking a new data store for a new project. 


## Overview

Relational database design intentionally avoids serving specific application access patterns to provide a neutral pattern for all applications. This implies that no application is especially well served; the data is the product and applications are consumers of that core product. This may be the best datastore choice when you must have a lot of different, loosely coupled domains represented in a single store, or when you have different applications with completely different access patterns. 

If you have a single, or small number of apps using similar access patterns, the application can become the primary focus and the data store is simply persistence. For these cases, you may be able to reap the benefits of a NoSQL store. Your schema design will suit your application specifically while trying to **avoid** the situations that relational systems allow you to **prevent**. Specifically, relational databases provide two key facilities that protect data integrity that are core considerations in NoSQL schema design - let's review those first.

## Foreign key constraints in a NoSQL world

Foreign key constraints are used to ensure that logically associated data, divided into multiple tables, with references between tables, is guaranteed to exist.

In NoSQL, you can frequently embed data, or nest one document within another, to avoid question of whether associated data exists. In this case a foreign key constraint is simply not needed because the data was never separated, data that logically belongs together is stored together.

Some situations prevent you from embedding documents:

* high performance requirements: large documents take longer to deliver and eat mobile data plans for lunch
* document size: large embedded documents may violate a document size constraint
* disparate application access patterns: logically related, but loosely coupled data is partially used in different parts of the app; fetching the entire document when only a small piece is actually used reduces application performance

For these cases you must ensure data sanity in application code.

## Transactions in a NoSQL world

In a relational database, transactions are used to ensure that an update spanning tables is applied as a single change.

In the NoSQL world, embedding data effectively pre-joins data that would exist in different tables in the relational world. Document updates are atomic so embedding data effectively gives you a transaction around this pre-joined data.

As an example, in the relational world you might have a separate table for person, address, and contact information and may need a transaction if applying a change that affected all three. In the NoSQL world, you could embed the address and contact document in the person document and the atomicity of a document write will provide the same benefits of a relational transaction.

For situations where you cannot embed data, you choose schemes to minimize the chance of data inconsistencies, tolerate those that remain, or implement some solution in code to avoid data update anomalies.

## Common considerations

A large part of schema design revolves around modeling data relationships. Before we dive into that, let's look at some of the constraints and considerations we must keep in mind and tools we have to work with.

### Performance considerations

* **16MB doc size limit**: this is a Mongo specific limit, Couchbase is 20MB; there will be some document size limit in each NoSQL offering
* **Document movement due to growth**: when a document size grows to exceed its current slot on the disk, it must be moved creating additional server write load
* **Working set size**: for best performance, the majority of your active set should fit in RAM. Large documents may cause excessive server read load due to swapping documents in and out of the working set.

### Application access patterns

Successful NoSQL schema design requires that you understand data relationships and how the application will use the data. Specifically:

* what pieces of data are used together
* what data is mostly read-only
* what pieces of data are written frequently
* what is read vs write load - which should be optimized
* is strong consistency required; if one data change includes two collections, there is a possibility that a read may reflect a change to only one collection. This is usually rare - can it be tolerated

Once this is understood, we will craft a schema to suit the data and these application access patterns specifically.

### Schema design choices

There are three strategies for associating two pieces of data, A and B:

* document embedding
    * an A document could contain a B document
    * a B document could contain an A document
* separate collections using true linking
    * an A document could hold the id of a B document
    * a B document could hold the id of an A document
* separate collections using linking in both directions

Document embedding has some nice benefits:

* improved read performance: on spinning disks, reading from one collection (disk location) is faster than reading from two
* one round trip to the database: there are no joins in Mongo, so reading from two collections requires to database reads
* avoids update anomalies possible when writing two collections

However, if the document moves often because writes increase its size, the server suffers additional write load.

True linking requires application code to ensure the existence of one document referenced by another.

Bidirectional linking inherits the considerations of true linking, is more prone to modification anomalies and requires two database writes to establish a new relationship. Application code must ensure the existence of both documents and avoid update anomalies.

## One : One Relationships

From [Wikipedia](https://en.wikipedia.org/wiki/One-to-one_(data_model))

> a one-to-one relationship is a type of cardinality that refers to the relationship between two entities (see also entity–relationship model) A and B in which one element of A may only be linked to one element of B, and vice versa. For instance, think of A as countries, and B as capital cities. A country has only one capital city, and a capital city is the capital of only one country

This is a decent example, ignoring the 13 countries that have 2 capitals.

Possible schema implementations:

* document embedding
    * a country document could contain a city document
    * a city document could contain a country document
* separate collections using true linking
    * a country document could hold the id of a city document
    * a city document could hold the id of a country document
* separate collections using linking in both directions

Considerations:

* **access**: how frequently do you access country vs city and is one only fetched as drill down from the other.
    * If you frequently read the country, but rarely use the capital, you may want them in separate collections to increase the number of documents held in the working set to reduce read load on the server.
* **document size limit**: if the document can grow beyond the document size limit, you cannot embed, you must link.
* **working set size**: large documents mean fewer can be held in the working set and cause increased server read load. Choose linking if this is a concern.
* **document size growth**: if one document grows frequently while the other is static, you may choose to hold each in a separate collection to reduce server write load. Every document addition runs the chance of exceeding the size of the slot it is stored in on disk, requiring the server to move the document to a new, larger slot.
* **strongly consistent**: if you cannot tolerate any data inconsistency you should embed one document in the other to take advantage of atomic document operations.

Clearly, you must have a good understanding of how the application will access the data, the size of the data, and the fetch versus upsert frequency. The problem is that schemas are designed before you truly know these things and they may change as the application evolves.

**For 1 : 1 relationships, you will almost always choose document embedding unless you have a performance concern listed above.**

## One : Many Relationships

From [Wikipedia](https://en.wikipedia.org/wiki/One-to-many_(data_model))

> a one-to-many relationship is a type of cardinality that refers to the relationship between two entities (see also entity–relationship model) A and B in which an element of A may be linked to many elements of B, but a member of B is linked to only one element of A. For instance, think of A as mothers, and B as children. A mother can have several children, but a child can have only one mother.

Using a more extreme example, city -> person, clarifies problems with using embedded documents. If you embed people in a city, you would clearly blow through the document size limit. If you embed city in people, you have multiple copies of a city that must be kept in sync.

This same example exposes problems linking people to a city via a people id array: a city like Mexico City with 36 million inhabitants would blow through the document size limit.

**For 1 : many relationships, use true linking with the many having a link to the one: a person contains the city id.**

## One : Few Relationships

In real world domains, most 1 : many relationships are actually 1 : few. 

Using a blog post -> comments relationship as an example, you expect the blog post and all comments to be less than the maximum document size. You can embed comments as an array of documents in the associated blog post and get the benefits of strongly consistent data and atomic operations, but you must handle the edge case of exceeding maximum document size.

**For 1 : few relationships, embed the few in the one.**

## Many : Many Relationships

From [Wikipedia](https://en.wikipedia.org/wiki/Many-to-many_(data_model))

> a many-to-many relationship is a type of cardinality that refers to the relationship between two entities A and B in which A may contain a parent instance for which there are many children in B and vice versa.

Most many : many relationships in the real world tend to be few : few - use linking to relate them.

Using a books -> author relationship as an example, consider what happens if you embed author in the books collection. Many authors have written more than one book so you have multiple copies of the author in different books. If you embed books in the author document, you end up with a similar problem. Multiple copies of data should be avoided unless there is a compelling performance reason and the domain suggests there are few instance of multiple copies or the data is static.

So that leaves us with a linking solution:

* If your application navigates from book to author, provide author ids in the book document
* If the access pattern is from author to book, go the other direction
* It may be convenient to link in both directions but this opens up greater potential for data inconsistencies so ensure your application is careful here.

**For few : few relationships, use linking with the direction defined by application access pattern.**

## Multikey indexes

If you choose a multi-document linking solution, use multikey (array) indexes to make reading them fast.

Consider a 1 : few relationship, student -> teachers, where a student document has a teachers id array. Searching for teachers for a specific student is straight forward, and already indexed.

```json
{
    "_id": 1,
    "firstName": "Dirk",
    "lastName": "Gently",
    "teachers": [13, 8, 7]
}
```

Given the above student, find all his teachers with:

```js
db.teachers.find({_id: {$in: [13, 8, 7] }});
```

You can also easily find all students for a specific teacher:

```js
db.students.find({teachers: 13});
```

But what if you want to find all students that have the same teachers? This is where multikey indexes provide real performance gains. First, create a multikey index in the students collection on the teachers field:

```js
db.students.createIndex({teachers: 1});
```

Finding all students that have the same three teachers as Dirk looks like:

```js
db.students.find({teachers: { $all: [13, 8, 7] }});
```

## Trees

Data organized as a hierarchy is easy to traverse from top to bottom, but traditionally difficult to traverse from bottom to top. Because we can store data in arrays in NoSQL, we have a simple and elegant solution.

Consider a shopping site that has products and each product has a category. When the user selects a product, you want to show the hierarchy as breadcrumbs, from general to most specific for the product. For example:

```Category: home > outdoors > winter > snow```

A product looks like

```json
{ 
    "category": 7,
    "product_name": "snow blower"
}
```

and category 7 looks like

```json
{
    "_id": 7,
    "categoryName": "snow",
    "ancestors": [3, 13, 5]
}
```

Note that each category has an ordered list of ancestor ids, from the top of the category tree to the current node. A sub-category of 'snow' would have ancestors `[3, 13, 5, 7]`.

To find all sub-categories of 'snow' - a top-down search:

```js
db.categories.find({ancestors: 7});
```

To find the list of categories to create breadcrumbs - a bottom up search:

```js
db.categories.find({_id: { $in: [3, 13, 5] }});
```

Now that you have all the category documents, you need to order them for breadcrumb presentation.

## When to denormalize

Relational systems normalize data to avoid modification anomalies. From a relational point of view, NoSQL schemas frequently denormalize data. True, but this is OK if we are not duplicating the data - that is the root concern. 

Relational systems assume that any denormalized data could be duplicated because they are providing a neutral access pattern for potentially many applications. But when using application access pattern to drive schema design, you may know that the data will not be duplicated, avoiding the root concern. And if data duplication occurs infrequently, that can be OK as well as long as you code for it - ensuring duplicates stay in sync.

Avoiding duplicate data:

* **1 : 1**: embed. No duplication because you know it is always 1:1 and you will not have duplicates.
* **1 : many**: embed many in the one. If going from one to many, then linking avoids duplication. If you need to have duplicates for performance reasons, and the data is fairly static, this can be acceptable too.
* **many : many**: link. Performance may dictate that you embed here in which case your code must ensure that duplicates stay in sync.


