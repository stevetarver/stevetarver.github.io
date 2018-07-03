---
layout: post
title:  "Pipeline Design Pt 3: Interactions with GitHub"
date:   2017-10-11 11:11:11
tags:   ci cd pipeline jenkins automation

---

**TL;DR** Define a git workflow for your development team, targeting our WiP build pipeline

<img style="float: left;" src="/images/logo/github.png">

Through this iterative design, we are revealing more concerns, and we need to switch from a top-down design to a bottom-up method to ensure all needs are being met. So far, we know that we have three isolated environments (`dev`, `pre-prod`, and `production`) that will align with well-known branches (`master`, `candidate`, and `release`). This allows daily work in master to be isolated from 'march to production' work in the release candidate branch. It also provides a dedicated workspace for production code; the `release` branch should always contain the code deployed to production.

With a slow march to production, we also need a place to work on production bugs - we don't want to interrupt daily work in `master`, don't want to derail a current march to production in `candidate`, and we don't want to pollute the `release` branch with intermittent work. We'll introduce a completely new branch, `hotfix`, for just this purpose.

Let's walk through how developers use these branches daily, and some use cases for our expected work.


## Production levels, environments, and branches

All products will enter our Kubernetes clusters through our build pipeline.

Our three production levels (`dev`, `pre-prod`, `prod`) provide their slightly different semantics in corresponding Kubernetes clusters:

* `dev`: continuous build and deployment of incremental feature branch merges to `master` branch
* `pre-prod`: continuous build, deployment, and proving for release `candidate` and `hotfix` branches
* `prod`: docker image deployment of customer facing services (images) mirrored by the `release` branch

At a high level, feature branches are merged into `master` and automatically built and deployed to `dev`. When enough features are accumulated, and are at sufficient quality, `master` is merged into the release `candidate` branch which is automatically built and deployed on the `pre-prod` cluster.  When the release candidate image is deemed suitable quality, the `candidate` branch is pushed to the `release` branch to reflect currently deployed code, a GitHub release is created, the proven release candidate image is re-tagged with the GitHub release version, and that image is deployed in `prod`.

Let's look a little deeper at GitHub branches and transitions.


## GitHub branches

Commits to standard GitHub branches provide triggers for execution of different phases of the Jenkins pipeline. 

| Branch      | Content             | Target       | Description                              |
| ----------- | ------------------- | ------------ | ---------------------------------------- |
| `master`    | next-up features    | `dev` | A merge into `master`, from a feature branch, a PR, or direct commit triggers a build and deploy to `dev`. |
| `candidate` | release candidates  | `pre-prod` | When current work in `master` is ready for promotion, `master` is merged into the `candidate` branch which triggers a build and deploy to `pre-prod` |
| `release`   | released code       | `prod`     | When a release is ready for production, `candidate` is merged into `release` and a user manually triggers the job which applies final image versioning, a GitHub tag and release, and deployment to production. |
| `hotfix`    | production hotfixes | `pre-prod` | When a production fault requires an immediate fix, a short-lived `hotfix` branch is created from the current `release` tag triggering a build and deploy to `pre-prod`. When the hotfix is of suitable quality, it is merged into `release` and the `hotfix` branch is deleted. |
| `*`         | all other branches  | none         | When any other branch is committed, the CI portion of pipeline (build, test) is run in the dev environment. For PRs, the results are attached to the PR. |

## Pipeline stages

The pre-release (`dev`, `pre-prod`) and release (`prod`) pipelines address different concerns. Each pipeline is implemented in a common Jenkins library which each product invokes through a common project root level `Jenkinsfile`.

### Pre-release pipeline stages

The pre-release pipeline focuses on building, testing, and creating docker images. The canonical pre-release pipeline provides the following stages:

| Stage            | Description                              |
| ---------------- | ---------------------------------------- |
| Build            | Produce and stash build artifacts for packaging: jars, binaries, etc. |
| Test             | Unit test the product and produce unit test and coverage reports. |
| Package          | Produce the docker image.                |
| Archive          | Push the docker image to the portr docker registry. |
| Deploy           | Create the Helm chart, push to Tiller, schedule a cluster deployment. |
| Integration Test | Test the image in the target cluster.    |

### Release pipeline stages

The release pipeline focuses on image selection (candidate, hotfix, redeploy) and a measured deploy to production. It is always manually initiated. The canonical release pipeline provides the following stages:

| Stage           | Description                              |
| --------------- | ---------------------------------------- |
| Create Release  | Create a new [semver](http://semver.org/) version, pull the source image, retag with bare repo name and new version, create a GitHub release. |
| Canary Deploy   | Create the Helm chart, push to Tiller, schedule a canary datacenter deployment. |
| Canary Test     | Run smoke tests in the canary datacenter. Prompt the user to continue the deploy or rollback. |
| Canary Rollback | Rollback the canary deployment to the previous version. |
| Prod Deploy     | Create the Helm chart, push to Tiller, schedule a full cluster deployment. |
| Prod Test       | Run smoke tests on all prod locations.   |

**NOTE**: A production rollback is trivially implmented as a re-deploy of the previous version.

## Continuous integration, delivery, and deployment flow

Tying the above ideas together, we arrive at a combined integration, delivery and deployment flow:

<img style="float: center;" src="/images/custom-cicdcd/pipeline-by-env.png">

* The `master` branch is processed in the `dev` pipeline and deploys to the `dev` environment.
* The `candidate` branch is processed in the `release candidate` pipeline and deploys to `pre-prod`.
* The `release` branch is processed by the `release` pipeline and deploys to `prod`. Note that no new docker image is built here; the `release` branch serves two purposes: provides a Jenkins job target and always reflects the code that is in production.
* The `hotfix` branch is processed by the `release candidate` pipeline and deploys to `pre-prod` (not shown).

## Git branch transitions

### Normal flow

The Jenkins pipeline responds to git merges and commits. At a high level, the normal feature flow through git looks like:

<img style="float: center;" src="/images/custom-cicdcd/git-overview-normal.png">

### Hotfix flow

A hotfix release is an emergency production code change. It adds a branch to our normal flow above to isolate that work, but the release pipeline steps are nearly identical.

<img style="float: center;" src="/images/custom-cicdcd/git-overview-hotfix.png">

## A feature's normal path to production

Let's walk through developer life living with this pipeline:

1. All work begins with a tracking card (Trello, Jira, post-it), whether external feature request, tech debt, or maintenance.
2. Developers work in "feature" branches with a name that identifies the tracking card and a short name: "M1-add-api-validation".
3. When feature work is complete, a PR is submitted against that branch, and when the PR process is complete, that feature branch is merged into the `master` branch.
4. The `master` merge triggers Jenkins to execute all stages in the `development` pipeline phase resulting in a deployment to our dev environment.
5. As issues are identified, they are resolved in commits to `master` triggering more executions of the `development` pipeline phase and deployment to the dev environment.
6. When the `master` code is of suitable quality - a release candidate - the `master` branch is merged into the `candidate` branch.
7. The `candidate` merge triggers Jenkins to execute all stages in the `release candidate` pipeline resulting in a new docker image deployed to our pre-production environment.
8. As issues are identified, they are resolved in commits to `candidate` triggering more executions of the `release candidate` pipeline phase and deployment to the ppe environment. Fixes are also merged down into `master`.
9. When the `candidate` code is of suitable quality, and ready to push to production, the `candidate` branch is merged into `release`.
10. A production release is always initiated manually, the user selects the candidate branch and Jenkins executes all stages in the `release` pipeline phase including docker image re-tag, GitHub tagging, GitHub release, and deployment to the production environment.

The git branch transitions look like:

<img style="float: center;" src="/images/custom-cicdcd/git-branch-normal.png">

1. Feature work is completed in the `feature-1` branch.
2. `feature-1` is merged into `master` triggering a build and deploy to `dev`.
3. A bug is fixed in `master` and committed triggering another build and deploy to `dev`.
4. `master` is merged into `candidate` triggering a build and deploy to `pre-prod`.
5. A bug is fixed in `candidate` and committed triggering another build and deploy to `pre-prod`.
6. `candidate` is merged into `release` triggering final grooming and a deploy to `prod`
7. The bug discovered in `pre-prod` is merged down into `master`.

## Other use cases

### Pull Requests

When non-project contributors submit work via pull request, the `pull request` section of the pipeline is executed. This abbreviated flow includes the Test stage producing unit test and code coverage results which are added as comments to the pull request. When the PR is found of suitable quality, it is merged into the `master` branch and becomes part of the normal flow to production.

**NOTE**: The Jenkins Multi-branch pipeline plugin automatically discovers all branches in a GitHub repository and executes this flow.

Notice that the git branch diagram for a PR is identical to a feature branch:

<img style="float: center;" src="/images/custom-cicdcd/git-branch-pr.png">


### Hotfixes

When a flaw is identified in production that requires immediate remediation:

1. The currently deployed `release` tag is branched into `hotfix`.
2. The `hotfix` commit triggers Jenkins to execute all stages in the `release candidate` pipeline phase resulting in a new docker image deployed to our pre-production environment.
3. If a release candidate is already in-flight, you should consider whether the hotfix should be part of the next release candidate OR pause work on the `candidate` branch to separate the work streams.
4. As issues are identified, they are resolved in commits to `hotfix` triggering more executions of the `release candidate` pipeline phase and deployment to the ppe environment.
5. When the `hotfix` code is of suitable quality, and ready to push to production, the `hotfix` branch is merged into `release`.
6. The normal `release` stages are executed.

The git branch transitions look like:

<img style="float: center;" src="/images/custom-cicdcd/git-branch-hotfix.png">

1. A flaw is identified in production that requires immediate remediation.
2. A `hotfix` branch is created from the current `release` branch.
3. The production flaw is fixed in `hotfix`.
4. `hotfix` is merged into `release` triggering a deploy to production.
5. The fix in `hotfix` is merged down into `candidate`.
6. The fix in `hotfix` is merged down into `master`.

## Notes on naming

Products use their GitHub repo name as both Jenkins job name and docker image name because we prize operational over semantic clarity; it is more important for an operator to be able to locate source for a build job or deployed artifact than the name of that artifact making perfect semantic sense.

We use docker images as our fundamental deployment unit and store those under a single repository group. in a private registry requiring each image to use a fully qualified docker image name. For this effort, we will store images in Docker Hub under the `stevetarver` group which yields image names like:

```
stevetarver/ms-ref-python-falcon
```

Pre-release builds append the branch name to the repo name and use a build timestamp as a tag to uniquely identify the image and indicate its provenance. 

```
stevetarver/ms-ref-python-falcon-master:20170817191210
stevetarver/ms-ref-python-falcon-candidate:20170817191211
stevetarver/ms-ref-python-falcon-hotfix:20170817191212
```

When a candidate or hotfix is deemed ready for production, a new version is created to represent the feature group or fix being deployed. New features increment the minor version and fixes increment the micro version.

The latest branch image is pulled and retagged to omit the branch name and use the new version.

```
stevetarver/ms-ref-python-falcon:1.1.0
```

## Epilog

Now we know how developers will interact with GitHub and how GitHub will trigger Jenkins builds and deploys. Our next step will be to create a generic Jenkins pipeline that can implement this pipeline and be flexible to work with all of our client projects.


