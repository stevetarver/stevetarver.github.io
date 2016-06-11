---
layout: post
title:  "A whole product concern logging implementation"
date:   2016-04-20 23:25:16
tags: logging java logback elk fluentd logstash kibana elasticsearch 12-factor postman

---

<img style="float: left;" src="/images/spring-boot-project-logo.png">

[Logging as a First Class Citizen](2016-01-09-logging-best-practices.md) describes what I think modern applications/services should be logging and why. Now, lets look at "how". I struggled with this for a while with Log4j, writing custom encoders to add kv pairs to log entries and aspects to extract API access event information - tedious and ugly. 

I switched from Splunk to ELK and was introduced to [Logstash](https://www.elastic.co/products/logstash). Most of what I read was about cleaning up log entries with Logstash filters and routing versatility with plugins. What I really wanted to know was what ELK preferred for efficient ingestion and Kibana queries. While searching for logging guidance and pouring over the documentation, I found that [Logback](http://logback.qos.ch/) and the [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder) solved all of my previous challenges while replacing custom code for configuration.

**TL;DR** An implementation of logging with whole product concern in Spring Boot and Logback.

## References

- [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder)
- [Logbask](http://logback.qos.ch/)
- [Implementation](https://github.com/stevetarver/rest-start-spring-boot)

The logstash-logback-encoder [README.md](https://github.com/logstash/logstash-logback-encoder) is a great resource for setup and evolution of your logging scheme. It took me a couple days of joining information from different sites and experimenting to get what I wanted. What I offer here is an opinionated solution and why each element of the strategy is important. You could probably just start with this setup and then evolve it to suit your situation.

While the implementation is Spring Boot specific, the ideas readily apply to other situations.

**NOTE** Code and log examples below are from this [implementation](https://github.com/stevetarver/rest-start-spring-boot).

## Why Logback

Logback is the most versatile, configurable, and easily extensible logging system I have seen. It allows great control over your log entry format, and specifically addresses our two logging concerns: issue remediation and API historical trending. Log Events provide the narrative messages that tell us what is going on in our code. Log AccessEvents provide rich detail about external callers and provide for historical trending.

The logstash-logback-encoder renders log entries that are most easily and efficiently ingested by the ELK stack. It produces a JSON log entry stream (as most logging systems will) and allows simple key-value additions to the log entry which is critical to this scheme.

I have used Log4J in the past and ended up having to write aspects to capture API call entries and exits for call count, duration, status. Using Log4J's JSON output would have required me to write things like logstash's StructuredArguments and Markers. With logback, I get that for free at the cost of a little configuration research.

## What should I be logging

Application logging is best suited to support the issues remediation and api call metrics and their trending.

Logging can provide narrative entries that tell you what happened and why; they allow you to trace call execution in diary form. These log entries describe every milestone in the call and each significant branching decision. As an example:

```
Contact change requested for bob.martin
Account bmc.helpdesk is authorized to make this change
Contact change request is valid
Contact change request is different than existing contact record
Contact record updated
Contact email has changed, sending email address verification to bob@bobmartin.com
```
Successful call requests are boring, but you get the idea. If the user submits a ticket about his contact details, you can see that they were properly updated and he should have a verification email in his inbox.

The API call metrics call count, duration, and response status can provide very basic, but essential context around how your application fairs in production. 

* Call count gives a basic indication of load. Call count peaks tell what load testing call volume should be.
* Contrasting call count with call duration shows when your application begins its downward spiral under load. At some call volume, your call duration will start to increase. If that duration increase is non-linear, you have a real problem. If not, at some point, increasing call duration provides an unacceptable UX, and finally cascade failures as timeout limits are reached.
* Contrasting call counts or duration with response status can also be a good indicator of pending application failures. There is likely some call volume where your response failure codes will increase and you need to ensure your peak production call volume is far below that.

Your base statistics are from production, but, as the wise man says: If you never test in development then everything in production is a test. These production stats become the baseline for your load testing and these metrics also provide a way to measure your load testing. All of this depends on your final product: Kibana dashboards built on top of these metrics.

**TIP**: When you need deeper insight into what pieces of code are causing your performance problems, Pivotal's [tc Server](https://www.vmware.com/products/vfabric-tcserver/overview) Insight plugin is in a class by itself. It is a drop-in replacement for Apache Tomcat and can be [built in to Spring Boot](http://stackoverflow.com/questions/27211048/using-spring-boot-web-application-with-pivotal-tc-server). The Insight plugin is exposed through a webapp whose main dashboard highlights performance problems. You can focus on a single, poorly performing API call and drill down into all internal calls made to accomplish the API task. You can quickly identify the problem source and frequently have enough context to understand the root problem. The license agreement is free for development use.

## Logging should not be a burden for developers

This logging implementation is a sort of middleware for the final product: Kibana searches and queries. Optimizing for efficient log entry ingestion and queries requires a machine optimized format. But that final product should not get in the way of development or make developer's lives tedious by forcing them to struggle through reading that machine optimized format.

There are three basic use cases:

- unit tests
- local development
- deployed code

Unit tests will use a `/src/test/resources/logback-test.xml` if provided. We can configure a simple STDOUT appender optimized for human reading.

During local development, we can use the same human readable unit test STDOUT appender configuration in `/src/main/resources/logback.xml`. Developers can manually enable/disable it, or depending on your deployment configuration, have it automatically enabled for the developer's environment. You can also provide a file appender that uses the same encoding scheme that deployed code uses so developers can verify that what they want sent to ELK is actually written.

Deployed code will use a separate appender that encodes log entries tailored to log ingestion and searching.

## What changes when I log for ELK

Frequently, developers force the logging system to adapt to their old habits - ingesting plain text as files, inefficiently parsing it, and not providing parsable fields. This simply doesn't scale: 

- log management systems need to be bigger and more costly
- log management ingestion can't keep up with log volume so operators have to wait for entries to appear
- dashboards can become excruciatingly slow to load and can lock out users
- log entries are more frequently dropped

When we think of logging as a product with ELK as a consumer, and operations and product management as the end user, this all changes. We can optimize log format and delivery and significantly reduce each of these problems.

Our delivery system will use something like Logstash or Fluentd to route from our application/service to the logging pipeline. That pipeline begins with intermediate storage like Redis or Kafka so log messages are not dropped should the backend be overloaded or down. Then log entries are moved to the log management system's durable storage.

Our logging format changes from plain text to a regular set of key-value pairs: JSON. In addition, everything that we want to search on, especially large metrics type searches, must be in a key-value pair. The basic logback configuration, as well as all other logging frameworks, force you to put all of your parameters in the log message field. This means that the log management system must regex the message field and typical inconsistency in log messages makes this practically impossible. Logback's Structured Arguments allow you to append key-value pairs to the log entry, effectively changing a message field regex to a select by column. It also helps with consistency, especially if you use a common dictionary of key names containing one definition of `userId`.

Logback's StructuredArguments provide one more tremendous benefit: recording complete method arguments. When a parameter is encoded as a StructuredArgument, the object is encoded as json, including all arrays and parent classes. When you have a production issue, you can find the log entry and paste the entire request body into something like Postman to recreate the call. Without this feature, recreating the problem is a significant time hole.

When we can separate request variables and narrative entries into their own fields, we can really clean up the narrative entries. A couple of Kibana clicks will show only those narrative entries providing a diary view shown above.

Enough concept and motivation, let's get to the implementation.

## LoggingEvents and AccessEvents

ref: [HTTP-access logs with logback-access, Jetty and Tomcat](http://logback.qos.ch/access.html)

LoggingEvents are used with traditional application logging calls like `log.info()`. They directly address our need for issue remediation via narrative log entries and API call parameters. The types of field `Providers` are listed [here](https://github.com/logstash/logstash-logback-encoder#providers-for-loggingevents).

AccessEvents are focused on the area between HTTP request receipt and application code delivery. They provide fields like HTTP method, URI, status code, and elapsed time and directly address our need for historical trending. The complete list of AccessEvent field `Providers` are listed [here](https://github.com/logstash/logstash-logback-encoder#providers-for-accessevents).


## Adding Logback and logstash-logback-encoder to your project

ref: [logstash-logback-encoder README.md](https://github.com/logstash/logstash-logback-encoder#including-it-in-your-project)

### Spring Boot

Logback 1.1.6 adds support for easily reading `logback-access.xml` from your jar. Below we include logback classic, core, and access, overriding the Logback dependencies included by `spring-boot-starter-web` and `logstash-logback-encoder`. Note that logback also requires slf4j which is already included in Spring Boot.

```xml
    <!-- Logging -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>4.6</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.1.7</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <version>1.1.7</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-access</artifactId>
        <version>1.1.7</version>
    </dependency>
```

Since Tomcat is embedded, we need to programmatically inject a Valve to send access events to Logback. Our config file `src/main/resources/logback-access.xml` will be automatically loaded from the jar through the class loader.

```java
@Configuration
public class LogbackAccessEventConfiguration {

    @Bean
    public EmbeddedServletContainerCustomizer containerCustomizer() {

        return new EmbeddedServletContainerCustomizer() {

            @Override
            public void customize(ConfigurableEmbeddedServletContainer container) {
                if (container instanceof TomcatEmbeddedServletContainerFactory) {
                    ((TomcatEmbeddedServletContainerFactory) container)
                            .addContextCustomizers(new TomcatContextCustomizer() {

                                @Override
                                public void customize(Context context) {
                                    LogbackValve logbackValve = new LogbackValve();
                                    logbackValve.setFilename("logback-access.xml");
                                    context.getPipeline().addValve(logbackValve);
                                }
                            });
                }
            }
        };
    }
}
```


### Tomcat

If you are using a dedicated Tomcat/jetty to host your app

- logstash-logback-encoder and its dependencies go in your war
  - jackson-databind / jackson-core / jackson-annotations
  - logback-core
  - logback-classic (required for logging LoggingEvents)
  - slf4j-api
- logback-access and its dependencies go in your container.
  - jackson-databind / jackson-core / jackson-annotations
  - slf4j-api

**NOTE**:

- Ensure that all Logback dependencies match - logback really hates mismatches.
- Logback v1.1.7 has a *SocketAppender bug where encoder providers are not recognized. This is reported to be fixed in the upcoming v1.1.8, but for now we'll use v1.1.6.

Your logback-access.xml goes in `$TOMCAT_HOME/conf/` (where server.xml is).

## Configuring LoggingEvents in Logback

ref: [Providers for LoggingEvents](https://github.com/logstash/logstash-logback-encoder#providers-for-loggingevents)

Our LoggingEvent configuration uses the `LoggingEventCompositeJsonEncoder` to allow StructuredArguments and Markers to insert k-v pairs into the log entry; de-clutters our narrative message.

We will rename the log entry fields to have a `log` prefix to avoid key name collisions in application code injected fields.

This encoder configuration defines what the log framework will log for you. Note the addition of the static `logContext` field that holds your application or service name. This really helps disambiguate log entries in searches. You can focus on code from only one service and avoid unintentional overlap in query terms.

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- https://github.com/logstash/logstash-logback-encoder#composite_encoder -->
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <version/>
                <!-- https://github.com/logstash/logstash-logback-encoder#provider_pattern -->
                <pattern>
                    <pattern>
                        {
                        "logMessage": "%message",
                        "logLevel":    "%level",
                        "logThreadId": "%thread",
                        "logClass":    "%class{32}",
                        "logMethod":   "%method",
                        "logContext":  "myServiceName"
                        }
                    </pattern>
                </pattern>
                <!-- log guid support -->
                <mdc/>
                <!-- StructuredArgument and Marker support -->
                <arguments/>
                <logstashMarkers/>
                <stackTrace>
                    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                        <maxDepthPerThrowable>10</maxDepthPerThrowable>
                        <maxLength>2048</maxLength>
                        <shortenedClassNameLength>32</shortenedClassNameLength>
                        <exclude>sun\.reflect\..*\.invoke.*</exclude>
                        <exclude>net\.sf\.cglib\.proxy\.MethodProxy\.invoke</exclude>
                        <rootCauseFirst>true</rootCauseFirst>
                    </throwableConverter>
                </stackTrace>
            </providers>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

## Configuring AccessEvents in Logback

ref: [Providers for AccessEvents](https://github.com/logstash/logstash-logback-encoder#providers-for-accessevents)

Logback integrates into Tomcat/Jetty to read Servlet Container access events. This is a completely separate logging system than LoggingEvents but we will combine both in STDOUT.

Although it is possible, we won't include request bodies in the access log. I think this information is better done in the application code so that all information is available in the API request log entry to reproduce the request. The LoggingEvents and AccessEvents can be tedious to tie together in a busy log management system.

Above, we renamed fields with a 'log' prefix. Since we are not injecting our own key names into the access event, we don't need to do that here. I did rename the fields from their standard forms, because they are not as intuitive as those listed.

```
<configuration>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- https://github.com/logstash/logstash-logback-encoder#accessevent-patterns -->
        <!-- https://github.com/logstash/logstash-logback-encoder#providers-for-accessevents -->
        <!-- http://logback.qos.ch/xref/ch/qos/logback/access/PatternLayout.html -->
        <encoder class="net.logstash.logback.encoder.AccessEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <version/>
                <method>
                    <fieldName>method</fieldName>
                </method>
                <requestedUri>
                    <fieldName>uri</fieldName>
                </requestedUri>
                <statusCode>
                    <fieldName>statusCode</fieldName>
                </statusCode>
                <elapsedTime>
                    <fieldName>elapsedTime</fieldName>
                </elapsedTime>
                <contentLength>
                    <fieldName>contentLength</fieldName>
                </contentLength>
                <remoteHost>
                    <fieldName>remoteHost</fieldName>
                </remoteHost>
                <pattern>
                    <pattern>
                        {
                        "queryString": "%q"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <appender-ref ref="STDOUT" />
</configuration>
```

## StructuredArguments and Markers

ref: 

- [Event-specific Custom Fields](https://github.com/logstash/logstash-logback-encoder#event-specific-custom-fields)
- [Additional StructuredArguments methods](https://github.com/logstash/logstash-logback-encoder/blob/master/src/main/java/net/logstash/logback/argument/StructuredArguments.java)

LoggingEvents allow the possibility of injecting additional fields into the log entry. This is a very good thing because it de-clutters the narrative message, provides additional fields for robust Kibana queries, and allow you to provide enough information to reproduce the API call when debugging.

StructuredArguments are the preferred method, but you may use LogstashMarkers to suppress static analyzers that complain about unused arguments.

Here are some common uses:

```java
// Use the implicit String.format
log.info("log message {}", localVarName);

// Add "name":"value" to the JSON output,
// but only add the value to the formatted message.
log.info("log message {}", value("name", "value"));

// Add "name":"value" to the JSON output,
// and add name=value to the formatted message.
log.info("log message {}", keyValue("name", "value"));

// Add "name":"value" ONLY to the JSON output.
log.info("log message", keyValue("name", "value"));

// Add multiple key value pairs to both JSON and formatted message
log.info("log message {} {}", keyValue("name1", "value1"), keyValue("name2", "value2")));
```

StructuredArguments have shorthand method names like `v()` for `value()` and `kv()` for `keyValue()`


## Loggging in code

Let's start tying this all together with some ReST API code examples. Our goals are to provide a narrative log message for diary view AND enough information to reproduce the API call for debugging.

Logback uses the slf4j facade and a static import to augment those methods. I am using the most excellent [lombok](https://projectlombok.org/) [@Slf4j](https://projectlombok.org/api/lombok/extern/slf4j/Slf4j.html) annotation to repace the boilerplate

```java
     private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(LogExample.class);
```
with `@Slf4j`

```java
import static net.logstash.logback.argument.StructuredArguments.*
// ...

@Slf4j
@RestController
@RequestMapping(value = "/contacts", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public class ContactsController {
    // ...
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public ResponseEntity get(
            @PathVariable long id,
            @RequestHeader HttpHeaders headers) {
        log.info("Get Contact {} requested", v("id", id), kv("headers", headers));
        return ResponseEntity.ok(contactsService.get(id));
    }
```
produces the LoggingEvent

```json
{"@timestamp":"2016-03-13T22:36:44.067-06:00","@version":1,
"logMessage":"Get Contact 5 requested","logLevel":"INFO","logThreadId":"http-nio-8080exec-1",
"logClass":"c.g.s.c.ContactsController","logMethod":"get","id":5,
"headers":{"host":["localhost:8080"],"connection":["keep-alive"],"cache-control":["max-age=0"],
"accept":["text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8"],
"upgrade-insecure-requests":["1"],"user-agent":["Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/48.0.2564.116 Safari/537.36"],
"accept-encoding":["gzip, deflate, sdch"],"accept-language":["en-US,en;q=0.8"],
"cookie":["_ga=GA1.1.1621264663.1446705863"]}}
```

There are a lot of headers here, which may be useful to a UI component. API components may want to be more restrictive and pluck only the interesting headers. 

Both will want to avoid authentication headers and other sensitive information which is more elegantly done in a Logstash grok filter.

This AccessEvent is also produced:

```
{"@timestamp":"2016-03-13T22:36:44.440-06:00","@version":1,"method":"GET",
"uri":"/contacts/5","statusCode":200,"elapsedTime":30,"contentLength":-1,
"remoteHost":"0:0:0:0:0:0:0:1","queryString":""}
```

The create method 

```java
    @RequestMapping(method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public ResponseEntity<Contact> create(
            @RequestBody @Validated(Contact.Post.class) Contact contact) {
        log.info("Create Contact {} requested", contact.getEmail(), kv("contact", contact));
        return ResponseEntity.ok(contactsService.create(contact));
    }
```
produces the LoggingEvent

```json
{"@timestamp":"2016-03-13T22:57:39.111-06:00","@version":1,
"logMessage":"Create Contact bob@thebuilder.com requested","logLevel":"INFO",
"logThreadId":"http-nio-8080-exec-1","logClass":"c.g.s.c.ContactsController",
"logMethod":"create","contact":{"id":null,"firstName":"Bob","lastName":"The Builder",
"companyName":"Bob's Construction","address":"1234 Beaver lane","city":"Austin",
"county":"Pitkin","state":"TX","zip":"81615","phone1":"3136364482",
"phone2":"3136369982","email":"bob@thebuilder.com","website":"http://bobthebuilder.com"}}
```
We can see we can easily copy the contact JSON into Postman to reproduce this call.

The AccessEvent event looks like

```json
{"@timestamp":"2016-03-13T22:57:39.492-06:00","@version":1,"method":"POST",
"uri":"/contacts","statusCode":200,"elapsedTime":487,"contentLength":-1,
"remoteHost":"0:0:0:0:0:0:0:1","queryString":""}
```

## Tying your services together

ref: [Logback MDC](http://logback.qos.ch/manual/mdc.html)

Let's say you get a customer ticket describing what the customer did and how it failed. How do you tie your UI and possibly several API components together to get a complete picture of the problem?

While something like [zipkin](http://zipkin.io/) is awesome, it is separate from the logging system - we want to see all narrative entries from components from the UI, through all API components, to datastore access.

In a thread per service call language like java, there is a pretty simple solution using logback's MDC component. We can use a tracing token (logguid) that follows the API chain wherever it goes. Each component follows the same strategy:


1. for every inbound endpoint
  1. if an inbound logguid header exists, insert it into MDC
  1. else generate one and insert that into MDC
1. for every outbound call
  1. add a logguid header using the value in MDC

You can create a global request interceptor for item one and a base class to handle outbound requests for item two.

How is this life changing? You get a customer ticket stating that their contact update didn't work. Trying to get more information from the customer is usually a waste of time. You are left to searching through log errors and trying to tie that back to a log entry containing the user's id. Really frustrating and always a waste of time.

With the addition of the logguid, you can search for the user's id and then search by logguid to see which interaction failed. 

Of course, if this is a frequent scenario, you could add userId to MDC and follow the same procedure as with the logguid.


## What next?

You now have the building blocks for making yours, and your operator's lives easier. When you connect your log stream to ELK, or any other log management system, you can begin to create great dashboards that help you understand production performance, quantify load test results, and simplify issue remediation.

If logging for a whole product concern is new, outstanding! If not, I hope I could provide a couple of new ideas, and if not, outstanding as well, cause you are ahead of the herd.

