---
layout: post
title:  "Postman: Waiting on pod ready"
date:   2018-03-27 11:11:11
tags:   k8s kubernetes helm postman service
---

**TL;DR**. A method for delaying Postman tests until your pod is ready.

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/postman.jpg">

When deploying first Kubernetes apps, you may be tempted to start the deployment testing stage when your service "should" be ready - perhaps waiting for your service readiness and liveness probe delays to expire. When you start running into deploy failures due to test subject not ready, you may add a delay buffer that helps for a while - but that doesn't address the root of the problem.

As your service and cluster grow, deploy test failures due to pod not ready increase:

* As the cluster pod count grows, it may take more time to deploy the service and move it to the ready state.
* As the service grows, your readiness and liveness probe delays will change and you will probably forget to update your build pipeline script to reflect these changes.

These errors are frustrating: they needlessly load the build infrastructure, increase the build and deploy time, and the false failures erode confidence in your build system and your cluster in general.

This month, we will answer the question: "How long shall I wait for my pod to be ready, and how will I know it is ready?"

## Strategy

I use [Postman](https://www.getpostman.com/) for all black-box test. Consolidating on a single tool is great because I can expect that operations will develop some skill with the tool, I can provide less documentation, and those tests can be used locally, during deployment validation, and at any time for operational verification.

Kubernetes provides two health probes:

* **readiness**: Is your service ready to receive requests? Has it completed long-running startup tasks and are all upstream partners available?
* **liveness**: Is your service still alive? Is it hung in an internal loop, deadlocked, etc.?

Once a service is ready and live, it is scheduled to receive messages. The trick is to determine when we can reach the service, and then when it is ready and live - and which to call first. Since we need to connect to the service for it to answer the readiness and liveness questions, connectivity must be the first check. Since the readiness check returns false until long-running startup tasks are complete, it is the second check, leaving liveness the third check.

**Notes**:

* I always implement a readiness probe, even if not currently used, so that all services have the same health endpoints.
* I always implmenet a ping probe, as a lightweight way for a service to check upstream partners and so that all services have the same health endpoints.

## Failed implementations

Postman provides a [sandbox](https://www.getpostman.com/docs/v6/postman/scripts/postman_sandbox) for script execution; the problem looked like it had a simple async/await solution. After a couple of attempts, I found that the sandbox was not "awaiting".

The sandbox also includes some of my favorite libraries like lodash and backbone, both having promise-type functionality, so I tried those and ran into the same problem - script exit prior to promise resolution.

After some digging and playing, it occurred to me that the sandbox parser must recognize a reason to not terminate the script during promise fulfillment: the sandbox does not recognize async/await or the lodash promises, but it does recognize `setTimeout()`.

## Implementation

I am really interested in general solutions: they eliminate the cognitive overhead of trying to figure out the nuances of one-off solutions. The "right" solution to this problem is a general strategy for all services.

Each of the Kubernetes health probes must be tested because they answer different questions. Each may run into the same "pod-not-ready" problem, so each should have some pod-ready testing. I use the above order for testing, and place the following code in each "prerequest" script:

```javascript
// Get the URL for this step - allows us to copy/paste this code into any step
var url = request['url'].replace('{{host}}', postman.getEnvironmentVariable('host'));
var retryDelay = 10000;
var retryLimit = 10;

function isServiceReady(retryCount) {
    pm.sendRequest(url, function (err, response) {
        // Service endpoint is not up yet - any kind of connection error
        var notUp = (null !== err);
        // Service is up, but still initializing or waiting on upstream partners
        // NOTE: there is always a response object, but .code only exists if we connected
        var notReady = (!response.hasOwnProperty('code') || response.code !== 200);

        if(notUp || notReady) {
            if (retryCount < retryLimit) {
                if(notUp) {
                    console.log('Service is down. Retrying in ' + retryDelay + 'ms');
                } else {
                    console.log('Service is not yet ready. Retrying in ' + retryDelay + 'ms');
                }
                // If not ready, retry this function after retryDelay
                setTimeout(function() {
                    isServiceReady(++retryCount);
                }, retryDelay);
            } else {
                console.log('Retry limit reached, giving up.');
                postman.setNextRequest(null);
            }
        }
    });
}

isServiceReady(1);
```

Let's walk through this implementation:

```javascript
// Get the URL for this step - allows us to copy/paste this code into any step
var url = request['url'].replace('{{host}}', postman.getEnvironmentVariable('host'));
var retryDelay = 10000;
var retryLimit = 10;
```

As the comment suggests, the first line is indirection so we can easily copy & paste this code into any preRequest block - it creates a test URL from the test script so it works equally well for the readiness, liveness, and ping probes. The next two lines setup our retry limits: 10 retries with a 10s delay. 

There is a tiny bit of art involved with defining retry limits: 

* We want the total duration to be about twice what we think the longest pod startup time.
* We don't want the stack to grow large - retries are nested promises.
* We want success to be reported with minimal delay.

The beginning of the `isServiceReady()` function collects all decision making information:

```javascript
function isServiceReady(retryCount) {
    pm.sendRequest(url, function (err, response) {
        // Service endpoint is not up yet - any kind of connection error
        var notUp = (null !== err);
        // Service is up, but still initializing or waiting on upstream partners
        // NOTE: there is always a response object, but .code only exists if we connected
        var notReady = (!response.hasOwnProperty('code') || response.code !== 200);
```

`pm.sendRequest` makes an async call to our service. If there was any kind of error, the service is not reachable. If the response is not 200, we need to wait a little longer.

The retry decision is straight-forward:

```javascript
        if(notUp || notReady) {
            if (retryCount < retryLimit) {
                if(notUp) {
                    console.log('Service is down. Retrying in ' + retryDelay + 'ms');
                } else {
                    console.log('Service is not yet ready. Retrying in ' + retryDelay + 'ms');
                }
                // If not ready, retry this function after retryDelay
                setTimeout(function() {
                    isServiceReady(++retryCount);
                }, retryDelay);
            } else {
                console.log('Retry limit reached, giving up.');
                postman.setNextRequest(null);
            }
        }
```

If there is no error condition, we are ready, and we let the function run to completion. If the service is notUp or notReady, we check if our retry budget is exhausted. If not, we retry again, else, we abort.

Note that we have taken some time to groom our log entries so it is easy to diagnose failed builds and tell what is going on when the test is running against production loads.

Finally, the function call:

```javascript
isServiceReady(1);
```

Calls our main function with an initial retry count.

## Epilog

This simple, cut & paste health endpoint addition to your test suites will make your tests much more reliable in all situations.

