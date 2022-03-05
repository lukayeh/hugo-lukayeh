---
title: "Starting a Hugo Site"
date: 2022-03-04
draft: false
tags:
 - "hugo"
 - "tech"
categories:
 - "techstuff"
---

# Introduction

Hi! this is going to be my first blog post here, and it's going to pretty much run through how I generated it in the first ***Hazzar I hear you say***. 

If you want to skip this and get the quick start on how to get HUGO up and running go ahead and read this document [here](https://gohugo.io/getting-started/quick-start/) from Hugos standard documentation or you can read my blog?

Just as a headsup I'm running on a MacBook Pro (15-inch, 2018), so this post atleast is tailored to MacOS if you need assistance with Windows then...well figure it out ![lolleo random](https://emojis.slackmojis.com/emojis/images/1643515011/10406/lolleo.png?1643515011)

# Getting Started

## Install Hugo

`Homebrew` is a package manager for  `macOS`, can be installed from  [brew.sh](https://brew.sh/).

```bash
brew install hugo
```

To verify your new install:

```bash
hugo version
```
My output:
`hugo v0.92.0+extended darwin/amd64 BuildDate=unknown`

## Create A New Site

Navigate to wherever you like to store your training materials personally I keep them under `~/training` then run:

```bash
hugo new site playground
```
*Just name it whatever you like, i'm going to call this one playground*
hugo will give you a message like so if its all worked out well:

```
Congratulations! Your new Hugo site is created in /training/playground.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```

## Add a Theme

Now this is where I sank alot of time playing with themes and deciding what I'd like to show! So go make a coffee and get ready to browse a endless stream of themes.

See [themes.gohugo.io](https://themes.gohugo.io/) for a list of themes to consider.

Because it caught my eye I'm going to use: https://github.com/adityatelange/hugo-PaperMod/wiki/Installation

Prior to adding the theme lets `git init` out directory:
```bash
cd playground
git init
```

As per their guidelines I used submodule to install and here we go:
```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod --depth=1
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
```
Hugo has a caveat here for non-git-users, but if you're not using git then realistically why are you here ;)

Now update config.yml to look like this:
```
baseURL = 'http://example.org/'
languageCode = 'en-us'
title = 'My New Hugo Site'
theme = 'PaperMod'
```


## Content
So you now need to start adding some of those rad ideas you have into a blog, or even just copy someone elses blog posts and slightly change the wording, well lets do just that..

Use the  `new`  command to generate a new post:

```bash
hugo new posts/my-first-post.md
```
Alternatively you can just create the files manually under: `content/<CATEGORY>/<FILE>` whatever floats your boat.

Edit the newly created content file it'll be here: `playground/content/posts/my-first-post.md`:

```markdown
---
title: "My First Post"
date: 2019-03-26T08:47:11+01:00
draft: true
---
```
Add whatever you like! 

Also I wish I'd seen this part before I started getting ahead of myself but I'll explain that in a future blog post *(he says completely aware he may not create anymore than this..)*

*"Drafts do not get deployed; once you finish a post, update the header of the post to say `draft: false`. More info [here](https://gohugo.io/getting-started/usage/#draft-future-and-expired-content)."*

## Running that hugo server

Alright we've messed around enough, you've probably spend forever playing with themes and it'll all go pete tong here but lets give it a go.

To run your server with drafts enabled do the following:

`hugo server -D`

**Navigate to your new site at  [http://localhost:1313/](http://localhost:1313/).**

Now you can play with your site locally to your hearts content!
Modify content, play with themes the worlds your oyster 

(I mean it isn't have you seen the news these days?!)

If you've not messed around too much and broke stuff which like me you probably have you'll have a running static website to view here http://localhost:1313

It may even look like this:
![](https://i.imgur.com/zjRIeJf.png)

## Build static pages

I legit just stole this part from the Quick Start guide on Hugo's websit ebut It is simple. Just call:

```
hugo -D

```

Output will be in  `./public/`  directory by default (`-d`/`--destination`  flag to change it, or set  `publishdir`  in the config file).

## Git this stuff

If you're like me you'll want to keep a copy of this, so you need a Github account.

Google how to get one of those and go ahead create a new empty repository.

```bash
git init #if you haven't ran it already
git add .
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:<REPO_STUFF_HERE>.git
git push -u origin main
```
and that is it, you've created a static website, stored it in source and now you're phyiscally and emotionally exhausted.

Great job!