# Choosing Ember


My UI is having growing pains. It has reached the point where the architecture is restraining us instead of serving us; it is slowly eroding our business agility and affecting UX.

The UI is jsp with some client-side javascript; more than just javascript sprinkles, but no dedicated SPA component. This made a lot of sense when the UI was created: the team has a lot of Java expertise so frameworks, building, packaging, deploying, and hosting were very familiar. The UI code was very simple so starting with a server-side app really helped get the product to market quickly. But now it is time to pay down that tech debt balloon that always accumulates around a product launch.

**TL;DR**: How I chose a UI framework and lessons learned from a first Ember.js 2.x app.

## What problems am I trying to solve?

- **UX**: Worst page loads take 11 seconds, and because of external backend dependencies, can take much longer.
- **Rapid development**: The code, test, debug loop is 30 seconds. Each trivial UI change I make, I have to redeploy to see in the browser so small additions end up taking a day.
- **Testing**: Client-side code is now significant and has zero testing. Unit testing would allow TDD/BDD and catch incremental development errors improving feature delivery time.
- **Separation of concerns**: We hava a kind of MVVM with server-side components and the separation of concerns has become a bit blurred. The code has evolved so that some times there are models, sometimes in-page scripts to take advantage of server-side variable substitution. A more opinionated and rigid project structure would clarify duties and responsibilities and simplify maintenance.
- **Fearless refactoring**: Refactoring is tedious and risky because of the above reasons. This has to be easy so that developers will refactor as part of normal development and keep the code base sane.

## Server-side or client side?

Server-side rendering is great for static or content heavy apps. We chose a server-side base because of team experience and the initial UI was simple enough to not require modern Node.js capabilities. But this UI is really a dashboard and I expect the functionality, and complexity, to really increase this year. 

We really want desktop application like responsiveness and sophistication so this moves us away from server generated pages. Dashboards scream SPAAAAAA! 

I expect to need a SPA component and a matching lightweight API that functions as a router to our published API, other internal APIs, and a database.

## Choosing a framework

I have some experience with [jQuery](https://jquery.com/), [Knockout.js](http://knockoutjs.com/), [Backbone.js](http://backbonejs.org/), some dabbling in [AngularJS](https://angularjs.org/), and have heard authors talk about [React](https://facebook.github.io/react/) and [Ember](http://emberjs.com/). Not an exhaustive list, but captures the major players and seems like a good candidate list.

This decision will obviously not be made on deep experience, but I am not sure that deep experience helps these kinds of decisions. When you have deep experience with a framework, you have likely become nose-blind - you don't notice the smells anymore and prefer the familiar. Instead, I will try to match frameworks against the problems I am trying to solve, implement a non-production app with that framework, and answer at the end: "Am I on the right track?"

**jQuery** is a building block or expected addition to many frameworks. You could build an amazing app with just jQuery, but I want the framework to do most of that code.

**Knockout.js** provides data binding and associated linking and dependency tracking as well as some basic templating. We are using Knockout.js today as data binding for [Cyclops](http://assets.ctl.io/cyclops/1.2.0/), our UX/UI pattern implementation framework. Any solution will have to allow Knockout.js integration, but on its own, Knockout.js just does not provide enough functionality.

**Backbone.js** is the simplest of the "full featured" frameworks built by Jeremy Ashkenas of [CoffeeScript](http://coffeescript.org/) fame (or infamy), and [Underscore.js](http://underscorejs.org/). Backbone.js is an un-opinionated framework providing basic MVVM separation of concerns and two way data-binding. It's strength is that it provides the basics that you can build anything on top of. I have had some pretty good experiences here, but think a team can benefit from more structure. A more opinionated approach, that matches my opinions, can eliminate a lot of boilerplate code and keep a team implementing the same things in the same way.

**AngularJS** is the most established brand and probably gets a lot of buy-in because it is from Google. My first experience with Angular 1.x was not pleasant. It felt heavy-handed, awkward, and just didn't make a lot of sense. There are several ways of doing things and if you pick the wrong strategy, shifting to the right strategy is a significant refactor. I also haven't heard a lot of good things about Angular 2. I really don't like making decisions based on the scents others put in the air, but frameworks should bring joy to the developer and this one didn't for me.

**React** Facebook's UI framework quickly built a reputation for simple, quick, robust application development. If I hadn't heard Yehuda Katz talk about incorporating the best ideas from React into Ember.js 2.x, I probably would have gone with React.

### And the winner is:

**Ember.js** 

Tom Dale has a [SproutCore](http://sproutcore.com/) heritage - my first really good experience with a front end framework. Yehuda Katz continually impresses me with his clarity of thought, knowledge of software engineering theory, clear vision, and his ability to clearly communicate justifications for design approaches and decisions. I love his fundamental tenet that languages and frameworks should bring joy to developers (inspired by DHH and Matz of course).

Together, these two provide a clear vision of what they think a UI framework should be in Ember.js. Version 2 brings a lot of maturity and stability to the framework; they are past the point of innovating, and have provided the first LTS version: 2.4.

_Is basing a framework choice on author personality wrong?_ I think about this much as I do hiring. I used to put candidates through the wringer in interviews. This is fundamentally flawed for, just, so many reasons and does not get me closer to selecting a good candidate. Today I focus on things like: 

- would I like pairing with the candidate - every day
- do they have mature opinions about software engineering
- do they prioritize engineering concerns as I do
- is there a body of work (code/thoughts) I can compare to what is said in the interview
- will they learn, on their own, everything needed to be successful in this new environment
- will they be happy here

If these things are a match, I want to try them out for a while to prove that match, then they become part of the team.

Those same concerns map pretty cleanly to how I think about framework choices. I don't have years of experience with these frameworks and don't have the time to gain that experience. I don't care about minimal performance gains or underlying implementation details. I care about how I feel when I use the framework, that it provides for my engineering concerns, and that working with it feels natural.

So, if I listen to the authors speak, get familiar with their body of work, and it's a match, I feel a lot more comfortable spending the time to evaluate their framework.

## The first project

Ember.js provides good [guides](https://guides.emberjs.com/v2.4.0/) and [tutorials](http://emberwatch.com/tutorials.html). There is a lot out there to help you with your first project, just ensure it is Ember.js 2.4; a lot has changed since v1.x.

I won't be trying to provide a tutorial, there are already so many good ones out there. I will just be standing up a simple app and noting my reactions.

## Creating the skeleton

The [ember-cli](http://ember-cli.com/) is responsible for a lot of the prototyping speed and is core to using Ember - gotta install that first.

```
npm install -g ember-cli
```

Now I can create my first app

```
ember new admin-ui
```

This one command does A LOT! 

- generate a bowerrc and manifest and downloads dependencies
- generate the node package.json and downloads dependencies
- initializes a git repo
- provides a travis.yml
- builds out the basic application structure
- builds out the basic test framework structure

During the build, I noticed this warning

```
Could not start watchman; falling back to NodeWatcher for file system events.
Visit http://www.ember-cli.com/user-guide/#watchman for more info.
```

If you have built other SPA's, you have probably noticed some NodeWatcher problems monitoring your project files for changes. Even in my small projects, getting all file changes to register and trigger a page refresh was error prone and tedious. 

Facebook's Watchman directly addresses this problem. Even if you are not at Facebook scale, eliminating disruptions to coding flow makes you happier and more productive.

**NOTE**: This is not the npm package `watchman` - don't install that.

```
brew install watchman
```

After I create a new project, my next step is to commit the skeleton to git so I can roll back if I really screw something up. When I opened the project in WebStorm, the git log shows no uncommitted files. ??? During project generation, the `.gitignore` was created and all files committed by Tomster. Thanks for the assist Tom.

Let's run it and see what happens.

```
$ ember server
version: 2.4.2
Livereload server on http://localhost:49154
Serving on http://localhost:4200/

Build successful - 7751ms.

Slowest Trees                                 | Total               
----------------------------------------------+---------------------
Babel                                         | 3215ms              
Babel                                         | 1880ms              
Babel                                         | 479ms               

Slowest Trees (cumulative)                    | Total (avg)         
----------------------------------------------+---------------------
Babel (12)                                    | 6509ms (542 ms)     
```

OK. Livereload is already setup and build response times are logged. The second app start dropped build time to 2s so we must not be rebuilding everything.

The skeleton page simply shows _Welcome to Ember_. If I modify `app/templates/application.hbs` to _Welcome home Steve_, let the editor window lose focus, the rebuild is 424ms and page re-load happens as expected. Already off to a good start.


## Testing the skeleton

I noticed a `tests` directory and a `/testem.js` - let's try that out.

```
$ ember test
version: 2.4.2
Built project successfully. Stored in "/Users/starver/code/admin-ui/tmp/class-tests_dist-8Wl0qCKV.tmp".
not ok 1 Error
    ---
        message: >
            Launcher PhantomJS not found. Not installed?
```

That's pretty helpful.

```
npm install -g phantomjs
```

```
$ ember test
version: 2.4.2
Built project successfully. Stored in "/Users/starver/code/rdbs/ds-admin-ui/tmp/class-tests_dist-98GG4OAa.tmp".
ok 1 PhantomJS 2.1 - JSHint - app.js: should pass jshint
ok 2 PhantomJS 2.1 - JSHint - helpers/destroy-app.js: should pass jshint
ok 3 PhantomJS 2.1 - JSHint - helpers/module-for-acceptance.js: should pass jshint
ok 4 PhantomJS 2.1 - JSHint - helpers/resolver.js: should pass jshint
ok 5 PhantomJS 2.1 - JSHint - helpers/start-app.js: should pass jshint
ok 6 PhantomJS 2.1 - JSHint - resolver.js: should pass jshint
ok 7 PhantomJS 2.1 - JSHint - router.js: should pass jshint
ok 8 PhantomJS 2.1 - JSHint - test-helper.js: should pass jshint

1..8
# tests 8
# pass  8
# skip  0
# fail  0

# ok
```

I like that Unit, Integration, and Acceptance tests are first class citizens and generated automatically when you use `ember-cli` to generate new components. When doing the TDD/BDD dance, you can `ember test --server` and retest on every file change. 

Not a big fan of QUnit though. I see that I can change to Jasmine or Mocha through Ember AddOns, so that is good news. 

No code coverage? Not configured with the default skeleton but I see that there are several addons that provide Istanbul or Blanket coverage. Something to investigate... later.


## Adding CoffeeScript

Right now, I really want to start coding. I see that Babel is installed, so I can use all the improvements in ES6, but I still prefer CoffeeScript. I just like the way it looks. Probably for the same reasons I like Ruby, Jade, Stylus, and even Groovy.

First, I'll update CoffeeScript since last September they added ES6 generators and ES6-style destructuring defaults. Then add the 
```
$ npm install -g coffee-script
$ ember install ember-cli-coffeescript
```

Now, all the code that ember-cli generates will be in CoffeeScript, my code can be in CoffeeScript, and all transpiling will be handled transparently.

Because Yehuda is working on TC39, specifically on modules, there is strong ES module support in Ember. CoffeeScript doesn't grok ES2015 modules yet, but there is a work-around: backtick the import and export statements so they run as JavaScript. This is done for you by `ember-cli-coffeescript`, but when adding your own imports, just follow the example in the code.


## The first page

What's it like to add a page? Zoltan as a pretty nice [tutorial](http://yoember.com/) that I am working through - I'll use those examples directly.

First add Bootstrap and a little easier style handling

```
$ ember install ember-cli-sass
$ ember install ember-cli-bootstrap-sassy
$ mv app/styles/app.css app/styles/app.scss
```

Simply importing Bootstrap into our `app.scss` file provides let's us start using Bootstrap styling.

```
@import "bootstrap";

body {
  padding-top: 20px;
}
```

Now we can replace `app/templates/application.hbs` content with

```
<div class="container">
  {{partial 'navbar'}}
  {{outlet}}
</div>
```

The double moustache, `{{`, indicates a [Handlebars](http://handlebarsjs.com/) substitution. In this case, we have a partial for the navbar while the main page content will be dumped into `{{outlet}}`. All Ember templates are based on Handlebars.

ember-cli will generate the navbar template for us

```
ember generate template navbar
```

And we'll use Zoltan's navbar for now

```
<nav class="navbar navbar-inverse">
  <div class="container-fluid">
    <div class="navbar-header">
      <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#main-navbar">
        <span class="sr-only">Toggle navigation</span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </button>
      {{#link-to 'index' class="navbar-brand"}}Library App{{/link-to}}
    </div>

    <div class="collapse navbar-collapse" id="main-navbar">
      <ul class="nav navbar-nav">
            {{#link-to 'index' tagName="li"}}<a href="">Home</a>{{/link-to}}
      </ul>
    </div><!-- /.navbar-collapse -->
  </div><!-- /.container-fluid -->
</nav>
```
We can run the app now and see how that turned out.

## The first route

On the server side, routes map a resource to a request handler. Each URI must be sufficiently distinct to identify a single request handler, may contain variables like a userId to select a user from the users collection, and can have some request variables to clarify the request. 

Yehuda says "Don't break the internet", meaning that a SPA's URI should follow many of the same concepts as a server side request.

URIs in SPAs have always been a pain point. You have a single URL and usually just the `location.hash` to tell you what page you are on.

- How do bookmark it?
- What happens when you reload it?
- How do you create links to different views?
- How do you redirect to different views?
- How do you tell what page is shown?
- How do you tell what objects are active?

Ember realized that the SPA benefits from having a single source of state and that the URI is uniquely qualified to suit this need. This requires re-thinking our notions of duties and responsibilities a bit, but when you recognize the clarity gained, even as the project scales, it is easy to leave the past behind.

In Ember, routes can render templates, load models, redirect based on permissions, and have actions that can change models or switch routes.

We can add an About page with:

```
$ ember generate route about
```

Ember generated the `app/routes/about.coffee` and the `app/templates/about.hbs` and then had a problem. We added the CoffeeScript cli add-on and it wants a `router.coffee`. Since the project was generated in javascript, we have a `router.js`. Let's undo what we did, convert `router.js` to `router.coffee`, and try again.

Ember has a destroy method that will remove all the pieces created by a generate.

```
$ ember destroy route about
``` 
Now convert our router javascript to CoffeeScript

```javascript
import Ember from 'ember';
import config from './config/environment';

const Router = Ember.Router.extend({
  location: config.locationType
});

Router.map(function() {
});

export default Router;
```

becomes

```coffeescript
`import Ember from 'ember';`
`import config from './config/environment';`

Router = Ember.Router.extend
  location: config.locationType

Router.map ->
  @route 'about'

`export default Router;`
```
Re-generating:

```
$ ember generate route about
version: 2.4.2
installing route
  create app/routes/about.coffee
  create app/templates/about.hbs
updating router
  add route about
installing route-test
  create tests/unit/routes/about-test.coffee
```
Ember generated our About route (which we could add models, templates, and actions to), our About template, added our About route to the router, and generated a test stub for us.

**Note**: you must add these files to git manually.

The `templates/about.hbs` contains one line: `{{outlet}}`. Let's add some text and see how it looks.

```
<h1>All about this app</h1>
{{outlet}}
```
Browse `http://localhost:4200/about` to check it out.

All good there, but no way to get to that page from the app. If we open `app/templates/navbar.hbs`, we can add 

```
{{#link-to 'about' tagName="li"}}<a href="">About</a>{{/link-to}}
```
and now click between Home and About. This is all pretty cool, but what did the Handlebars substitution really generate? There is nothing in the page source, but the Chrome inspector shows:

```html
<li id="ember396" class="ember-view active"><a href="">About</a></li>
  <a href>About</a>
</li>
```

In the handlebars `#link-to` tag, we only specified 'about', but during route generation, we saw that Ember generated a route, a template, and modified the main router.

Ember relies on strong naming conventions to tie all the pieces together. Following these conventions allows Ember to know that the URI should be `http://localhost:4200/about`, that URI is linked by the router's `@route 'about'` to `routes/about.coffee`, and that route knows to render `templates/about.hbs` without explicitly stating so. This eliminates a lot of boilerplate linking these pieces together - a common pain point and something we can easily leave behind.


## Blueprints

If you look at `ember --help`, you will see entries like

```
ember generate <blueprint> <options...>
  Generates new code from blueprints.
```

Blueprints are code snippets that define common Ember patterns.

`ember help generate` shows a list of all blueprints currently available. You can add community provided blueprints, install blueprints locally and modify them, as well as create your own.

## Components

Components represent a small section of a page, have a template, and can have a model backed by the store. This is your classic `partial` and is frequently used with repeating elements on a page. If you had a blog, and that blog listed all posts in summary form, that page might contain something like

```
{{#each model as |post|}}
  {{#blog-post title=post.title}}
    {{post.body}}
  {{/blog-post}}
{{/each}}
```

More on components [here](https://guides.emberjs.com/v2.4.0/components/defining-a-component/)

## Controllers

Controllers provide a lot of functionality, but the unique contributions are that they:

- provide state based on the current route
- route user actions from a component to a route

Although you can put some Component like behavior in a Controller, you should not. In the future, Components will subsume all Controller functionality and Controllers will go away. Using Controllers for only the above functionality will make that transition easier.

In the About page example above, we used only a route that knew to render its template. If we have a page that the user can interact with, perhaps edit or toggle visibility, we need a controller. Let's see how that works.

Following Zoltan's [example](http://yoember.com/), we can create a Jumbotron with a 

## Routes

## Models

## DV vs PD

## Debugging

If you open the Chrome Developer tools and take a look at the console, you will see something like:

```
DEBUG: -------------------------------
ember.debug.js:6394DEBUG: Ember      : 2.4.3
ember.debug.js:6394DEBUG: Ember Data : 2.4.3
ember.debug.js:6394DEBUG: jQuery     : 2.2.2
ember.debug.js:6394DEBUG: -------------------------------
ember.debug.js:6394DEBUG: For more advanced debugging, install the Ember Inspector from https://chrome.google.com/webstore/detail/ember-inspector/bmdblncegkenkacieihfhpjfppoconhi
```
Ember provides debug output to the console; you can turn it on in `config/environment.js`

```
  if (environment === 'development') {
    // ENV.APP.LOG_RESOLVER = true;
    // ENV.APP.LOG_ACTIVE_GENERATION = true;
    // ENV.APP.LOG_TRANSITIONS = true;
    // ENV.APP.LOG_TRANSITIONS_INTERNAL = true;
    // ENV.APP.LOG_VIEW_LOOKUPS = true;
  }
```
You can uncomment any of these lines to get a deeper view into what Ember is doing behind the scenes.

Also, if you are using Ember features in a deprecated way, you will see console warnings of that deprecation and a suggested fix to your code.

The Ember Inspector is a Chrome extension that detects if an Ember app is running and let you see things like all routes, their associated pieces, interact with objects in your application, inspect your data, and overlay the html page with details about the pieces that implement it. You can see more details following the [console link](https://chrome.google.com/webstore/detail/ember-inspector/bmdblncegkenkacieihfhpjfppoconhi).

This could be Ember's coolest feature. It really levels up live debugging and is possible only because Ember is so opinionated; the inspector has a set of invariant conventions it can use to tie all the pieces together.

### Debugging Breakpoints

In addition to turning on tracing for Ember core components, you can insert the `debugger` command and break in the browser code. Many things have lifecycles; for example, a route goes through init, beforeModel, model, afterModel, activate, setupController, and renderTemplate. To watch setupController work, you can insert the following in your route

```js
  setupController(controller, model) {
    debugger;
  }
```
and break in Chrome to inspect the state of your route, other components, or your app.

## Benefits of opinionated frameworks

Many frameworks provide many ways to skin the proverbial cat. Each team deals with this by adopting conventions arrived at by past pains and enforce them through peer pressure and code review. Hopefully, the team has felt enough pain in the past that they can anticipate the needs of the application and choose the right initial solution or a painful refactor is on the horizon.

Opinionated frameworks offer few, or just one, way to do things. The success of that framework depends on the ability of that opinionated strategy to scale and those opinions to be adopted.

Opinionated frameworks keep a team on track, with everything fitting into the opinions of the framework.

Collaborating remote teams have different cultures, different experiences, and different approaches to solving problems. Opinionated frameworks provide a single solution to problems and simplify collaborating on projects, or assuming responsiblity for those projects.

Larger companies frequently have more work than FTEs can complete and augment their staff with contractors. They slice off pieces of work to dish out to the contractors, the work is done, and the company meets their immediate goals. The pain comes when contractor work needs to be extended and maintained. FTEs with a decidedly different culture and approach must mine the code and wonder "What was that guy thinking?" That frustrating and slow maintenance is something an opinionated framework solves. Once you know that framework, you know the one way that things are done, and anyone should be able to walk into a project based on that framework and understand it.


I have dealt with swiss army knife approaches and much prefer the pain of having to figure out the opinions of a framework once and applying them everywhere to having to understand all the features of a framework and having to figure out how each project decided to piece them together.

## Release


## Dockerizing


## Ember rules

Here are some rules to keep you on track

- Every project is generated with `ember-cli`.
- Every project file is generated with `ember-cli`. 
- All additional functionality is provided by Ember add-ons and added with `ember-cli`. If you need functionality that is not provided in an add-on, consider writing your new functionality as an add-on and contribute back to the community. It shouldn't be much more work than adding that functionality in an ad-hoc way.
- blueprints
