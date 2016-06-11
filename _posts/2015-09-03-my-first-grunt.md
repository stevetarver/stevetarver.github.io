---
layout: post
title:  "My first grunt"
subtitle: "Walk through of creating a simple Grunt task"
date:   2015-09-03 
tags: grunt excel task json nodeunit npm
---

<img style="float: left;" src="/images/grunt-1.png">


I had this idea to make static site data generation easier. It worked out pretty well but I thought I might use it again, so I made it an npm package. Then I thought it would be even handier as a grunt task...  never done that before... could be fun.

How did that all work out? Pretty well actually; here's how it went.


## Background

I am working on a static site that stores configuration and volatile data in JSON files. Most of the data  is exported from other systems - about 62 files in all. When faced with the prospect of editing each of those files for every initial development change, and every backend data change, I quickly decided that I wanted everyone on the team to be able to update this data. Anyone can edit an Excel file, so, I wrote a script to export Excel as JSON.

To make things even easier, I decided that I wanted to add this as a build step so every time I distribute the site, it has the freshest data. Seems there are two types of Grunt tasks:

- Tasks that provide pure build functionality
- Tasks that wrap or aggregate existing functionality

This Grunt task is the latter. Here is the Grunt task source and the underlying functionality source

## Getting Started

Grunt provides some pretty decent documentation to get you started, so even if you know little about Grunt internals, you can create a task after reading a couple of pages.

Seems the most difficult task is choosing a package name. It needs to start with 'grunt', cannot start with 'grunt-contrib', and should reflect what the module does, without an npmjs.org collision - that's usually the hard part. My choice was easy because the underlying functionality already had a distinct name (although excruciatingly boring).

Creating the plugin boilerplate consists of:

```bash
npm install -g grunt-init
git clone git://github.com/gruntjs/grunt-init-gruntplugin.git ~/.grunt-init/gruntplugin
mkdir grunt-excel-as-json
cd grunt-excel-as-json
grunt-init gruntplugin
npm install
```

The boilerplate provides a working task, test, package.json, and Gruntfile. You can run the default task and see the tests execute.


## README.md

For me, README.md is the start of any project. It helps me focus on how users will interact with the product instead of where my head is usually at, mired in details. But before we go further, a little rant.

> Finding quality packages in npmjs.org is hard. On the search results page, you can see stars, but people don't really use the star rating system: browserify has nearly 2 million downloads a month and 388 stars. On the individual home pages, you can't see stars but you can see downloads. On less frequently used modules, you can't really tell experimental use from production use by download count. I have wasted a lot of time experimenting with modules that looked promising, but could never get working. Now, I choose modules based on my impression of their documentation quality. If it looks thorough and complete, I will try it, knowing that I can figure out if it is the module I need quickly - fast fail.

What exactly do I want this task to do? The excel-as-json package does just one thing: convert an Excel sheet to a JSON file. I want the user to be able to:

1. specify several 1:1 file mappings
1. include some options for each mapping
1. specify different targets like 'dist' and 'test'
1. be succinct

## TDD (v0.0.1)

In TDD/BDD fashion, I comment out the task and test boilerplate guts, insert a console.log() message in the task, and run `grunt test`.

Something amazing happened: my console.log message happened during Gruntfile task but not during the test run. Insert TLA here.

In the Gruntfile, on top of my task configuration, I found

```// Configuration to be run (and then tested).```

And then in the NodeUnit test I found

```js
var actual = grunt.file.read('tmp/default_options');
var expected = grunt.file.read('test/expected/default_options');
test.equal(actual, expected, 'should describe what the default behavior is.');
```

The tests don't interact with my task code, they evaluate the results of my task.

What to do? What to do?  Replace the test framework and figure out how to mock the Grunt environment? Nah. excel-as-json is responsible for testing its functionality; I just need to prove my task calls it properly. Not ideal or robust, but workable.


## TDD (v0.0.2)

Now that I understand the NodeUnit tests (not a fan by the way), I need to set up a proper grunt task configuration to generate something for my test to evaluate.

First to pick a name. Turns out that neither the task file name or the task configuration name need to match the package. Grunt will load all the tasks in your package.

The name that grunt-init picked for the task kinda sucked: excel_as_json.js. So I changed the file and the config name to convertExcelToJson. In the task file, this became

{% highlight js %}
grunt.registerMultiTask('convertExcelToJson', 'Convert Excel files to JSON files', function() {
{% endhighlight %}

Beyond that, I needed a little help, so off to Grunt doc for creating tasks and file formats. Looks like the Files Array format satisfies requirements 1, 2, and 4. So the configuration would look like

```js
convertExcelToJson: {
  dist: {
    files: [
      {src: 'test/fixtures/row-oriented.xlsx', dst: 'tmp/row-oriented.json'},
      {src: 'test/fixtures/col-oriented.xlsx', dst: 'tmp/col-oriented.json', isColOriented: true}
    ]
  }
},
```

After creating the Excel test files, time for another test. Of course the first run didn't work.


## TDD (v0.0.3)

Building the project with `grunt --stack` gets you to the failure point quickly.

The heart of the task is iterating over the files collection:

```js
this.files.forEach(function(f) {
```

Grunt has its own logging system, so I inserted some log statements to show what arguments were being passed. Turns out that the source files are always passed as an array, even if there is only one item. Problems solved.

## Async

Reading and writing files should be an asynchronous operation - how does Grunt deal with that? What happens when Grunt finishes before the asynchronous excel-as-json package does.

Grunt has robust file manipulation utilities; file handling is at its core. Within the task, you have your task's entire configuration, and some other handy things attached as this.* properties and documented here.

If you have asynchronous operations, you initialize the async facility

```CoffeeScript
grunt.registerMultiTask('convertExcelToJson', 'Convert Excel files to JSON files', function() {
  var done = this.async();
```

To register the task as an asynchronous operation and call done() when all operations are complete.

In my case, I have potentially many asynchronous file reads and writes and need to call done() exactly once when the last file is processed. Simple solution: counter.

```js
var fileCount = this.files.length;
var filesProcessed = 0;

this.files.forEach(function(f) {
  // Convert a single file asynchronously  convertExcel(f.src[0], f.dst, f.isColOriented, function(err, data) {
    // When the last file is processed, signal done    if(++filesProcessed === fileCount) {
      done();
    }
  });
});
```


## Protect the user

I have wasted so much time struggling with npm packages that I really don't want to be that guy, you know, the one you curse because their package looks like exactly what you need but takes forever to get working.

So the next step for me was to imagine every way that a user could screw up and try to protect against it. These are your task sanity checks. I'm actually pretty good at this. So I bungled the configuration every way I could and provided some help log entries that should make short work of fixing the configuration. Grunt provides some help here.

If things go horribly wrong and you think you should abort the build, use grunt.fail.fatal

```js
if (!grunt.file.exists(f.src[0])) {
  grunt.fail.fatal('convertExcelToJson: Source file "' + f.src[0] + '" not found. Cannot continue.');
```

to produce

```bash
Running "convertExcelToJson:dist" (convertExcelToJson) task
Fatal error: convertExcelToJson: Source file "test/fixtures/ro-oriented.xlsx" not found. Cannot continue.
```

Using grunt.fail.warn still aborts the build but with a less strong message

```js
if (f.src.length > 1) {
  grunt.fail.warn('convertExcelToJson: Multiple source files not supported: using only the first one.');
}
```

produces

```bash
Running "convertExcelToJson:dist" (convertExcelToJson) task
Warning: convertExcelToJson: Multiple source files not supported: using only the first one. Use --force to continue.
```

For non-fatal errors that just suggests they are doing something wrong, grunt.log.warn seems like the best solution. It logs a little differently but does not bail on the build.

```bash
Running "convertExcelToJson:dist" (convertExcelToJson) task
>> convertExcelToJson: Multiple source files not supported: using only the first one.
```

## Cleaning up and publishing

Now that functionality is in place, rudimentary testing passes, time to install into a local project to verify everything works.

You can create a peer directory, `npm init`, and then

```bash
npm install ../grunt-excel-as-json
```

Once that testing passes, you can control what is published by adding a `.npmignore` file. Perhaps a `.travis.yml`? Commit to GitHub, create a release, and then the normal npm publish procedure.

## Epilog

On the plus side

- With a day's poking around, I created a Grunt task that cleans up my build
- I can build my next trivial Grunt task in a couple of hours
- Overall, pretty positive experience

Needs some work

- NodeUnit feels pretty simple - want something more robust
- Unit tests don't interact with code, test cases limited to successful task output
- Since the unit tests only evaluate results, cannot test negative cases
- No code coverage?





