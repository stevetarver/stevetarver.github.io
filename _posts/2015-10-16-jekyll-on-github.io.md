---
layout: post
title:  "First GitHub.io on Jekyll"
date:   2015-10-16
tags: github pages jekyll static-site macdown
---

**TL;DR**: Up and running with jekyll on github.io

<img style="float: left;" src="/images/logo/jekyll-github-logo.png">

Curiously, programmers repeat tedious processes until some level of pain or annoyance is met. They could fix the problem straight away, but they endure until it pisses them off sufficiently and they finally fix it. But, ... then again, maybe that's just me and my lazy system of prioritization or system of lazy prioritization.

I finally reached that point with blogspot.com. The tedium of using anything but their standard layouts, adding the right number of blank lines after headings, and especially syntax highlighting code snippets finally put me there. I have such great tools for doing everything else in my life, why is it so hard to just capture my thoughts and have a descent layout.

All I want is a place:

- that is free
- that I have decent presentation that is easy to change
- where I can effortlessly capture my thoughts
- where technical posts (syntax highlighting) is no problem

I looked at several options and there was there were no clear winners. I originally passed on github.io because there was no clear getting started guide with the features I wanted. My journey brings me back, and with a small effort, I found the perfect host:

- version control as a site provider
- write in MarkDown with the excellent [MacDown](http://macdown.uranusjr.com)
- many themes to use as a starting point
- local hosting for site development

**Solution**: github.io with Jekyll

I started with the default site and mehh... I tried the github.io pre-fab sites and they were OK, a definate improvement. then I made the leap to a Jekyll site.

Here is a guide to setting up your first github.io site with jekyll in about 7 minutes - even if you have never used ruby, Jekyll, or a static site generator.

## Setup

### References

- [Using Jekyll with Pages](https://help.github.com/articles/using-jekyll-with-pages/)
- [Jekyll Quick-start guide](http://jekyllrb.com/docs/quickstart/)
- [Jekyll GitHub Pages](http://jekyllrb.com/docs/github-pages/)

### Create and run the site

3. Install and configure XCode. I know, a given, but if you automatically update and don't frequently use XCode, you can run into a couple of obscurely annotated build errors. Start XCode and agree to terms and licence to avoid them. To verify command tools are installed:
	
	```
	xcode-select --install
	```

1. Install node.js. You should already have this installed, but if not, it is best done from installers [here](https://nodejs.org)
1. Install Jekyll

   ```
   sudo gem install jekyll
   ```

1. Create a new local site using the GitHub site naming convention {username}.github.io

   ```
   jekyll new <username>.github.io
   ```
   
4. Create a Gemfile in your project root


   ```  
   source 'https://rubygems.org'   
   gem 'github-pages'
   ```

3. Install ruby gems

   ```
   sudo gem install bundler
   bundle install
   ```

1. Run the site

   ```
   jekyll serve
   ```
   
1. View the site at [http://localhost:4000](http://localhost:4000)
2. Revise the site. Set your title, description, and other insteresting things in ```_config.yml``` Also, add this to set standard highlighting. [More detail](http://jekyllrb.com/docs/templates/#code-snippet-highlighting)

	 ```
	 highlighter: pygments
	 ```

### Add the site to GitHub

2. Create the standard repo in GitHub ```{username}.github.io```
3. Link your local project to the GitHub repo. For me this command looked like

   ```
   git init
   git remote add origin https://github.com/stevetarver/stevetarver.github.io.git
   ```
      
1. Add project files to git
2. Commit the directory and push to GitHub
3. Your changes should be immediately viewable on http://{username}.github.io


## Select a theme

### References

- [Jekyll Templates](http://jekyll.tips/templates/)
- [Jekyll themes](http://jekyllthemes.org)

There are a couple of central theme sites and some tools that make changing themes easy. For a first cut, the basic procedure is to find a them you like, clone it, and copy its files to your project. 

Jekyll has a lot better separation of post and presentation than blogspot. In blogspot, every theme change required that I re-install the syntax highlighting solution in the main html page.

Some tips:

1. Add a _drafts directory to hold posts that are in the works.
1. Add an images directory to store site images
1. _posts file naming convention is YYYY-MM-DD.post name.md
1. How to Displaying an index of posts: http://jekyllrb.com/docs/posts/
1. You can evolve your site as the mood motivates
1. Server your drafts with jekyll serve --drafts


## Theme Directories

The hardest part of changing themes is finding one you like. Here are some directories to look through:

- [dr. jekyll's themes](https://drjekyllthemes.github.io)
- [Jekyll Themes](*http://jekyllthemes.org)   
- [Jekyll Themes & Templates](http://jekyllthemes.io)

After that, download/clone the repo and selectively copy into your github.io repo.
