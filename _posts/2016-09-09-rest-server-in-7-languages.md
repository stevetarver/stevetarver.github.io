---
layout: post
title:  "ReST servers in 7 languages"
date:   2016-09-09 19:32:15
categories: rest golang gorilla groovy spark java springboot python falcon ruby sinatra node express bash
---

If you're in the cloud, you should be building [12-factor apps](https://12factor.net/). Emancipation from Enterprise IT frees you from monoliths and dogmatic language requirements. Then using microservices carves your solution into small, isolated chunks that encourage exploration.

So, if all language and frameworks are an option, how do you choose? The 12-factor app manifesto provides a pretty good scale; strong support for those tenants is a base. In addition:

- language simplicity
- amazing libraries
    - how much do the libraries do for you
    - do they do the right things for you
- validation 
    - is it configuration based and extensible
- logging 
    - is it done automatically by the library
    - is it easily consumed by the log aggregator
- datastore access simplicity
- json:api support
- rich error responses
- how much fun is it to work in

Not a bad list of concerns, it omits things that most already do well ... but how do you eat that elephant?

What if we just started with: _What does the simplest ReST server look like?_

## Bash

Server code

```shell
#!/bin/sh
while true; do { echo -e 'HTTP/1.1 200 OK\r\n'; echo "Hello World"; } | nc -l -p 8080; done
```

Run Server

```shell
./bash-web-server.sh
```

Although this seems a bit silly, and is difficult to extend in a meaningful way, it is pretty cool to stick in an Alpine docker container for cluster deployment testing, etc. - it's only 4.81 MB.


## GO ([gorilla/mux](https://github.com/gorilla/mux))

Server code

```go
package main

import (
    "fmt"
    "net/http"
    "github.com/gorilla/mux"
)

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    router := mux.NewRouter().StrictSlash(true)
    router.HandleFunc("/", Index)
    http.ListenAndServe(":8080", router)
}
```

Run Server

```go
go run
```


## Groovy ([Spark](http://sparkjava.com/))

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


## Java ([Spring Boot]())

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

NOTES:

* although the code is brief, there is significant build and configuration
* you could use the spark framework here for a simpler implementation


## JavaScript ([Node/Express](http://expressjs.com/))

Server code

```js
var express = require('express');
var app = express();

app.get('/', function (req, res) {
  res.send('Hello World!')
});

app.listen(3000, null);
```

Run server

```
node .
```

## Python ([Falcon](http://falconframework.org/))

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


## Ruby ([Sinatra](http://www.sinatrarb.com/))

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

