---
layout: post
title: "Dealing with .pyc"
date: 2014-02-18 11:08
comments: true
categories: openstack python
---

I have often found that when dealing with multiple branches and refactoring patches I get caught out by left over \*.pyc files from python files that don't exist on this branch.
This bit me again recently so I went looking for options.

A useful environment variable that I found via some stackoverflow questions is: [PYTHONDONTWRITEBYTECODE](http://docs.python.org/2/using/cmdline.html#envvar-PYTHONDONTWRITEBYTECODE) which, when set, prevents python from writing .pyc and .pyo files.
This is not something that I want to set permanently on my machine but is great for development.

The other tool I use for all my python projects is [virtualenvwrapper](http://virtualenvwrapper.readthedocs.org/en/latest/) which allows you to isolate project dependencies and environments in what I think is a more intuitive way than with virtualenv directly.

Armed with the simple idea that these two concepts should be able to work together I found I was not the first person to think of this.
There are other guides out there but the basic concept is simply to set PYTHONDONTWRITEBYTECODE when we activate a virtualenv and reset it when we deactivate it.

Easy.

Add to *~/.virtualenvs/postactivate*:

```bash
export _PYTHONDONTWRITEBYTECODE=$PYTHONDONTWRITEBYTECODE
export PYTHONDONTWRITEBYTECODE=1
```

Add to *~/.virtualenvs/predeactivate*:

```bash
export PYTHONDONTWRITEBYTECODE=$_PYTHONDONTWRITEBYTECODE
unset _PYTHONDONTWRITEBYETCODE
```
