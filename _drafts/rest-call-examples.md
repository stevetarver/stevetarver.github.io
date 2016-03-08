---
layout: post
title:  "Install Homebrew"
date:   2015-05-09 19:32:15
categories: rest
---

A key part of the 12-factor app ecosystem is polyglot languages and polyglot datastores. 

If you have a choice of many languages, how do you choose?

Languages can offer an inherent simplicity, but supporting libraries can be the deciding vote.

What criteria should be considered?

- language simplicity
- an amazing library
  - how much does the library do for you
  - does it do the right things for you
- validation
- logging - is it done automatically by the library
- datastore access
- rich error responses

Not an exhaustive comparison, just a listing of some popular languages and frameworks to give a little flavor and get the ball rolling.


# GO ([gorilla/mux](https://github.com/gorilla/mux))

Server code

```go
package main

import (
    "fmt"
    "html"
    "log"
    "net/http"

    "github.com/gorilla/mux"
)

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    router := mux.NewRouter().StrictSlash(true)
    router.HandleFunc("/", Index)
    log.Fatal(http.ListenAndServe(":8080", router))
}
```

Run Server

```go
go get
```




# Groovy ([Spark](http://sparkjava.com/))

Download and install an SDK manager and install groovy

```
curl -s http://get.sdkman.io | bash
sdk install groovy
```

Server code

```groovy
@Grab(group = 'com.sparkjava', module = 'spark-core', version = '2.3')
import static spark.Spark.*

get '/hello', { req, res -> 'Hello World!' }
```

Run server

```
groovy Server.groovy
```


# Java ([Spring Boot]())

_Note: although the code is brief, there is significant build and configuration_

_Note: you could use the spark framework here for a simpler implementation_

[GitHub project](https://github.com/spring-guides/gs-actuator-service)

Server code



```java
package hello;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping("/hello")
public class HelloWorldController {

    @RequestMapping(method=RequestMethod.GET)
    public @ResponseBody Greeting sayHello(@RequestParam(value="name") String name) {
        return "Hello World!";
    }
}
```

```java
package hello;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class HelloWorldConfiguration {

	public static void main(String[] args) {
		SpringApplication.run(HelloWorldConfiguration.class, args);
	}

}
```

Run server

```
java -jar build/libs/gs-spring-boot-0.1.0.jar
```


# Python ([Falcon](http://falconframework.org/))

Server code

```python
import falcon
import json
 
class QuoteResource:
    def on_get(self, req, resp):
        resp.body = json.dumps("Hello World!")
 
api = falcon.API()
api.add_route('/quote', QuoteResource())
```

Run server

```
gunicorn sample:api
```


# Ruby ([Sinatra](http://www.sinatrarb.com/))

Server code

```ruby
require 'sinatra'

get '/hi' do
  "Hello World!"
end
```

Run server

```
ruby hi.rb
```

