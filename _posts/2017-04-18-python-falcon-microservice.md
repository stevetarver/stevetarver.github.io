---
layout: post
title:  "A Python Falcon micro-service"
date:   2017-04-18 12:23:26
tags:   python falcon kubernetes microservice

---

**TL;DR** A simple Python micro-service on Falcon

<img style="float: left;" src="/images/python.jpeg">

I want to be able to compare and contrast micro-services built in java, golang, nodejs, and python. I have a lot of time in SpringBoot, written some in golang, a couple more in nodejs, but nothing in modern Python. Seems like this is a good weekend for playtime in Python land.

I'll be using python3 to experiment with all the new language features and develop a feel for the reality of living with a python3 production service. 

I picked [Falcon](https://falconframework.org/) for a ReST framework because it satisfies basic concerns mentioned in previous posts, and:

* It is simple, minimalist
* Doesn't concern itself with server side rendering - this is all clutter for me

## Getting started

After creating a new GitHub repo [ember-falcon-mongo](https://github.com/stevetarver/ember-falcon-mongo), I set up a virtualenv as described in [my last post](http://stevetarver.github.io/2017/03/08/python-virtual-env.html).

The project structure is:

```
├── backend
│   ├── constraints.txt
│   ├── requirements-pd.txt
│   ├── requirements.txt
│   ├── src
│   │   ├── api
│   │   ├── app.py
│   │   ├── common
│   │   ├── controller
│   │   └── repository
│   └── test
```

which separates source and test code and separates source concerns into:

* `api`: inbound requests, validation
* `common`: there are always common concerns
* `controller`: service level orchestration of peer and repository methods to satisfy inbound requests
* `repository`: mongo operations

This organization looks very much like what I do in java, go, and node; it is comforting to walk into a new project when it has a familiar feel.

**NOTE**: If using PyCharm, be sure to visit your preferences to set your project interpreter to the virtualenv interpreter.

## The first route

I know the following:

* I will be using my [contacts sample data](https://github.com/stevetarver/sample-data) in a MongoDB datastore.
* I want to deploy to a Kubernetes cluster so I will want a liveness, ping, and readiness endpoint.
* I will want to do a little dependency injection for testing. 

Pulling this together, I can create my `backend/app.py`:

```python
def initialize() -> falcon.API:
    """
    Initialize the falcon api and our router
    :return: an initialized falcon.API
    """
    # Create our WSGI application
    # media_type set for json:api compliance
    api = falcon.API(media_type='application/vnd.api+json')

    # Routes
    api.add_route('/contacts', ContactsApi())
    api.add_route('/contacts/{id}', ContactApi())
    api.add_route('/liveness', Liveness())
    api.add_route('/ping', Ping())
    api.add_route('/readiness', Readiness())
    return api


def run() -> falcon.API:
    """
    :return: an initialized falcon.API
    """
    return initialize()
```

Many frameworks use annotations or decorators to tie a route to source. Falcon expects a class for each route - for example, a separate class for `/contacts` and `/contacts/{id}`. This seems a bit odd, but they justify that oddness with performance. Falcon is clearly the fastest framework out there, so we'll go with it and see what the class organization looks like with more complex routing tables later.

Naming can cause subtle issues in duck-typed languages so I want to be clear about what each component that I import is. More concretely, stubbing out files results in:

```
├── backend
│   ├── src
│   │   ├── api
│   │   │   ├── contacts_api.py
│   │   ├── controller
│   │   │   └── contacts_controller.py
│   │   └── repository
│   │       └── contacts_repository.py
```

And classes will be:

* ContactsApi
* ContactApi
* ContactsController
* ContactsRepoMongo

**NOTE**: The ContactRepoMongo class is a bit oddly named, but I think I might want to allow this service to connect to a Percona Server or RethinkDb store in the future.

Instead of decorations to tie route and source together, Falcon expects a class method matching an http verb. For example, when a `GET /contacts` is received, our route table matches that to the `ContactsApi.on_get()` method.

After defining:

```python
class ContactsApi():

    def on_get(self, req: falcon.Request, resp: falcon.Response) -> None:
        resp.body = {'hello'}
```

I can run the app with

```
PYTHONPATH=$PYTHONPATH:. \
gunicorn \
    --reload \
    'app.app:run()'
```

## Talking to the database

First, let's get a database running to provide our contacts data. The `datastore` directory contains some scripts that create a derived MongoDB docker image seeded with contacts sample data. If you run the `./build.sh` and then the `./run.sh` scripts, you'll have a local, isolated MongoDb to work with. You can read more in that directory's [README.md](https://github.com/stevetarver/ember-falcon-mongo/tree/master/datastore).

All Contacts requests will go through api -> controller -> repository. We can stub that in pretty quickly and start talking to the database.

First, add the controller to the api class:

```python
class ContactsApi():

    def __init__(self):
        self._controller = ContactsController()

    def on_get(self, req: falcon.Request, resp: falcon.Response) -> None:
        data = self._controller.get_list(req)
        resp.body = self._make_response(data)
```

The ContactsController will probably be pass-through, but we'll leave it in the execution path to provide a common look and feel with other projects.

```python
class ContactsController():

    def __init__(self):
        self._repo = ContactsRepoMongo()

    def get_list(self, req: falcon.Request) -> List[Dict]:
        return self._repo.get_list(req)

```

[PyMongo](https://api.mongodb.com/python/current/) is nothing short of lovely to work with. It handles pooling and reconnects transparently, the syntax is clear and crisp. Only one gotcha: bson ObjectIds are not serializable. Simple fix: cast to str. The string representation of the ObjectId can be used in subsequent read or modify requests.


```python
class ContactsRepoMongo():
    """
    Handles all interactions with the MongoDB contacts collection

    NOTES:
    MongoDB bson ObjectIds are not json serializable, however, you can cast
    the ObjectId to a str, which is, and use that str to construct an ObjectId
    for searching.
    """

    def __init__(self):
        # 'mongodb://localhost:27017/'
        self._uri = os.getenv('MONGO_URI', '')
        if not self._uri:
            raise ValueError('MONGO_URI env var not set; required to connect to mongodb')
        self._mongo = MongoClient(self._uri)
        self._contacts = self._mongo.test.contacts

    def get_list(self, _: falcon.Request) -> List[Dict]:
        result = []
        for c in self._contacts.find():
            c['_id'] = str(c['_id'])
            result.append(c)
        return result
```

At this point, you can restart the server and see actual data from the database. If you clone the repo, you can `./run.dv.sh`

## Make it json:api compliant

The first step in [json:api](http://jsonapi.org/) compliance is getting the right response document shape. Just a little python provides this quickly.

```python
def make_response(data_type: str, id_key: str, data: Union[Dict, List[Dict]]) -> str:
    """
    Format a normal response body IAW json:api.

    A common response body looks like:
        {
        "data": {
            "type": "contacts"
            "id": "5973ecc8c453c823e4c471d2",
            "attributes": {
                "_id": "5973ecc8c453c823e4c471d2",
                "address": "89992 E 15th St",
                ...
                },
            }
        }
    Note: the passed data ends up in the attributes section of the json:api data object

    :param data_type: generic type of data, e.g "contacts"
    :param id_key: key name in data that holds the id value
    :param data: and object or list of objects containing response data
    :return: JSON string respresentation for the response body
    """
    result = None
    if type(data) is list:
        items = []
        for item in data:
            items.append(_make_response_item(data_type, id_key, item))
        result = dict(data=items)
    else:
        result = dict(data=_make_response_item(data_type, id_key, data))
    return json.dumps(result, ensure_ascii=False)


def _make_response_item(data_type: str, id_key: str, data: Dict) -> dict:
    return dict(
        type=data_type,
        id=data[id_key],
        attributes=data,
    )
```

We can refactor the ContactsApi to make use of this common facility. We'll abstract that into a common, private base class so that both Contact classes can inherit the definition.

```python
class _ContactsApi():

    def __init__(self):
        self._controller = ContactsController()

    def _make_response(self, data: Union[Dict, List[Dict]]) -> str:
        return make_response('contacts', '_id', data)


class ContactsApi(_ContactsApi):

    def on_get(self, req: falcon.Request, resp: falcon.Response) -> None:
        data = self._controller.get_list(req)
        resp.body = self._make_response(data)

```

And while we're at it, let's make rich error information in json:api format as well. In the ContactsRepoMongo, we'll catch connection related errors and provide a common handler.

```repository
class ContactsRepoMongo(LoggerMixin):

    def get_list(self, _: falcon.Request) -> List[Dict]:
        try:
            result = []
            for c in self._contacts.find():
                c['_id'] = str(c['_id'])
                result.append(c)
            return result
        except (pymongoErrors.AutoReconnect, pymongoErrors.ConnectionFailure, pymongoErrors.NetworkTimeout):
            self._handle_service_unavailable()

    def _handle_service_unavailable(self) -> None:
        raise falcon.HTTPServiceUnavailable(
            title='Datastore is unreachable',
            description="MongoDB at {} failed to respond to ping. "
                        "This is a transient, future attempts will work "
                        "when the datastore returns to service".format(self._uri),
            href='https://www.ctl.io/api-docs/v2/#firewall',
            retry_after=30
        )
```

The default Falcon error serializer provides json:api error responses out of the box, but the names need a little touch up. We can do that pretty simply with another common function:

```python
def falcon_error_serializer(_: falcon.Request, resp:falcon.Response, exc: falcon.HTTPError) -> None:
    """ Serializer for native falcon HTTPError exceptions.

    Serializes HTTPError classes as proper json:api error
        see: http://jsonapi.org/format/#errors
    """
    error = {
        'title': exc.title,
        'detail': exc.description,
        'status': exc.status[0:3],
    }

    if hasattr(exc, "link") and exc.link is not None:
        error['links'] = {'about': exc.link['href']}

    resp.body = json.dumps({'errors': [error]})
```

And then register that serializer when we initialize our app. In `app.py`

```python
def initialize() -> falcon.API:
    """
    Initialize the falcon api and our router
    :return: an initialized falcon.API
    """
    # Create our WSGI application
    # media_type set for json:api compliance
    api = falcon.API(media_type='application/vnd.api+json')

    # Add a json:api compliant error serializer
    api.set_error_serializer(falcon_error_serializer)

    # Routes
    api.add_route('/contacts', ContactsApi())
    api.add_route('/contacts/{id}', ContactApi())
    api.add_route('/liveness', Liveness())
    api.add_route('/ping', Ping())
    api.add_route('/readiness', Readiness())
    return api
```

Since we've modified `app.py`, we will have to restart gunicorn to see the results. 

## What about health endpoints

Kubernetes exposes the concepts of liveness (am I alive) and readiness (am I ready to receive requests). Frequently these are combined into a single endpoint, but they really shouldn't be; they are separate concerns. Liveness indicates this pod's health, and readiness indicates the ability to reach upstream partners. 

There are two times when your pod may be healthy, but cannot serve requests:

* Briefly, during starup, the service may be healthy but has not yet connected to its repository.
* When an upstream partner is unreachable due to a network partition, a reschedule event, or it is unhealthy.

By separating these concerns, we avoid Kubernetes tearing down and recreating all middletier nodes when the real problem is that the datastore is unreachable. It also avoids tearing down and recreating both when there is a network partition. In addition to avoiding churn, it clarifies where the problem lies. When many pods are being recycled, it is difficult to tell where the problem lies. When many pods indicate liveness, are not recycled, and fail readiness, the logs remain easily accessible and the cause is much more clear.

The implementation for liveness is a bit tricky - how does unhealthy code tell that it's unhealthy? How does an insane person tell that they are insane? One approach is to send a synthetic call through the API layer that reaches down into the repository layer. The trick is: how do you ensure each thread is healthy? The thread you test may be healthy while there are 1 or 2 that are not. In golang, I can have a health channel linked to each thread in a global list. For this python code, a good solution will take a bit of thought and I am going to punt for now.

The implementation for readiness should send a lightweight request to each of its upstream partners to prove it can connect to them.

The readiness implementation introduces another concern: how do I check that I can reach my upstream partners without introducing additional load to that component or its upstream partners? Our service is not responsible for liveness or readiness of upstream nodes; Kubernetes is responsible for that. We just need to ensure the network path exists; a ping.

The final concern is how to report the results. We could return any 500 error, but we want to allow for clients to implement a reasonable retry strategy. If we can execute our logic, but cannot connect to upstream partners, we will return a 503 SERVICE UNAVAILABLE. If we can't answer the call, gunicorn will return a 500 for us. Clients can key off the response code to see if retry makes sense.

Lastly, can we include any information to help human operators diagnose problems? We are making calls to the database so perhaps include timing information in case the database is overloaded or we have some weird routing and latency is a problem. And, of course, we will return everything in a json:api compliant response.

Here is the current implementation which separates the three concerns, but punts on the liveness implementation:

```python
class Liveness(object):
    """
    Are we functional? Or should our scheduler kill us and make another.

    We will pump a get message through our service to prove all parts viable

    Return 200 OK if we are functional, 503 otherwise.
    """
    def on_get(self, _: falcon.Request, resp: falcon.Response):
        start = datetime.now()
        _ = ContactsController().find_one()
        duration = int((datetime.now() - start).total_seconds() * 1000000)

        resp.body = make_response('liveness',
                                  'id',
                                  dict(id=0,
                                       mongodb='ok',
                                       mongodbFindOneDurationMicros=duration))


class Readiness(object):
    """
    Are we ready to serve requests?

    Check that we can connect to all upstream components.

    Return 200 OK if we are functional, 503 otherwise.
    """
    def on_get(self, _: falcon.Request, resp: falcon.Response):
        start = datetime.now()
        ContactsRepoMongo().ping()
        duration = int((datetime.now() - start).total_seconds() * 1000000)

        resp.body = make_response('readiness',
                                  'id',
                                  dict(id=0,
                                       mongodb='ok',
                                       mongodbPingDurationMicros=duration))


class Ping(object):
    """
    Can someone connect to us?

    Light weight connectivity test for other service's liveness and readiness probes.

    Return 200 OK if we got this far, framework will fail or not respond
    otherwise
    """
    def on_get(self, _: falcon.Request, resp: falcon.Response):
        resp.body = make_response('ping',
                                  'id',
                                  dict(id=0))
```

## Epilog

I'm feeling pretty positive about python as a micro-service language right now. It's pretty enjoyable work:

* Code is clean and concise
* Type hinting provides low risk refactors and good code completion hints
* I don't have to create the boiler plate models to deserialize requests and serialze responses - woohoo!


