---
title: Using Minerl with Github Pages
layout: post
format: markdown
type: post
tags: minerl, github
timestamp: 1372411888
description: A tutorial on using Minerl, a static site generator written in perl, with github pages.
---
Well this does not seem convincing, but I wrote [Minerl](https://github.com/neevek/minerl) because I am going to start blogging about my study and work experience after having worked as a programmer for more than 5 years, for doing that, I need a blog and then I need a static site generator because I was planning to use [Github Pages](http://pages.github.com/), so I made one, which is also part of my study experience.

Github Pages is cool, with it you don't need to rent a VPS just to host a few megabytes of all your pages, it also saves you a few bucks per month. You don't need to manage your web server since there's no one, Github Pages also eases the process of publishing your site(simply with `git push`).

To start using Github Pages, you need to register an account on `github.com`, create a repo of the name `YOUR_NAME.github.com`, like mine for this site is [neevek.github.com](https://github.com/neevek/neevek.github.com). After the repo is created, clone it into a local directory and put pages in that directory, and when you are ready to publish the pages, push them and github does the rest for you. 

I will demonstrate in the following the process of how I build this site with Minerl and publish it on Github Pages.

First I clone the repo, currently I have the following structure:


    neevek@~$ git clone https://github.com/neevek/neevek.github.com.git
    neevek@~$ cd neevek.github.com 
    neevek@neevek.github.com[master]$ tree
    .
    ├── 2013
    │   └── 06
    │       └── 27
    │           └── introduction-to-minerl.html
    ├── CNAME
    ├── _src
    │   ├── _pages
    │   │   ├── 2013
    │   │   │   └── 06
    │   │   │       └── 27
    │   │   │           └── introduction-to-minerl.md
    │   │   ├── about.md
    │   │   ├── all-posts.md
    │   │   ├── archive.html
    │   │   ├── index.md
    │   │   └── tag.html
    │   ├── _raw
    │   │   ├── css
    │   │   │   └── default.css
    │   │   └── js
    │   │       └── hello.js
    │   ├── _templates
    │   │   ├── allposts.html
    │   │   ├── archive.html
    │   │   ├── default.html
    │   │   ├── index.html
    │   │   ├── post.html
    │   │   └── tag.html
    │   └── minerl.cfg
    ├── about.html
    ├── all-posts.html
    ├── archives
    │   └── 2013
    │       └── 06.html
    ├── css
    │   └── default.css
    ├── index.html
    ├── js
    │   └── hello.js
    └── tags
        ├── minerl.html
        └── perl.html

As you can see, there is a `_src` directory, which holds a `minerl.cfg` file and other *resource* files of the site, the reason why the directory starts with an underscore is from [this](http://help.github.com/articles/files-that-start-with-an-underscore-are-missing), so contents in `_src`(including the directory itself) will not be published as part of the site. Now when I finish this new post, I will do this:

    cd _src/
    minerl build
    cd ../
    git add .
    git commit -m 'added a new post'
    git push origin master

With a few commands, you have just published a new post in a geek way.

You may be curious about why `minerl build` output the final pages to the root of the repo, well remember that all directories that Minerl use are configurable, take a look at my `minerl.cfg` file:

    [system]
    output_dir = .. 
    raw_dir = _raw
    page_dir = _pages
    page_suffix_regex = \.(?:md|markdown|html)$
    template_dir = _templates
    template_suffix = .html
    recent_posts_limit = 5

    [template]
    author = neevek
    email = i at neevek.net

In the configuration file I have pointed `output_dir` to the parent of the current directory, isn't that easy?

Okay, now when you use Minerl, you should be clear about how to structure the directories and files of your Github Pages repo.
