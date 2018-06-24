---
layout: post
title:  "ReST: Spring Boot starter (part 1)"
date:   2016-05-12 13:25:16
tags: rest spring-boot java lombok spock jdbc 12-factor

---

**TL;DR** Spring Boot ReST service starter

<img style="float: left;" src="/images/logo/spring-boot-project-logo.png">

I was stuck in Java 6 land for a REALLY long time. And then I carved out some time to learn CoffeeScript (thank you Jeremy). And then I carved out some time to learn Ruby (thank you Matz). And I remembered a love that had stagnated working in the Enterprise with Java: languages. I love figuring out the most elegant solutions to problems and REALLY love when I have no restrictions; not on language, not on framework.

So I thought "What if I built a canonical service in JavaScript (CoffeeScript, Elm), Go, Elixir, Ruby, and Java - just to compare them?"

So that's where this started, and ironically, because of some things going on at work, the first project is in Java with Spring Boot.

## On Spring Boot

I have a little experience with Java, and Spring. Both have frustrated me in the past. 

I have built Java EE things for application servers and then added the simplification and obfuscation of Spring to my toolset. After solving problems in other languages, I am less a fan of the highly structured approach; the micro-services I try to build are just not that complicated. I value simplicity, rapid evolution, uptime, and the ability to learn about the service from the production environment; the interesting parts for me these days are DevOps concerns.

Java 6 seems so mechanical and after the elegance of CoffeeScript and Ruby, just painful to look at. I am lucky to live in a Java 8 world now, and what a bright world that is. Still verbose, teetering on the edge of pedantic, but with closures, streams, and recent improvements, it adds functionality common in modern languages, just not as pretty to look at. Add in Spring and lombok to eliminate boiler plate and you have a pretty attractive environment.

Spring has 4 different types of configuration, and just way too many ways to do everything. Jumping into a new project means you have to figure out the combinations of ways the developers decided to piece Spring together. Spring Boot provides a more opinionated view into the Spring universe and AMAZING documentation that guided me through my first ReST service in a day.

If the Spring doc is so great, why are you writing yet another starter tutorial? Well, this isn't a tutorial so much as it is a description of how I think a cloud service should look. You can find the project on my [GitHub](https://github.com/stevetarver/rest-start-spring-boot), but I will really be focusing on technology decisions and working through solving implementation challenges to produce a decent cloud app (which, of course means 12 factor app).


## What's important in a ReST framework

Covered in [Choose a ReST framework](2015-07-17-choose-a-rest-framework.md) at length, let's summarize key areas (and some I left out earlier). This list represents the order I will be addressing each while building the project.

Part 1

- Routes: elegant and flexible code selection from URI
- URI path/request param extraction: marshal URI path and request params into code variables
- Data store connectivity: simplicity in managing datasources (setup, pooling, efficiency)
- Marshalling data from the backing store and to the client
- Rich error responses: can I provide enough error info to help the client help themelves
- Parameter validation: generate 400's when the client doesn't adhere to the API spec
- Logging: a suitable logging framework for log aggregator ingestion

Part 2

- Metrics: are metrics exposed for systems like Prometheus and are they extensible?
- Health: do health metrics exist and are they extensible
- Security: are standard security protocols provided by the framework
- Testing: is unit and integration testing elegant
- Externalized configuration: Consul integration anyone?
- API doc: is there a suitable API doc generation system like Swagger

## What you need to get started

- Java 8
- Maven
- An IDE (lotsa love for [IntelliJ](https://www.jetbrains.com/idea/) here)

## Getting started

Maven suffers from some of the same problems as Yoeman: everybody and their brother has made an archetype, you never know what you are gonna get, it is rarely what you want, and few are maintained.

Because Spring Boot is opinionated and lives in their own ecosystem, they have the power to fix this, and they have. The Spring Boot [starter site](https://start.spring.io/) allows you to pick the features you want and generate a zip that is your initial, runnable project.

After playing with the advanced feature selection, I always start with the basic starter. A `@SpringBootApplication` enables auto-configuration - if certain jars are on your class path, Spring will assume that you are using them, try to configure them, and fail to load the Spring context leading to difficult to diagnose problems. In our case, the presence of data jars like jpa or the MySQL driver will cause Spring to try to configure the datasource, but since we haven't coded that yet, it will fail tests and fail to build.

I just include Actuator and incrementally add required jars as I build up the program.

So, press the start.spring.io Generate Project button, unzip the downloaded file,

```shell
mvn install
mvn spring-boot:run
```

And your ReST service is up and running on `http://localhost:8080/`. All the [Actuator endpoints](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html) are available.

## Routes

How do we map a URI to the code that will handle the request?

Spring solves routing simply and elegantly: class and method annotations. A class uses a `@RequestMapping` annotation to tell Spring that it wants all matching URI requests. Class methods will include a `@RequestMapping` to further disambiguate the request and match it to a specific handler.

For ReST services, a URI represents a hierarchy of Resources; canonically of the form `/collection/selector/collection/selector/...`. The request handler is responsible for validating parameters, calling a service to accomplish the task, and providing a proper ReST response to the client. With Jersey, these request handlers are known as `Resources` and live in a `/resource` directory. With Spring MVC, these request handlers are known as `Controllers` and live in a `/controller` directory.

Since we are kinda working in a Spring MVC world, we'll create a Controller for Contacts. The request mapping looks like:

```java
@RestController
@RequestMapping(value = "/contacts", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public class ContactsController {
    
    @RequestMapping(method = RequestMethod.GET)
    public ResponseEntity<List<Contact>> getList() {
        
        List<Contact> response = new LinkedList<Contact>();
        return ResponseEntity.ok(response);
    }
}
```

If a user GETs `http://localhost:8080/contacts`, the `getList()` method is executed.

Pretty clear. Pretty simple. Routing goes with the code that implements the handler. 

Other routing systems put all routes and references to the handlers in a single file which has some advantages. The single routing file is a map to all API facing code and URI inconsistencies are readily apparent. Spring's approach, especially on larger projects, can lead to URI collisions and inconsistencies in URIs which produce an ugly and confusing API for the client.

**NOTE:** The `@RestController` annotation lets Spring know that this class should be scanned for request mappings. Without it, none of the `@RequestMapping`s would be considered for URI matching.


## URI path/request variable extraction

If a client GETs `http://localhost:8080/contacts/5`, Spring matches on this class and method

```java
@RestController
@RequestMapping(value = "/contacts", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public class Contacts {

    @RequestMapping(value = "/{id}", method= RequestMethod.GET)
    public ResponseEntity<Contact> get(
            @PathVariable int id)
    {
        return ResponseEntity.ok(new Contact());
    }
```

The `{id}` request mapping indicates id is a variable and extracts it into `get(int id)`'s id parameter. If there are multiple path variables, they are extracted in order into the method arguments.

**NOTE**: If the caller sent a GET on `http://localhost:8080/contacts/steve`, Spring would not be able to convert `steve` to an int, decide that there was no request handler for the URI, and return a 400 BAD REQUEST.


Request params (queryStrings) are handled in a similar way.

```
    @RequestMapping(method = RequestMethod.POST)
    public ResponseEntity<Contact> create(
            @RequestParam(name = "firstName") String firstName,
            @RequestParam(name = "lastName") String lastName,
            @RequestParam(name = "companyName") String companyName,
            @RequestParam(name = "address") String address,
            @RequestParam(name = "city") String city,
            @RequestParam(name = "county") String county,
            @RequestParam(name = "state") String state,
            @RequestParam(name = "phone1") String phone1,
            @RequestParam(name = "phone2") String phone2,
            @RequestParam(name = "email") String email,
            @RequestParam(name = "website") String website)
    {
        return ResponseEntity.ok(new Contact());
    }
```
Here, Spring uses request parameter key name matching to allow any order in the request params.

**NOTE**: As with path params, if the request param cannot be marshaled into the target method param type, Spring will return a 400.

This is a pretty ugly method, no worries, we'll clean that up later when we look deeper at serialization.

## Test what we've done

To prove the API facing code, let's modify our `get(int id)` method and test it in a browser.

```
    @RequestMapping(value = "/{id}", method= RequestMethod.GET)
    public ResponseEntity<Contact> get(
            @PathVariable int id)
    {
        Contact result = new Contact();
        if(5 == id) {
            result.setFirstName("Bob");
        }
        return ResponseEntity.ok(result);
    }
```

Run the service with `mvn spring-boot:run`, verify a clean start in the terminal, and then `http://localhost:8080/contacts/5`

```json
{
  "id": 0,
  "firstName": "Bob",
  "lastName": null,
  "companyName": null,
  "address": null,
  "city": null,
  "county": null,
  "state": null,
  "phone1": null,
  "phone2": null,
  "email": null,
  "website": null
}
```

If you use any other contact id, you will see nulls and zeros. If you use `foo`, you will get a 400.

## Lombok

We're going to use Lombok to clean up our domain models so let's take some time to set that up now.

Lombok's awesomeness lies in its ability to eliminate boiler plate, but tedious to integrate into your IDE - no one site has all the steps. These instructions are for IntelliJ - apologies if you have to use another IDE ;)

In `pom.xml`, dd the Lombok dependency

```
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.8</version>
            <scope>provided</scope>
        </dependency>
```

Configure the compiler

```
	<build>
		<plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.5.1</version>
                <inherited>true</inherited>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                    <encoding>UTF-8</encoding>
                    <annotationProcessors>
                        <annotationProcessor>lombok.launch.AnnotationProcessorHider$AnnotationProcessor</annotationProcessor>
                    </annotationProcessors>
                </configuration>
            </plugin>
            ...
		</plugins>
	</build>
```

In IntelliJ:

1. Install the plugin
1. In your project Preferences
  - Build, Execution, Deployment -> Compiler -> Annotation Processors. Check Enable Annotation Processing
  - Add to Annotation Processors `lombok.launch.AnnotationProcessorHider$AnnotationProcessor`
  - Other Settings -> Lombok Plugin -> Enable Lombok for this project...
1. Add lombok.jar as a global library:
  - File -> Project Structure -> Global Libraries -> ‘+’
  - Navigate to your lombok jar. Note, you will have to type in `.m2` to see the maven repo. My path looks like `/Users/starver/.m2/repository/org/projectlombok/lombok/1.16.8/lombok-1.16.8.jar`
  - Enable the jar in module dependencies: File -> Project Structure -> Modules Dependencies tab, check the lombok.jar - it should already be in the list because of maven dependencies.
1. Restart IntelliJ


## Seeding contact data

My [sample data](https://github.com/stevetarver/sample-data) repo has sql import scripts. You can clone this repo and use the script to provide some reasonable data to experiment with in this project.

## JPA or simpler

Spring builds on Hibernate to provide ORMs that can eliminate boilerplate database access code. I have used this on some projects, but never a case where the simplications warranted the complexity and performance hit. Unless you have done the initial configuration a couple of times, it is always a frustrating experience. I prefer JdbcTemplate.

## Datastore connectivity

To prove database connectivity, let's implement `GET /contacts`. We need a JdbcTemplate to fetch the data from the database

First, let's create our domain model - based on my sample data mentioned above. Through the beauty of Lombok, this looks like:

```
@Data
public class Contact {
    private long id;
    private String firstName;
    private String lastName;
    private String companyName;
    private String address;
    private String city;
    private String county;
    private String state;
    private String phone1;
    private String phone2;
    private String email;
    private String website;
}
```

This is simply beautiful! Lombok transparently adds @ToString, @EqualsAndHashCode, @Getter on all fields, @Setter on all non-final fields, and @RequiredArgsConstructor for us. Lombok goes with me everywhere I do java.

Time to add the jdbc and mysql dependencies to the `pom.xml`

```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.38</version>
		</dependency>
```

We would never ship this way, but for now, add something like the following to your `application.properties`

```
spring.datasource.url=jdbc:mysql://localhost/test?autoReconnect=true
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

At this point, we can run the project, browse to `http://localhost:8080/health` and see

```json
{
  "status": "UP",
  "diskSpace": {
    "status": "UP",
    "total": 999695822848,
    "free": 405963988992,
    "threshold": 10485760
  },
  "db": {
    "status": "UP",
    "database": "MySQL",
    "hello": 1
  }
}
```

The db section is the basic health for a datasource. As we add more code, more entries will be added.

Working from the outside in, let's create a service class that will be responsible for orchestrating data access objects, providing transactions, etc., to accomplish the tasks of the ReST API layer. `service/ContactsService.java` looks like:

```
@Service
public class ContactsService {

    @Autowired
    private ContactsDao contactsDao;

    public List<Contact> getList() {
        return contactsDao.getList();
    }
}
```

This is really just indirection clutter right now, but this is a standard pattern that we expect to need in the future, so we start with a pass-though implementation.

Next is the Data Access Object, the thing that will actually get data, map the ResultSet into domain models, and return it to the service layer.

The JdbcTemplate consists of two parts: a query and a result row mapper. The `dao/ContactsDao.java` looks like:

```
@Component
public class ContactsDao {

    private static final String GET_CONTACTS = "SELECT * FROM CONTACTS";

    @Autowired
    private final JdbcTemplate jdbcTemplate;

    public List<Contact> getList() {
        return jdbcTemplate.query(GET_CONTACTS, new BeanPropertyRowMapper<Contact>(Contact.class));
    }
}
```

The `@Component` annotation exposes this class for use as an `@Autowired` target in the service class. Spring figures out from our MySQL pom dependency and our application.properties our target database and how to connect to it.

I purposefully made the domain model members match the backing table column names, so I can use the simplified `BeanPropertyRowMapper` to translate from a JDBC ResultSet item to a domain model. This avoids having to create a custom row mapper just for name translation.

Convention over configuration is a pretty common idiom in modern frameworks. Taking the time to organize your data for application use and defining names that are appropriate throughout the stack eliminates an absolute binary ton of code.

Finally, let's update our controller to call the service.

```
    @RequestMapping(method= RequestMethod.GET)
    public ResponseEntity<List<Contact>> getList() {

        return ResponseEntity.ok(contactsService.getList());
    }
```

There is a profound lack of testing, error checking, logging, etc. Baby steps, Padawan, Baby steps. But, on the testing front, I wholly believe in BDD/TDD, but not in a pedantic way. We are just installing the plumbing in well known patterns so far. There are no useful unit tests we could do, and no code that I need to drive with a unit test to figure out how it works. More on unit tests and integration tests later.

If you run the project, browse to `http://localhost:8080/contacts`, you will get a list of 500 contacts. Pretty cool, right?

Similar to the above implementation, we can hook up the rest of the DELETE, GET, PATCH, POST, PUT ReST methods on contacts.

## Marshalling data

Lombok turns a typically verbose and tedious domain model into a simple list of data members. Pretty nice.

Spring automatically maps request bodies to the associated type; our domain model. It doesn't even require an annotation, it just works.

Spring jdbc's BeanPropertyRowMapper automatically fills in the domain model using reflection and liberal name matching and just works because we matched our domain model property names to our database column names.

Java's brutish string handling makes our SQL ugly and piecing together a PATCH update tedious, but that is a language limit. We could work around that by implementing those pieces in groovy.

## Manual integration testing

Ad hoc testing ReST services can be oh, so tedious. Chrome has a [Postman](https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop?hl=en) app that allows a lot of flexibility in creating requests, storing those requests in collections for simple repeating, and the ability to share those collections with, well, who ever cares. This is an essential tool for ReST testing. 

## Datasouce pooling

Including the `spring-boot-starter-jdbc` dependency pulled in a Tomcat database dependency so we get mature and robust datasource connection pooling. 

## Validation

If we can validate path and requst parameters before the request handler is invoked, we can eliminate a lot of repetitive, boilerplate code. Spring uses the standard JSR 303 validation, based on original Hibernate ideas, and is among the best I have seen. We won't explore all the functionality, but we can get a taste by refactoring some code. Add the dependency to the pom with

```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
```

We are concerned with two types of validation: structural and business domain.

Structural validation ensures the data will fit in the database. Spring implements JSR 303 validation with annotations, so field length validation looks like:

```
    @Size(max=32)
    private String firstName;

```


Business domain validation ensures that we meet business constraints like every contact must have a firstName, lastName, and email. We have one domain model and six things to do with it (DELETE, GET, GET list, PATCH, POST, PUT), each may require different validation. We need a way to specify different criteria for potentially 6 scenarios. Let's clarify this:

POST, PUT:

- firstName: required
- lastName: required
- email: required

DELETE, GET: don't need a contact body, no validation required.

PATCH: By convention, omitted fields (nulls) are ignored in the update so we can't create an invalid patch - we can't null out the required fields.

Domain validation options for [Java 7 EE](https://docs.oracle.com/javaee/7/api/javax/validation/constraints/package-summary.html) and [Hibernate](https://docs.jboss.org/hibernate/validator/4.1/api/org/hibernate/validator/constraints/package-summary.html) and apparently you can use either.

We can add a `@NotBlank` to our required first name to ensure our code never sees firstNames that are null or an empty string. We can also say that the shortest firstName is 2 characters long.

```
    @NotBlank
    @Size(min=2, max=32)
    private String firstName;
```

In Spring, we can define validation groups and conditionally apply the validation based on group membership. Validation groups are marker interfaces.

```
    // Validation groups
    public interface Post {}
    public interface Put {}

    @NotBlank(groups = {Post.class, Put.class})
    @Size(min=2, max=32)
    private String firstName;

```

In the RestController, when a client passes in a Contact, we can add `@Validated` to the `@BodyParameter` to validate the object and specify the validation groups

```java
    @RequestMapping(method = RequestMethod.POST, consumes=MediaType.APPLICATION_JSON_UTF8_VALUE)
    public ResponseEntity<Contact> create(@RequestBody @Validated(Contact.Post.class) Contact contact) {
        return ResponseEntity.ok(contactsService.create(contact));
    }
```

Let's test the validation - a Validation based 400 response looks like

```
{
  "timestamp": 1457768154169,
  "status": 400,
  "error": "Bad Request",
  "exception": "org.springframework.web.bind.MethodArgumentNotValidException",
  "errors": [
    {
      "codes": [
        "Size.contact.state",
        "Size.state",
        "Size.java.lang.String",
        "Size"
      ],
      "arguments": [
        {
          "codes": [
            "contact.state",
            "state"
          ],
          "arguments": null,
          "defaultMessage": "state",
          "code": "state"
        },
        2,
        0
      ],
      "defaultMessage": "size must be between 0 and 2",
      "objectName": "contact",
      "field": "state",
      "rejectedValue": "TXX",
      "bindingFailure": false,
      "code": "Size"
    },
    {
      "codes": [
        "Email.contact.email",
        "Email.email",
        "Email.java.lang.String",
        "Email"
      ],
      "arguments": [
        {
          "codes": [
            "contact.email",
            "email"
          ],
          "arguments": null,
          "defaultMessage": "email",
          "code": "email"
        },
        [],
        ".*"
      ],
      "defaultMessage": "not a well-formed email address",
      "objectName": "contact",
      "field": "email",
      "rejectedValue": "bobthebuilder.com",
      "bindingFailure": false,
      "code": "Email"
    }
  ],
  "message": "Validation failed for object='contact'. Error count: 2",
  "path": "/contacts"
}
```

This system is robust, extensible, the validation is on the model properties making it easy to find and keep in sync, and the error messages sacrifice readability for detail. I don't know how to improve on it.

## Rich error responses

How do I help my client help themselves? 

First of all, Spring makes it easy to return any text message you want in a `ResponseEntity`. You can also return a rich error object defined by the [JSON API standard](http://jsonapi.org/format/#errors). Spring also provides `Advice` and `UnhandledExceptionMapper` handlers that could do things like map all SQL errors to appropriate codes and information about what went wrong and how to correct it.


## Logging

We need to find a logging system built with repository ingestion and query efficiency in mind; one that logs in a way that is easy for Splunk, ELK, Graylog, etc., to identify key value pairs. And one that makes logging those k-v pairs easy.

Finding a suitable logging framework is bizzarely difficult. Seems that most logging frameworks think of logging as only useful in the developer's console. If the logging framework can log in JSON, it is not extensible - you can't easily add your own k-v pairs to the log entry.

[logback-logstash-encoder](https://github.com/logstash/logstash-logback-encoder) is the only shining star I could find, but you only need one.

Spring Boot `spring-boot-starter-web` includes logback and its dependencies, so we just need to add

```
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>4.6</version>
        </dependency>
```

The standard for 12 factor apps is to log to stdout and delegate getting log entries to the log repo to the execution environment. The execution environment should consume stdout and log to a persistent store to avoid log loss during network interrruptions. The persistent store could be Redis, a rolling file log appender, a Kafka endpoint, or even RabbitMQ. 

Logback and Spring Boot provide their own default logging configuration, so we'll start with that and gradually improve it.

When logging with logback, you use the slf4j interface. Lombok provides an annotation for that, so instead of 

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ContactsController {

    static final Logger LOG = LoggerFactory.getLogger(ContactsController.class);
```

we use

```
@Slf4j
public class ContactsController {
```
which exposes a `log` member you can use in the standard way.

We need to actually log something so we can perfect our layout so...

```
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public ResponseEntity get(@PathVariable long id) {
        try {
            Contact c = contactsService.get(id);
            log.info("\"contact\": " + c.toString());
            return ResponseEntity.ok(c);
        } catch (EmptyResultDataAccessException e) {
            return ResponseEntity.notFound().build();
        }
    }
```

At this point, `mvn install` broke: the Spring Application Context failed to load complaining

```
Failed to instantiate [ch.qos.logback.classic.LoggerContext]
Reported exception:
java.lang.AbstractMethodError: ch.qos.logback.classic.pattern.EnsureExceptionHandling.process(Lch/qos/logback/core/pattern/Converter;)V
```

A little research reveals that Spring Boot left out maven dependency management for logback-core, uses logback-classic:jar:1.1.5, the logstash-logback-encoder pulls in logback-core:jar:1.1.3 and those two hate each other. Easy fix though, just downgrade the spring-boot-starter-parent to 1.3.2.RELEASE and all builds well. This issue is already fixed and will be in 1.3.4.

Back to testing: when we GET a contact, this is logged

```
2016-03-12 02:54:52.599  INFO 10005 --- [nio-8080-exec-1] c.g.s.controller.ContactsController      : "contact": Contact(id=5, firstName=Donette, lastName=Foller, companyName=Printing Dimensions, address=34 Center St, city=Hamilton, county=Butler, state=OH, zip=45011, phone1=513-570-1893, phone2=513-549-4561, email=donette.foller@cox.net, website=http://www.printingdimensions.com)
```

This is really readable but not yet appropriate for production - we want a Logstash format that is easily ingested by all log management systems. Some of the problems are that all fields are not JSON, the Contact is not JSON and some values contains spaces - ELK will give up when trying to figure out k-v pairs part of the way through. This example also hints that we will want two configurations: one for production that will be a difficult to read, compact JSON, and a human readable config for development and unit tests.

Logback has a solution for two log patterns: put logback.xml in src/main/resources with the prod config and logback-test.xml in src/test/resources with a more readable config. When you are BDD/TDD, you can see readable logs. When in production, the logs will be more Logstash friendly.

Let's start with getting our Contact into proper JSON. Lombok is providing the toString() through @Data, so we'll need to replace that with @Getter/@Setter and provide our own toString(). We'll start with a really simple implementation and worry about performance later. Include the current Gson maven dependency and then make a Contact.toString()

```
    @Override
    public String toString() {
        return (new Gson()).toJson(this);
    }
```

Now our log entry looks like

```
2016-03-12 03:32:50.920  INFO 10153 --- [nio-8080-exec-1] c.g.s.controller.ContactsController      : "contact": {"id":5,"firstName":"Donette","lastName":"Foller","companyName":"Printing Dimensions",
"address":"34 Center St","city":"Hamilton","county":"Butler","state":"OH","zip":"45011",
"phone1":"513-570-1893","phone2":"513-549-4561","email":"donette.foller@cox.net",
"website":"http://www.printingdimensions.com"}
```

The contact looks awesome. But the rest has got to go. Let's see if we can fix that. If we create a basic logging config file, using the pattern from my `Logging as a first class citizen` post, we produce log entries like

```
2016-03-12T03:51:01.577-0700 "logLevel":"INFO", "logThreadId":"http-nio-8080-exec-1", 
"logClass":"com.github.stevetarver.controller.ContactsController", "logMethod":"get",
"contact": {"id":5,"firstName":"Donette","lastName":"Foller",
"companyName":"Printing Dimensions","address":"34 Center St","city":"Hamilton",
"county":"Butler","state":"OH","zip":"45011","phone1":"513-570-1893",
"phone2":"513-549-4561","email":"donette.foller@cox.net",
"website":"http://www.printingdimensions.com"}
```

That is pretty nice and ingestable by any log aggregator, but we can do a little better using the logstash library.

The Logstash StructuredArguments allow you to append json to your log entry and produce the easiest to ingest data.

```
example log entries and what they produce
```

More StructuredArguments and Marker examples at [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder#event-specific-custom-fields)

## Epilog

Getting logging right ended up being a bigger chore than anticipated so I split it out into its own log post: [A whole product concern logging implementation](2016-04-20-whole-product-logging.md).

If fact, this whole post is getting pretty long - I am going to cut it in two. Still very interesting topics to cover, but I need to publish this and move on. Stay tuned for part 2.

If you want to check out the code up to this point, see the [GitHub project](https://github.com/stevetarver/rest-start-spring-boot). The git commits are roughly ordered by this blog post's feature additions so you can see what was added at each phase.

