---
layout: post
title: "Git Commands for Messy People"
date: 2014-08-15 10:59
comments: true
categories: openstack
---

I am terrible at keeping my git branches in order.
Particularly since I work across multiple machines and forget where things are I will often have multiple branches with different names being different versions of the same review.

On a project I work on frequently I currently have 71 local branches which are a mix of my code, some code reviews, and some branches that were for trialling ideas.
[git review](https://pypi.python.org/pypi/git-review) at least prefixes branches it downloads with `review/` but that doesn't help to figure out what was happening with local branches labelled `auth` through `auth-4`.

However this post isn't about me fixing my terrible habit it's about two git commands which help me work with the mess.

The first is an alias which I called `branch-date`:

{% codeblock lang:ini %}
[alias]
    branch-date = "!git for-each-ref --sort=committerdate --format='%1B[32m%(committerdate:iso8601) %1B[34m%(committerdate:relative) %1B[0;m%(refname:short)' refs/heads/"
{% endcodeblock %}

This gives a nicely formatted list of branches in the project sorted by the last time they were committed to and how long ago it was.
So if I know I'm looking for a branch that I last worked on last week I can quickly locate those branches.

{% img "basic-alignment center" /images/branch-date.png "Naming scheme win!" "List of branches ordered by date" %}

The next is a script to figure out which of my branches have made it through review and have been merged upstream which I called `branch-merged`.

Using git you can already call `git branch --merged master` to determine which branches are fully merged into the `master` branch.
However this won't take into account if a later version of a review was merged, in which case I can probably get rid of that branch.

We can figure this out by using the `Commit-Id:` field of our Gerrit reviews.

{% gist a7f8080e42ec795945b8 git-branch-merged.sh %}

So print out the branches where all the `Commit-Id`s are also in master.
It's not greatly efficient and if you are working with code bases with long histories you might need to limit the depth, but given that it doesn't run often it completes quickly enough.

There's no guarantee that there wasn't something new in those branches, but most likely it was an earlier review or test code that is no longer relevant.
I was considering a tool that could use the `Commit-Id` to figure out from gerrit if a branch is an exact match to one that was previously up for review and so contained no possibly useful experimenting code, but teaching myself to clean up branches as I go is probably a better use of my time.
