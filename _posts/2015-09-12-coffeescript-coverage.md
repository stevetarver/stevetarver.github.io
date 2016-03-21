---
layout: post
title:  "CoffeeScript coverage"
subtitle: "CoffeeScript coverage on client side with all the connective tissue."
date:   2015-09-12
categories: intellij mongo mongodb syntax highlighting
---

<img style="float: left;" src="/images/coffeescript_logo.png">


When you find the right language, build and test system, supporting tools and services, they feel like a favorite T-Shirt, and you want to wear them all the time.

How do you combine things Grunt, CoffeeScript, Browserify, and satisfy engineering concerns like unit tests and code coverage reports, and integrate all of that with CI and coverage services?

## What am I really trying to do?

There are core facilities that are common to projects as diverse as Backbone static SPAs, Express full stack apps, and simple simple npm packages. I want a system that I can use in each of these types of projects, but will focus on the client side today, expecting that most of it will easily translate to the other project types. Specifically, the system:

- Uses CoffeeScript for everything except distributed files
- Runs Mocha/Chai in an html page using PhantomJS
- Generates coverage reports as html for immediate viewing
- Uploads coverage reports to CodeCov.io or Coveralls.io
- Has coverage reports that show annotated CoffeeScript although the tests were run against browserify'd JavaScript.
- Does not generate intermediate JavaScript files
- Only uses Grunt plugins/Browserify transforms - no custom webhooks, shell scripts, etc.
- Uses minimal packages

Why CoffeeScript? I really enjoy the clarity of thought and intention it provides by eliminating the needless clutter in JavaScript syntax. I like it everywhere; from the Gruntfile, to the implementation, to the specs.

Why Mocha/Chai? It works well on both client and server side, has a pleasantly readable format, and generates nice reports.

Why Grunt? Grunt, Gulp, whatever? I've used both and not found a compelling reason to prefer either.

## Where to start?

I've written an example implementation [here](https://github.com/stevetarver/example-grunt-browserify-coffee-coverage) that I will be pulling code snippets from. Clone it, follow the README and check out the facilities it provides. This article follows the README pretty closely - you could skip this article and just read the README.

Developing the project went pretty much as follows

- Create the testing infrastructure: client/specs/public contains a standard PhantomJS/Mocha test page that includes Mocha/Chai, the browserify'd test bundle, mocha styling, and a call to mocha.run()
- Create specifications in client/spec/specs and a single file client/spec/spec-main.coffee that includes all specs primarily so you can order the specs to make the output more readable.
- Create functionality in client/scripts
- Create a Gruntfile that will
  - Copy spec resources to the build directory
  - Browserify the specs and include coverage
  - Run Mocha in the html page under PhantomJS
  - Generate coverage reports
- Integrate with Travis, CodeCov.io, Coveralls.io, David-dm and use shields.io to generate badges for each.

## The Tricks

The key elements are...


### grunt-mocha-phantom-istanbul

This Grunt task is a grunt-mocha fork with a small but powerful addition: the ability to extract Istanbul coverage information and write standard Istanbul reports.

This Grunt task configuration runs the mocha tests in build/index.html and writes the lcov report to the coverage directory. This provides html for immediate code improvement and lcov.info to upload to CodeCov.io or Coveralls.io

```coffeescript
mocha:
  test:
    src: "build/index.html"
    options:
      run: true
      reporter: 'Spec'
      log: true
      logErrors: true
      coverage:
        lcovReport: 'coverage/'
```

It does not instrument the code so we will do that with...


### browserify-coffee-coverage

This Browserify transform wraps the most excelent coffee-coverage package.

Digging through the package source and test reveals two options we need: instrumentor and ignore.

The instrumentor option lets us choose Istanbul as opposed to JSCoverage. The ignore option lets us exclude the test files from the coverage report.


### grunt-browserify

This Grunt task provides robust Browserify functionality in Grunt.

The Browserify transform pipeline is normally given as an array of strings.

Many transforms will take arguments and to accomodate this, grunt-browserify has a special syntax of:

```coffeescript
[transformName, {arg1: 'value1', arg2: 'value2'}]
```

Elements of the transform pipeline array may be either transform name, or the transform with args syntax above. More concretely:

```coffeescript
browserify:
  test:
    src:  "#{spec}/spec-main.coffee"
    dest: "#{testbuild}/js/spec-main.js"
    options:
      transform: [
        ['browserify-coffee-coverage', {instrumentor: 'istanbul', ignore: '**/spec/**'}],
        'jadify'
      ]
      debug: true
```

## Integrating with Travis, CodeCov, Coveralls, and David-dm

It's hard to find quality code, so I love the badges that README.md files wear showing build status, code coverage, and dependency up-to-datedness. Now that our project has test coverage, let's hook that up to CI and coverage services so we can get some badges.


### Create accounts

Visit each of

- https://travis-ci.org
- https://codecov.io
- https://coveralls.io

to create an account, link it to your GitHub account, and enable repos - it's free for open source projects. After each has sync'd to your GitHub account, you will be able to enable projects, and thereafter, Travis will build on every push if a .travis.yml exists, and our configuration will push to CodeCov.io and Coverall.io.

I include configuration for both CodeCov.io and Coveralls.io in case you have a preference - I haven't really figured out which I like better yet.

CodeCov.io has a pretty cool Chrome, FireFox, Opera extension that overlays coverage information on several GitHub code views and works perfectly with the CoffeeScript coverage we have created. Check out an overview at YouTube and install from https://github.com/codecov/browser-extension#codecov-browser-extension. Safari and IE are planned additions.


### Configure .travis.yml

Your basic .travis.yml looks like

```yaml
language: node_js
node_js:
  - '0.12'
before_install:
  - 'npm install -g coffee-script'
  - 'npm install -g grunt-cli'
after_success:
  - 'npm run-script codecov'
  - 'npm run-script coveralls'
```

Before Travis installs our package and builds it, we want grunt-cli installed globally so we can use it to run our build.

After our build completes successfully, we want to upload the coverage information to the coverage services. We will be using npm packages codecov.io and coveralls respectively.

After those packages are installed, we can create scripts in our package.json to upload coverage.

```json
"scripts": {
  "test": "grunt",
  "codecov": "cat ./coverage/lcov.info | ./node_modules/.bin/codecov",
  "coveralls": "cat ./coverage/lcov.info | ./node_modules/.bin/coveralls"
},
```

After these changes are made, our next push will trigger a build and upload to the coverage services. You can visit each site to see how the build went and examine their offerings.


### David

david-dm is another great service that tracks your npm dependencies to show when they become out of date.

Clicking on the badge will take you to a david-dm page that lists all of your dependencies, your required version along with the stable and latest versions of those dependencies.


### shields.io

shields.io provides badges for each of these services, and many more.

Most services will provide links to badges for your project for that service, but sometimes those links are hard to find. Check out the badges at the top of the project README.md for a hint at how to decorate your page.

**NOTE**: I included the npm version badge as an example. Since I have not published this package to npm, it shows as invalid.

## Epilog

There you have it - 3 well chosen packages, a couple of Gruntfile configurations yielding a pattern I will use over and over again.

Things to look at (after project clone, install, and build)
The console after a build: shows well formatted specification results and coverage summary.
build/index.html: Mocha prints to the console, but also has a sweet test results presentation.
coverage/lcov-report/index.html: your coverage results with annotated CoffeeScript available through drill down.
