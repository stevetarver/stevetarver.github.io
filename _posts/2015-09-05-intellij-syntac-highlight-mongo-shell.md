---
layout: post
title:  "IntelliJ: Syntax highlight mongo shell documents"
subtitle: "Create custom highlighting for mongo scripts in IntelliJ"
date:   2015-09-05
categories: intellij mongo mongodb syntax highlighting
---

![IntelliJ](/images/intellij.png) 

Much DBA work is done in scripts for testing in dev and promotion up the quality stack. How can we make that easier in Mongo?

The Mongo shell is great and there are hacks to make it even better like [mongo-hacker](https://github.com/TylerBrock/mongo-hacker), but what about script writing? You can use two file types: Javacript or text. Both have advantages, but only one has standard syntax highlighting to make script writing easier. In IntelliJ, we can fix that.


## File types: pros and cons

### Javascript:

In javascript, you can use all the Mongo API facilities including the ability to connect to remote servers and authenticate against them. You have standard syntax highlighting in many IDEs, but it won't cover Mongo specific commands and operators. You won't have access to all mongo shell functionality, but there are equivalent operations for almost everything anyone would want to do.

{% highlight bash %}
mongo --eval javascript.js
mongo host:port javascript.js
{% endhighlight %}

#### Pros

- Full Javascript API access
- Block and inline comments
- Reasonable syntax highlighting guesses at dotted objects, object keys and values

#### Cons

- No Mongo specific syntax highlighting
- No Mongo specific context appropriate command and operator suggestions
- Javascript rules for double quoting operators and keywords - slightly different than the shell
- Keyword completion will show many things that are not appropriate - will break the script

### Text files

With text files, the commands you enter are exactly as you would enter them in the shell. You can develop your script in the mongo shell and then copy it to a script with no javascript conversion. Setting up a custom syntax highlighting scheme allows you to highlight all Mongo commands and operators AND context sensitive keyword suggestions. Keyword completion allows you to type two letters of a command or operator and IntelliJ shows a dropdown list of all keywords that have those two letters in them. You won't see javascript specific operator highlighting, like conditionals and loops, but when writing complex aggregation pipelines, I find the operator syntax highlighting more useful.

You can pipe or redirect a text file to the mongo shell like this:

{% highlight bash %}
cat script.txt | mongo
mongo < script.txt
{% endhighlight %}

#### Pros

- Full shell command access
- Exactly the same syntax as working in the shell - same double quoting, etc.
- Syntax highlighting for keywords and commands
- Keyword completion that includes only Mongo commands and operators

#### Cons

- Missing some Javascripty syntax highlighting and keyword completion

You can set up this syntax highlighting scheme and see which you prefer. Perhaps some scripts will work better as javascript, and some better as text.

## Create a new .mongo file association

1. Open Preferences: ⌘, (Menu: IntelliJ IDEA -> Preferences...)
2. In the Preferences dialog left sidebar, show the Editor -> File Types page
1. Under Recognized File Types, click the ➕ (Add) button

Fill in the dialog as below, except for keywords which we'll do next

![IntelliJ Edit File type](/images/intellij-mongo-edit-file-type.png)

There are four tabs for keywords, differentiated by highlight color.

Standard entry is one keyword at a time. With over a hundred operators alone, this is far too tedious to do by hand. There is a better way, described next. For now, click OK on the Edit File Type dialog.

In the Preferences dialog, under Registered Patterns, add '*.mongo'.

Click Apply and OK on the Preferences dialog.


## Setting keywords

I waded through the MongoDB 3.0 manual and collected all the commands and operators, sorted and de-duplicated them, and added them to the .mongo filetypes xml files.

Open ~/Library/Preferences/IntelliJIdea14/filetypes/Mongo Shell.xml and replace the contents with the xml at the bottom of this post.

Restart IntelliJ and you should have syntax highlighting for *.mongo files.


## Adjusting syntax highlighting

For all manually configured file types, there is one highlighting scheme; a separate color for keywords on tabs 1, 2, 3, 4. You can view and edit these color by opening Preferences and selecting Editor -> Colors and Fonts -> Custom. the color associations are for Keyword1, Keyword2, Keyword3, Keyword4.


### ~/Library/Preferences/IntelliJIdea14/filetypes/Mongo Shell.xml

{% highlight xml %}
<filetype binary="false" description="Mongo Shell" name="Mongo Shell">
  <highlighting>
    <options>
      <option name="LINE_COMMENT" value="//" />
      <option name="COMMENT_START" value="/*" />
      <option name="COMMENT_END" value="*/" />
      <option name="HEX_PREFIX" value="" />
      <option name="NUM_POSTFIXES" value="" />
      <option name="HAS_BRACES" value="true" />
      <option name="HAS_BRACKETS" value="true" />
      <option name="HAS_PARENS" value="true" />
      <option name="HAS_STRING_ESCAPES" value="true" />
      <option name="LINE_COMMENT_AT_START" value="true" />
    </options>
    <keywords keywords="_hashBSONElement;_journalLatencyTest;_migrateClone;_recvChunkAbort;_recvChunkCommit;_recvChunkStart;_recvChunkStatus;_replSetFresh;_transferMods;addShard;aggregate;applyOps;authSchemaUpgrade;authenticate;availableQueryOptions;buildInfo;captrunc;checkShardingIndex;clean;cleanupOrphaned;clone;cloneCollection;cloneCollectionAsCapped;collMod;collStats;compact;configureFailPoint;connPoolStats;connPoolSync;connectionStatus;convertToCapped;copydb;copydbgetnonce;count;create;createIndexes;createRole;createUser;cursorInfo;dataSize;dbHash;dbStats;delete;diagLogging;distinct;driverOIDTest;drop;dropAllRolesFromDatabase;dropAllUsersFromDatabase;dropDatabase;dropIndexes;dropRole;dropUser;emptycapped;enableSharding;eval;explain;features;filemd5;findAndModify;flushRouterConfig;forceerror;fsync;geoNear;geoSearch;geoWalk;getCmdLineOpts;getLastError;getLog;getParameter;getPrevError;getShardMap;getShardVersion;getnonce;godinsert;grantPrivilegesToRole;grantRolesToRole;grantRolestoUser;group;handshke;hostInfo;insert;invalidateUserCache;isMaster;isSelf;isdbgrid;listCollections;listCommands;listDatabases;listIndexes;listShards;logApplicationMessage;logRotate;logout;mapReduce;medianKey;mergeChunks;moveChunk;movePrimary;netstat;parallelCollectionScan;ping;planCacheClear;planCacheClearFilters;planCacheListFilters;planCacheListPlas;planCacheListQueryShapes;planCacheSetFilter;profile;reIndex;removeShard;renameCollection;repairCursor;repairDatabase;replSetElect;replSetFreeze;replSetGetConfig;replSetGetRBID;replSetGetStatus;replSetHeartbeat;replSetInitiate;replSetMaintenance;replSetReconfig;replSetStepDown;replSetSyncFrom;replSetTest;resetError;resync;revokePrivilegesFromRole;revokeRolesFromRole;revokeRolesFromUser;rolesInfo;serverStatus;setParameter;setShardVersion;shardCollection;shardConnPoolStats;shardedfinish;shardingState;shutdown;skewClockCommand;sleep;split;splitChunk;splitVector;testDistLockWithSkew;testDistLockWithSyncCluster;top;touch;unsetSharding;update;updateRole;updateUser;use;usersInfo;validate;whatsmyuri;writeBacksQueued;writebacklisten" ignore_case="false" />
    <keywords2 keywords="$;$add;$addToSet;$all;$allElementsTrue;$and;$anyElementTrue;$avg;$bit;$cmp$eq;$comment;$concat;$cond;$currentDate;$dateToString;$dayOfMonth;$dayOfWeek;$dayOfYear;$divide;$each;$elemMatch;$eq;$exists;$explain;$first;$geoIntersects;$geoNear;$geoWithin;$group;$gt;$gte;$hint;$hour;$ifNull;$in;$inc;$isolated;$last;$let;$limit;$literal;$lt;$lte;$map;$match;$max;$maxScan;$maxTimeMS;$meta;$millisecond;$min;$minute;$mod;$month;$mul;$multiply;$natural;$ne;$near;$nearSphere;$nin;$nor;$not;$or;$orderby;$out;$pop;$position;$project;$pull;$pullAll;$push;$pushAll;$query;$redact;$regex;$rename;$returnKey;$second;$set;$setDifference;$setEquals;$setIntersects;$setIsSubset;$setOnInsert;$setUnion;$showDiskLoc;$size;$skip;$slice;$slize;$snapshot;$sort;$strcasecmp;$substr;$subtract;$sum;$text;$toLower;$toUpper;$type;$unset;$unwind;$week;$where;$year" />
    <keywords3 keywords="use" />
    <keywords4 keywords="_id" />
  </highlighting>
  <extensionMap>
    <mapping ext="mongo" />
  </extensionMap>
</filetype>
{% endhighlight %}