---
title: "Suckless: An Introduction"
date: 2020-03-24T12:36:53Z
draft: false
toc: true
images:
tags:
  - suckless
  - linux
---


## Introduction

Whether it's slack or chrome, people are constantly making jokes about bloated software.
The programs in the suckless ecosystem are all very minimal and bloat free.
The idea is that they come with the minimal amount of code to get the job done, and then the user can patch the codebase to add any features they want.
The other 'different' thing out suckless programs is that config is not loaded at runtime, but is stored in a `config.h` file that is compiled into the binary, so if you make a config change you need to recompile.

I have been using suckless programs for a while now and have a method I use to maintain my config and manage my repos.
When I first discovered suckless I couldn't find many resources that showed how to do this.
This blog post is my attempt to make the suckless programs more accessible to users who are newer to the world of linux and opensource.


## Note

This blog post assumes some basic knowledge of how to use git and the command line.

If you want to learn more about how to use git I would advise looking at [Roger Duler's git guide](https://rogerdudler.github.io/git-guide/) (this helped me out loads when I was learning git).

Most command line tools / programs either have a man page or a help option.
This means if you can confused about a command, e.g. A`patch`, you can run `man patch` or `patch --help` to get some useful information about how to use it.

I am by no means an expert.

I am writing this blog post as an attempt to be helpful and benefit the community.
If you have any suggestions, spot a mistake, have something to add, or think I have misunderstood anything: feel free to either email me or tweet or something like that.
Also let me know of any other good resources that you think I should link to in this blog post.

With all that out the way, lets begin.


## Getting Started

As an example I will walk you through setting up a program like [st](https://st.suckless.org/).
st is the terminal program created by suckless.

We will be using `git` a lot when working with suckless programs, so make sure you have it installed (if you are interested in suckless I assume you know the basics of git).

### Setting up the git repo

First we need to download all of the source code for st. 
I keep all of my suckless programs in a directory called suckless, but you can manage your file system how you like.

```bash
cd ~/suckless
git clone https://git.suckless.org/st
```

We are going to be configuring st, but will want to be able to get any updated make by the developers.
In my case, I also want to have my custom build on gitlab.
To do this I first created a repository on [gitlab](https://gitlab.com) (you can use github, or any other git repo manager).
I then changed the URL for the origin remote to be that of my gitlab repo and created a new remote called upstream for the official repo.
All changes I make will be in a branch called custom.

```bash
git remote set-url origin git@gitlab.com:mokytis/st.git
git remote add upstream https://git.suckless.org/st
git checkout -b custom
```

### Making config changes

We are now going to edit the config for st.
Make sure that you are in the custom branch.

```bash
git checkout custom
```

st provides us with the file `config.def.h`.
This file acts as a template for our config, which will be in the file `config.h`.
So to start making config changes, make a copy of `config.def.h` called `config.h`.

```bash
cp config.def.h config.h
```

Now open up the `config.h` file and make some changes.
I like to use the [Fira Code typeface](https://github.com/tonsky/FiraCode) in my terminal, and I don't like having the border in my terminal, so I will change the first few line to look like this.

```c
static char *font = "Fira Code:pixelsize=15:antialias=true:autohint=true";
static int borderpx = 2;
```

When you have made a change you can run `sudo make clean install` and rerun `st` to see how it looks.
If you are happy with the changes you have made and you can push them to your remote repo.

```bash
git push -u origin custom
```

### Patching

Patches are [diffs](https://en.wikipedia.org/wiki/Diff) created by the community that add a feature to a suckless program.
There is a list of st patches at [https://st.suckless.org/patches/](https://st.suckless.org/patches/).

Here is a list of patches I like to use:
* [anysize](https://st.suckless.org/patches/anysize/) - allows the st window to be any size (not just a multiple of character size) resulting in no gaps between windows
* [alpha_focus_highlight](https://st.suckless.org/patches/alpha_focus_highlight/) - adds two transparency values, one for focused and unfocused windows
* [scrollback](https://st.suckless.org/patches/scrollback/) - allows you to scroll through terminal output

If we want to use a patch we first need to download the diff file and save it in out `suckless/st` directory.
We can do that using `wget`.
The `patch` command can then be used to update out source files to contain the patch.

```bash
wget https://st.suckless.org/patches/anysize/st-anysize-0.8.1.diff
patch < st-anysize-0.8.1.diff
```

In my case I applied the `st-anysize` patch which modifies the `x.c` file.
I can not commit this change to my custom st build branch and push it to gitlab.

```bash
git add x.c
git commit -m "applied st-anysize-0.8.1.diff"
git push origin custom
```

If you want to reverse a patch you can do this using the `-R` flag.

```bash
patch -R < st-anysize-0.8.1.diff
```

Make sure to commit any changes you make, when you make them.
A commit should contain one change (I forget this a lot, and it ends up being a pain in the future, so try to remember).

### Make config

The `config.mk` file is used when compiling `st`.
I like to make a small change to this file which allows me to build and instal st without root (build to a local directory).
To do this, I just change the `PREFIX` variable.

```mk
PREFIX = ~/.local/
```

As always, I commit this change and push it to gitlab.

```bash
git add config.mk
git commit -m "make now installs localy, no longer requireing root"
git push origin custom
```

### Merging with new version of st

Inevitably the `st` developers will release a new version and you will want to update, but keep your custom config.
What do you do?
First you need to pull down the new version of st.
Then you can merge the changes with your custom branch.
`git mergetool` allows you to fix any merge conflicts (where both you and the st devs have edited the same part of a file)

```bash
git pull upstream master
git checkout custom
git merge master
git mergetool
git commit -m "merged with master"
git push origin custom
```

## Help! I did it wrong

I know I make mistakes from time to tome when using the command line.
Here I am going to document some mistakes I make, and how I fix them.
This list is likely to grow over time.
Feel free to contact me with any common mistakes and remedies you think should be added.

### I committed my change to the wrong branch

You have just commit a fresh config change then realise you committed it to `master` not `custom`.
What do you do?

```bash
git reset --soft HEAD^
git checkout custom
git commit
```

How does this work?
* `git reset` changes to state of git to a previous state
* `HEAD^` points to one commit older than HEAD (current state)
* `--soft` tells git to not modify the working tree (i.e. the change you make stay but are not committed any more)
* `git checkout custom` moves us back to the `custom` branch

