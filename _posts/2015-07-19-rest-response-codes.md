---
layout: post
title:  "ReST Response Codes"
subtitle: "Develop criteria for comparing ReST frameworks for cloud applications."
date:   2015-07-19 17:09:09
categories: error-code response-code rest
---

ReST uses HTTP status codes as operation response codes. HTTP status codes are sufficiently comprehensive for ReST, yet target no specific domain; each service provider must figure out how to map their infrastructure and logical errors into the status code set.

This article focuses on a fairly narrow domain: ReST services within a SOA network. It attempts to present a minimal yet expressive set of error codes that allow collaborators to understand current state of upstream providers and react appropriately.

## Goals

Appropriate HTTP status code use provides rich error information that should improve each part of the product life cycle. The following goals are used as a guide to developing an "appropriate" set.

- Minimal: The entire set should be as small as possible: simple to understand, simple to handle in code
- Expressive: Each code should imply a distinct client or support action or significantly clarify the API
- Consistent: Every service should be able to use the same set of codes, with the same meanings so every client developer learns one idiom and applies it to every application


Specific benefits for the product team:

- The client developer has rich API error information
  - simplifies fault identification and correction in code development and testing
  - eliminates time lost tracking down another developer or log to understand the error
  - enables common error handling routines instead of per API/method error handling
  - improves productivity, more enjoyable work experience
- The service developer has response code guidance
  - same error codes for the same error conditions provided by all developers in all APIs
  - acts as a checklist for all errors a developer should look for
  - encourages developing common error detection and handling routines
  - less lost time researching and explaining errors to client developers and testers
  - improves productivity, more enjoyable work experience
- The quality engineering developer has a conformance specification
  - can produce and test for all errors in new code, verify compliance
  - can create common error handlers instead of discovering and implementing separate error handlers for each API
  - improves productivity, more enjoyable work experience
- Support staff has status code definitions
  - can more easily/quickly identify specific issue faults and locate root cause
  - can monitor for poorly performing APIs using common values instead of researching each API to understand its error meanings
  - can create common log monitor dashboards, aggregating many APIs, for evaluating entire system performance
  - speeds remediation, reduces work, more enjoyable work experience

## References

- [Apigee: What about errors](https://blog.apigee.com/detail/restful_api_design_what_about_errors)
- [SOA Bits: Error Handling](http://soabits.blogspot.com/2013/05/error-handling-considerations-and-best.html)
- [Stormpath: Spring ReST error best practices](https://stormpath.com/blog/spring-mvc-rest-exception-handling-best-practices-part-1/)
- [w3.org: RFC 2616 sec 9](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html)
- [w3.org: RFC 2616 sec 10](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
- [IANA: HTTP Status Codes](http://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml)
- [Wikipedia: HTTP Status Codes](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes)
- [IETF: RFC 6585](http://tools.ietf.org/html//rfc6585)
- [IETF: RFC 5789](http://tools.ietf.org/html/rfc5789)

## HTTP Methods

Many APIs and articles have different opinions about how to use HTTP verbs, so to start, let's define the verbs and their meanings in this domain.


| Verb | Description | 
|----|----|
|GET | Read. Safe (no modifications allowed on GET). Uses a URI path element to select from a resource collection or query parameters to return a list of resources |
| DELETE | Delete. Idempotent. Remove the specified item from the collection so it is not visible in other operations. Frequently, this is a soft delete where the item is marked inactive, but depends on resource implementation. |
| PATCH | Update. Modify subset of fields in a resource. Implies that unspecified fields are not modified.|
| POST | Create. Introduces a new resource into a collection.|
| PUT | Update. Idempotent. Modifies a whole resource, providing a new version of that resource. Implies that unspecified fields are nulls.|


**NOTE**: The PATCH verb significantly clarifies update operations, especially for Enterprise ReST, and has seen significant early adoption although RFC 5789 still in a "proposed" state. Some older web proxies and containers cannot generate or consume a PATCH method. If you are not subject to this constraint, you should use PATCH because of the clarity it provides: PUT always means resource replacement (e.g. user edits from a browser) and PATCH always means selected field update (services can set selected fields without first fetching the resource, then merging changes).

**Using PATCH**: If using Java, you will likely have to create your own PATCH annotation (trivial), but this will integrate cleanly with a standard spring+jersey ReST infrastructure - simply annotate your ReST interface method with the new @PATCH.


## Response Codes

This list contains all HTTP response codes that should be used in ReST services; an expressive, yet minimal, list arranged in blocks

- 2XX Success
- 4XX Client Errors
- 5XX Service Provider Errors

**Note**: In Java, use javax.ws.core.Response Status enums when possible for clarity and to avoid magic numbers. 

|Code|Enum|Description|
|---|---|---|
|200|OK|General success. No entity returned. <br/>Applies to: GET, DELETE, PATCH, PUT
|201|CREATED|New resource created. Unique identifier returned. <br/>Applies to: POST, PUT
|202|ACCEPTED|Action accepted but not yet enacted. E.g. In asynchronous processing reliable acquisition where a message is validated and placed on a queue for another actor to complete. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|400|BAD_REQUEST|Client supplied request has invalid parameters. This indicates an error in client code or data and should be logged so clients can fix their code. Provide specific details on the parameters that failed validation and why. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|401|UNAUTHORIZED|The request requires authentication and no token was provided, or, the authentication token failed validation. IOW: "Unauthorized" really means "Unauthenticated";  "You need valid credentials for me to respond to this request". <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|403|FORBIDDEN|The authenticated user does not have authorization to submit the request. IOW: "Forbidden" really means Unauthorized "I understood your credentials, but so sorry, you’re not allowed!”. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|404|NOT_FOUND|The resource cannot be found. If the URI contains multiple path variables, the response entity should indicate which item was not found. E.g. /incidents/INC000123/configItems/OI-0001/policies, indicate if the incident or the configItem was not found. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|405| |Method not allowed for this resource. E.g. If all resources can be DELETEd but one, return 405 when the client tries to DELETE that restricted resource to clearly indicate the restriction is intentional. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|409|CONFLICT|The request could not be completed due to a conflict with the current state of the resource. E.g. Submitting a resource change with a base revision earlier than the current revision; similar to a database dirty write. This implies there is a resource version parameter (e.g. timestamp) and it was out of date. <br/>Applies to: PATCH, PUT.
|429| |Too many requests. The user has sent too many requests in a given amount of time. Used with rate limiting, expected to be implemented through Layer 7. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|500|INTERNAL_SERVER_ERROR|Any local or upstream generated error that the service does not understand. Repeated submission of this request will fail. Requires developer changes to fix the fault or catch the upstream provider error. Return rich error information including stack traces for internal only services to speed remediation, but never to external clients. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|501| |Method not implemented. Could use for partial implementations where an implementation is planned but not yet complete. <br/>Applies to: GET, DELETE, PATCH, POST, PUT
|503|SERVICE_UNAVAILABLE|Any upstream connection refused, timeout, or other service unavailable error. An upstream provider is unable to handle the request due to a temporary overloading or maintenance. Used to distinguish code problems that cannot be retried from connectivity problems which could resolve themselves in a small time frame allowing a retry strategy. <br/>Applies to: GET, DELETE, PATCH, POST, PUT

## Implementation Guidelines

### 200 vs 201: Return 201 on create success

When you create a resource within a collection that has a well known identifier, you use PUT. Consider a provisioning or management API for Windows that wants to record the applications installed on the C drive. One might imagine a URI like

{% highlight bash %}
/departments/POR0000022222/servers/SL2LS431785/drives/C
{% endhighlight %}

The caller audits the C drive and PUTs the audit results to the above URI. The only way the caller can tell if they created a new resource or modified an existing one is by the response code; otherwise the calls look exactly the same. The RFC created a special code for this case to distinguish between general success and resource creation. If the caller receives a 201, they know that they created a new resource. If they receive a 200, they know they modified an existing one.

When you create a resource without a well known identifier, you use POST and the service returns a unique identifier for the new resource.

To ensure 201 means resource created for every verb (PUT, POST), and to adhere to the RFC, return 201 when a resource is created either by PUT or POST.


### 404: Do not return NOT_FOUND for empty search results

A ReST API is organized as a collection of resources and selectors. The URI is a path through that hierarchy that allows the client to manipulate a specific resource.

{% highlight bash %}
/companies/CPY0000011111/departments/POR0000022222/employees/PPL0000033333
{% endhighlight %}


If the client misspells "companies", jersey will return 404 for you because it will not know which part of your code to invoke.
Similarly, if you use a URI element (path variable) to locate an item in a collection (look up company CPY0000011111) and that company does not exist, you should return a 404.
In both cases, no operation could be performed, because the resource could not be found.

Although you had to look up the company, this is not a search because, in our domain, all elements of a resource path are assumed to exist. The client should have already verified that the ids are valid.
Use query params to provide a search facility where a resource may not exist

{% highlight bash %}
/companies/CPY0000011111/departments/POR0000022222/employees?id=PPL0000033333
/companies/CPY0000011111/departments/POR0000022222/employees?lastName=smith
{% endhighlight %}

In a search, if the operation succeeds, a 200 is returned even when no items matching the search criteria in the collection.

The specification is clear that 404 is reserved for identifying an invalid Request-URI (the part of the URL between host[:port] and optional ?).

A 404 is not appropriate above because the requested resource (/companies/CPY0000011111/departments/POR0000022222/employees) exists and was found, and the operation was applied successfully.


### 405, 501: Clearly identify inappropriate/unimplemented methods

The default spring+jersey on Tomcat implementation returns a NullPointerException with a stack trace when you send an HTTP verb that is not implemented.

Provide an implementation for each of DELETE, GET, PATCH, POST, PUT, returning 405 when the verb is inappropriate and 501 when it will be in a future implementation.

**TODO**: Provide example of how to configure Spring to do this automatically.


### 409: Protecting against dirty writes

Scenario: Mohammad gets a customer email change request, opens his browser, pulls up the customer, and suddenly gets the need for some go-juice. Meanwhile, Surya gets a phone number change request from the same customer which she completes immediately. Mohammad comes back and finishes the email change and pushes the submit button. He also just reverted the new phone number to the previous phone number.

If multiple modifications for a single resource can overlap, or if it is essential that this never happen, you should design a versioning system for your resource.

A database example for the scenario above:

- Add a TimeDate column (millis since epoch)
- Add a trigger that updates that column on record change.
- Include the timestamp column in your resource representation as version
- On PUT to a resource, first verify that the resource version matches the database version
- On match fail, return 409

409 CONFLICT indicates to the client that the base version of the resource they submitted has been changed since they fetched it. GUI clients should present the data to the user for edit, other services should request a fresh version of the resource and merge in their changes.


### 500 vs 503: Separating connectivity from code errors

When 500 (code errors) are clearly separated from 503 (connectivity errors):

- each 500 error can be treated as a work item for a developer
- each 503 can be treated as remediation issue for support
- 503 can be used to selectively implement retry strategies and eliminate commonly dropped service calls eliminating manual cleanup
- 503 can be monitored in the log and when a threshold is exceeded, a trap can be sent to support and cases created/remediated quickly
- 500 and 503 can be charted in splunk, including aggregating multiple services and methods to show spikes in code and connectivity problems


If 500 errors can be treated as developer work items, exceptions from upstream providers should be avoided by validating/conditioning the message or fixing the upstream provider code. If that is impossible, those types of 500 errors should be marked so they can be easily eliminated from the work list.


### 503: Provide for retry strategies

When a service layer is provided on top of a database, an API method could fail if the service could not get a database connection. This situation could clear up quickly and a subsequent try could succeed. This pattern is seen frequently and not limited to databases.

Inability to connect to an upstream service provider (WS, database, etc) should be identified and returned as a separate status code (503).

Separating connectivity errors from general 500 errors in the initial design allows applying a Layer 7 retry policy in minutes to remediate production capacity and latency problems in the future.

This is a significant boon since these failures must otherwise be cleaned up manually, which can be tedious and time consuming, and diminishes our reputation with our customers.

**WARNING**: Retry policies must be implemented cautiously. Retry with some strategies and in situations could exacerbate the problem. In cloud scale architectures, circuit breakers are a good defense against excessive retries.
