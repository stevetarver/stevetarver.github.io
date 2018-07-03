---
layout: post
title:  "Pipeline Design Pt 4: A custom Jenkins pipeline"
date:   2017-11-14 11:11:11
tags:   ci cd pipeline jenkins automation

---

**TL;DR** Create a custom, generic Jenkins pipline

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/jenkins.png">

From previous articles, we understand our build system constraints, target environments, infrastructure components and their interactions; now we need to figure out how to tie all that together. We'll use Jenkins as both build and deploy tool because many are familiar with it, especially in the Enterprise IT realm, and their addition of a declarative pipeline makes this a pretty attractive offering.

Googling 'jenkins pipeline' will hint at the first problem we want to solve: everyone has a different way of implementing a build and deploy pipeline. Instead of a cacophony of developer best ideas, we want a single process that is good enough for all of our developers. This allows us to ensure:

* a single, consistent build and deploy process is used
* corporate standards are checked and validated
* compliance concerns are met
* we can easily augment the pipeline with common concerns and all products will inherit them.

So, how do we have all developers use Jenkins pipelines without any developers implementing Jenkins pipelines?

## The declarative Jenkins pipeline

The [Jenkins pipeline doc](https://jenkins.io/doc/book/pipeline/) provides a nice overview of Scripted and Declaritive pipelines. I like the declarative version because it is sufficiently robust and guards against people making odd, one-off implementations. The basic form of our pipeline will be:

```
pipeline {
    agent any 
    stages {
        stage('Test') { 
            steps {
                // 
            }
        }
        stage('Package') { 
            steps {
                // 
            }
        }
        stage('Archive') { 
            steps {
                // 
            }
        }
    }
}
```

## A custom Jenkins library

The [Jenkins shared library doc](https://jenkins.io/doc/book/pipeline/shared-libraries/) provides a great primer: organize files in a GitHub repo like this:

```
(root)
+- src                     # Groovy source files
|   +- org
|       +- foo
|           +- Bar.groovy  # for org.foo.Bar class
+- vars
|   +- foo.groovy          # for global 'foo' variable
|   +- foo.txt             # help for 'foo' variable
+- resources               # resource files (external libraries only)
|   +- org
|       +- foo
|           +- bar.json    # static helper data for org.foo.Bar
```

And you can import that library in Jenkins UI configuration, or import it directly into a `Jenkinsfile`.

The really cool part is that we can implement a whole Jenkins pipeline inside a var and have a client `Jenkinsfile` execute it. This provides our single definition of pipeline.

## The developer client experience

One of Jenkin's biggest challenges is managing disparate system and plugin configuration. When we developed the ephemeral Jenkins docker image, we eliminated much of this - most of the configuration is baked into the image. To support building products in any language, we require developers to provide 'CI containers' containing all their favorite build tools - where we will actually build their products. This eliminates more unique configurations. By providing a Jenkins pipeline abstraction, we can hide Jenkins version instability from our developers as well. What should that abstraction look like?

We know that we have three distinct deployment targets. We also know that each environment will use different GitHub branches and may perform different actions. We want to stem this kind of domain leakage and present a simple, coherent abstraction to our developer clients. The metaphor of a function call with a config bundle works nicely.

Jenkins shared libraries allow us to introduce groovy code into the pipeline sandbox and pass it a single closure - a config bundle. This allows us to:

* include our Jenkins shared library functions
* invoke a container build and deploy pipeline
* pass configuration for the build environment, the pipeline, and provide indirection for scripts the client will provide to implement each of the build stages.

Using those ideas, we can have a client developers use a Jenkinsfile in this form:

```groovy
@Library('jenkins-pipe@release-1.0') _

containerPipeline([
    // Configure the environment available in the build pipeline
    environment: [
        DOCKER_CI_IMAGE: 'stevetarver/maven-java-ci:3.5.4-jdk-8-alpine-r0',
    ],
    // Configure the build pipeline
    pipeline: [
        dockerGroup: 'stevetarver',
        slackChannel: 'tarver-build',
        slackCredentialId: 'tarver-build-slack-token',
        test: [
            unitTestResults: 'target/surefire-reports/*.xml',
            codeCoverageHtmlDir: 'target/site/clover'
        ],
        facts: [
            configFile(fileId: 'ms-ref-java-spring-config', variable: 'CONFIG_FILE')
        ],
        secrets: [
            file(credentialsId: 'ms-ref-java-spring-secrets', variable: 'SECRET_FILE')
        ]
    ],
    // Identify scripts to run for each stage of the build pipeline
    stageCommands: [
        build: "mvn -Dspring.profiles.active=dev clean compile",
        test: "mvn -Dspring.profiles.active=dev clean clover:setup test clover:aggregate clover:clover",
        package: "./jenkins/scripts/package.sh",
        deploy: "./jenkins/scripts/deploy.sh",
        integrationTest: "./integration-test/run.sh -t integration -e dev.ops",
        prodDeploy: "./jenkins/scripts/prod_deploy.sh",
        prodTest: "./integration-test/run.sh -t integration -e prod.ops"
    ]
])
```
We provide separate config blocks for

* environment variables the client needs in their build scripts
* pipeline configuration for each of the environments
* client specified scripts to implement the duties and responsibilities of each stage (defined in a previous article)

## Pipeline stage duties and responsibilities

The lower production level (`dev`, `pre-prod`) pipelines provide the following stages:

| Stage            | Executes in  | Description                              |
| ---------------- | ------------ | ---------------------------------------- |
| Build            | CI container | Produce any artifacts required - jars, go exe's, etc. |
| Test             | CI container | Perform all unit tests, export results, fail if appropriate quality gate not met. |
| Package          | Any executor | Perform all steps necessary to build the docker image(s). |
| Archive          | Any executor | Push the docker image to the docker registry - implemented by the pipeline. |
| Deploy           | Any executor | Deploy the project to the Kubernetes cluster via `helm`. |
| Integration Test | CI container | Run integration tests on your deployed service. |

The `prod` environment pipeline provides the following stages:

| Stage           | Executes in  | Description                              |
| --------------- | ------------ | ---------------------------------------- |
| Create Release  | Any executor | Create a new GitHub release, incrementing the `minor` version number from the previous release. |
| Canary Deploy   | Any executor | Deploy the service to the canary location for evaluation. |
| Canary Test     | CI container | Perform a smoke test on the deployed service. This image has already been tested in lower production levels; |
| Canary Rollback | Any executor | Rollback to the previous deployment when canary tests fail. Requires manual approval. |
| Prod Deploy     | Any executor | If canary tests pass, continue the deploy to all desired locations. Requires manual approval. |
| Prod Test       | CI container | After all prod facilities are deployed, verify the deployment sane with lightweight, happy path, newman test that touches all major areas of functionality. |


## The Jenkins shared library implementation

The entry point for our docker image producing pipeline is `var/containerPipeline.groovy`. It will identify its target environment and then validate the args the client has provided are appropriate and sufficient for that environment.

```groovy
    // Execute the pipeline -------------------------------------------------------------
    // env.TARGET_ENV is set per Jenkins deployment to identify the target environment
    if (env.TARGET_ENV == 'prod') {
        containerCdPipeline(config)
    } else {
        containerCiPipeline(config)
    }
```

Since the `dev` and `pre-prod` have the same behavior except the target branch and deployment target, both are lumped into the 'else' block. The `var/containerCiPipeline.groovy` pipeline begins with a `node` preamble to work around some Jenkins declarative pipeline limits:

```groovy
    node {
        checkout scm

        initEnvVars()
        sh "env | sort"

        retryWithDelay {
            withCredentials([usernamePassword(
                    credentialsId: env.dockerJenkinsCreds,
                    passwordVariable: 'DOCKER_REG_PASSWORD',
                    usernameVariable: 'DOCKER_REG_USER')]) {
                sh """
                    echo 'eagerly fetching ci image...'
                    docker login -u ${DOCKER_REG_USER} -p ${DOCKER_REG_PASSWORD} ${env.dockerRegistryUrl}
                    docker pull ${env.DOCKER_CI_IMAGE}
                    docker logout ${env.dockerRegistryUrl}
                """
            }
        }
    }
```
Limits? Specifically, Jenkins does a lightweight checkout and the source repository may be in an unknown state, especially because we let the clients opt out of some stages. Performing the scm checkout prior to the pipeline provides the repo in a known state and lets us initialize environment variables we will use for image tagging. The second problem is horrible connectivity to Docker Hub. Nothing erodes confidence in the build system like build failures that have nothing to do with code problems. Bizarrely, the Jenkins pipeline provides no facilities for retry; the most significant addition we made early on was simple retry logic. Eagerly pre-fetching the CI container allows us to insert that retry logic.

From there, we move on to implementing build stages. 

```groovy
    pipeline {
        agent any
        options { timestamps() }

        stages {
            stage('Build') {
                agent {
                    docker {
                        image DOCKER_CI_IMAGE
                        registryUrl env.dockerRegistryUrl
                        registryCredentialsId env.dockerJenkinsCreds
                        alwaysPull false
                        reuseNode true
                        args dockerCiArgs
                    }
                }
                when { allOf { not { branch 'release' }; expression { config.stageCommands.get 'build'} } }
                steps {
                    echo '===== Build stage begin ============================================================================'
                    sh "rm -f ${gradleLock}"
                    sh "${config.stageCommands.get 'build'}"
                    echo '===== Build stage end   ============================================================================'
                }
            }
        // ...
        }
    }
```
Each stage has a similar implementation: pull the CI container, conditional logic to determine if the stage should be executed, then execute the build script the client provided during initialization. For the build stage above, the conditional says: execute for every branch except release AND if the client provided a 'build' script.

The `var/containerCdPipeline.groovy` follows much of the same form but concerns itself only with the release phase, tackling tasks like creating a more readable docker image tag, creating a GitHub release from commit logs, and the measured production deploy: canary deploy, smoke test, approval, and final deploy.



## Epilog

First, a shout out to my partner [Ralph McNeal](https://www.linkedin.com/in/ralphmcneal/), a co-author of this effort who has seemingly endless patience, a great eye for the design aesthetic, and is a genuine joy to work with.

Next month we'll walk through a local proving ground for this setup - in Minikube.
