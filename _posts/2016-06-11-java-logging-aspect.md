---
layout: post
title:  "A logging aspect in java"
date:   2016-06-12 19:32:15
tags: logging java spring logback

---

**TL;DR** Implementation of a logback logging aspect for API metrics collection.

<img style="float: left;" src="/images/java_logo.png">

Every Enterprise IT migration from the traditional to the cloud is different and every corporation is at a different place in that path. What if you like the ideas in _Logging as a first class citizen_ and want to use the implementation and concepts in _A whole product concern logging implementation_? I gave a traditional deployment to tomcat solution and a cloudy Spring Boot solution, but what if you are somewhere in the middle?

Immutable infrastructure presents the notion of creating a server and then never modifying it. The most basic form might be a manual Tomcat install and strict configuration control: no modifications except for app deploys. This is great operational hygiene and really simplifies things like the Tomcat upgrade I am going through; any step closer to immutable architecture is worth taking.

In _A whole product concern logging implementation_, I listed the jars and configuration files you have to add to the Tomcat distro for access log event mining to produce API metrics. What if you didn't have to do that; you could deploy any vanilla Tomcat, put your war in the webapps directory and your app would just work? Then, when the next Tomcat security vulnerability is discovered, your update becomes:

1. create servers
1. install java/Tomcat
1. copy in your app
1. add new servers to load balancer pool
1. age old servers out of load balancer pool

You eliminate concern over:

* missing incremental configuration changes that turned your app server into a snowflake
* long maintenance periods (based on how long each server takes to update)
* servlet container update failures and having to fix them during the maintenance period
* testing servers during the maintenance period

Let's up our operational hygiene game and eliminate the need to deploy jars and configuration to Tomcat by introducing a logging aspect to generate API call metrics.

**NOTE** See _A whole product concern logging implementation_ for a more detailed description of our metrics generation and use of logback.

## What?

Logging aspects address concerns that apply to many classes. Some of the most common are transactions around database operations and authentication/authorization verification. There are some methods in many classes that you want to include in a transaction or apply security checks to; you want to apply code across a collection of classes instead of within a class hierarchy. This is the arena of aspect oriented programming.

Logging fits well in this category. Our goal is: for every exposed API call, log the request with inbound parameters, the exit with call duration and status, and exceptions with call duration and status. Every API entry and exit is logged to provide complete metrics and allow efficient and robust queries to back operational dashboards.

Aspects use Pointcuts to identify where you want to apply the code. The simplest (and thus most efficient) pointcuts are package or annotation based. If all of our Resource or Controller classes, and nothing else, were in, or under, one package, a package pointcut would work. But what if other classes snuck into those packages? Those public methods would be included in your API metrics dashboards. Or what if one of your Resource classes has a public method that you don't want logged? An annotation based pointcut solves both these problems with the only cost being to define the annotation and add it to a class or some methods. A little more work, but much more robust solution.

see [Spring Aspects](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html)

## The Annotation

The annotation is pretty simple: ElementType TYPE and METHOD allow the annotation to be applied to the class and method, respectively.

```java
package com.github.stevetarver.logging;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface LoggedApi {}
```

## The Aspect

In this aspect code, notice:

- @Slf4j lombok annotation providing the `log` member
- StructuredArguments static import that provides value(), keyValue()

```java
package com.github.stevetarver.logging;

import lombok.extern.slf4j.Slf4j;
import net.logstash.logback.argument.StructuredArgument;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.LinkedList;
import java.util.List;

import static net.logstash.logback.argument.StructuredArguments.keyValue;

@Slf4j
@Aspect
@Component
public class LoggingInterceptor {

    private static final List<String> EXCLUDED_PARAMS = Arrays.asList("authentication");
    private static final String EXECUTION_POINT = "executionPoint";
    private static final String LOGGED_API_NAME = "loggedApiName";
    private static final String DURATION = "duration";
```

These are the pointcut definitions discussed above. Unlike package based pointcuts, annotation pointcuts are processed for definition and execution. We need to add a point cut for interesting types of execution; we don't want to log when the annotations are processed.

```java
    // Matches annotation on class
    @Pointcut("within(@com.github.stevetarver.logging.LoggedApi *)")
    public void loggedApiClass() {}

    // Matches annotation on method
    @Pointcut("@annotation(LoggedApi)")
    public void loggedApiMethod() {}

    // Matches all public methods
    @Pointcut("execution(public * *(..))")
    public void publicExecution() {}

    // Matches any method
    @Pointcut("execution(* *(..))")
    public void anyExecution() {}
```

Call duration is a key metric. Normally we would break this method up into a before, after returning and after throwing advice, except that we need to capture call duration on all exit points, hence the `@Around` advice.

For logging the API call invoke, we defer to the private method, `extractParameters()` described below. It returns a `StructuredArguments[]` which we can use in the `.info()` override. Interesting to note that you can't add variables in this form. Logback won't log the parameters but it also won't complain, not thier style, so it is an easy error to miss.

We tag API call request, response, and exception with an `executionPoint` for easy identification of each in log management system queries. For example, if we want to chart API call counts, we can search for an `executionPoint: request`. If we want call duration, we know that lives in responses.

The `methodName` us used as log entry `loggedApiName` for clean dashboard series naming. This code concatenates the class name and method to produce something like `ContactsController.get`. If your method names are more verbose, say `ContactsController.getContacts`, you could simply use the method name.


```java
    // @LoggedApi on a class means all public methods
    // @LoggedApi on a method means just that method, ignoring visibility
    @Around("(loggedApiClass() && publicExecution()) || (loggedApiMethod() && anyExecution())")
    public Object logApi(ProceedingJoinPoint joinPoint) throws Throwable {

        final String methodName = joinPoint.getSignature().getDeclaringType().getSimpleName()
                + "." + joinPoint.getSignature().getName();
        // NOTE: do not add args to this call, the structured args will not be logged
        log.info(methodName + " request", extractParameters(methodName, joinPoint));
```

The need to capture timing for normal and exceptional execution makes this code a bit awkward.

If the response was a ResponseEntity, we extract the status code and log it.

This code assumes global exception handlers for exceptions throw out of the ReST controller layer which provides a common conversion from exception to ReST ResponseEntity. Otherwise, we would not expect exceptions to cause a ReST API exit. It also provides a way to identify unhandled exceptions; when no global exception handler exists.

```java
        final long start = System.currentTimeMillis();
        try {
            Object response = joinPoint.proceed(joinPoint.getArgs());

            int status = 0;
            if(response instanceof ResponseEntity) {
                status = ((ResponseEntity)response).getStatusCode().value();
            }
            log.info("{} response", methodName,
                    keyValue(LOGGED_API_NAME, methodName),
                    keyValue(EXECUTION_POINT, "response"),
                    keyValue(DURATION, System.currentTimeMillis() - start),
                    keyValue("status", status));

            return response;

        } catch (Throwable throwable) {
            log.info("{} exception: {}", methodName, throwable.toString(),
                    keyValue(LOGGED_API_NAME, methodName),
                    keyValue(EXECUTION_POINT, "exception"),
                    keyValue(DURATION, System.currentTimeMillis() - start));
            throw throwable;
        }
    }
```
This helper exists solely to extract API parameters from an API request. As such, and to avoid the varargs complications mentioned above, we add other key-value pairs that belong with the request.

```java
    /**
     * Helper method for extracting request parameters.
     * Returns a StructuredArgument[] that can be used as a vararg.
     * NOTE: We have to collect all kv pairs here because varargs will expand
     *       an array of type, but you cannot have any other args in the log.info() call.
     *       Logback will not complain, it will simply not log the StructuredArgs.
     */
    private StructuredArgument[] extractParameters(String methodName, ProceedingJoinPoint joinPoint) {

        final String[] names = ((MethodSignature)joinPoint.getSignature()).getParameterNames();
        final Object[] values = joinPoint.getArgs();

        List<StructuredArgument> result = new LinkedList<StructuredArgument>();
        result.add(keyValue(EXECUTION_POINT, "request"));
        result.add(keyValue(LOGGED_API_NAME, methodName));

        for(int i=0; i<values.length; i++) {
            if(!EXCLUDED_PARAMS.contains(names[i])) {
                result.add(keyValue(names[i], values[i]));
            }
        }

        return result.toArray(new StructuredArgument[result.size()]);
    }
}
```

## What a logged API call looks like

If we had a ContactsController, the create method would log something like this for the create method.

```json
{
  "@timestamp": "2016-06-11T22:57:39.111-06:00",
  "@version": 1,
  "executionPoint": "request",
  "loggedApiName": "ContactsController.create",
  "logLevel": "INFO",
  "logMessage": "ContactsController.create request",
  "logThreadId": "http-nio-8080-exec-1",
  "logClass": "c.g.s.c.ContactsController",
  "logMethod": "create",
  "contact": {
    "id": null,
    "firstName": "Bob",
    "lastName": "The Builder",
    "companyName": "Bob's Construction",
    "address": "1234 Beaver lane",
    "city": "Austin",
    "county": "Pitkin",
    "state": "TX",
    "zip": "81615",
    "phone1": "3136364482",
    "phone2": "3136369982",
    "email": "bob@thebuilder.com",
    "website": "http://bobthebuilder.com"
  }
}
```


## Epilog

This aspect produces some basic metrics that can significantly improve your understanding of production performance via log management system dashboards. It does this in a way that allows your application to be a single deployable artifact to improve your operational hygiene - no Tomcat configuration needed.
