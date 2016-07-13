---
layout: post
title: "The Positional Library"
date: 2016-07-13 15:27:52 +1000
comments: true
categories: openstack python
---

So one of my favourite things in python 3 syntax is:

```python
def function(arg, *, kw1, kw2=None):
```

This `*` syntax says that `kw1` and `kw2` must be presented as keyword
arguments. Being that `kw1` is after the `*` and does not have a default value
it becomes a required keyword argument.

```python
function('a', kw1=1)
```

This may seem kind of pedantic but the traditional python syntax does not
distinguish between optional arguments and positional arguments with defaults.
This means if your function has a large number of keyword arguments you cannot
assume your users don't pass all those arguments positionally (I've seen it).
This makes it difficult to refactor these functions because you cannot
reorganize or bundle those arguments up into `**kwargs` later.

In [keystoneauth](https://git.openstack.org/cgit/openstack/keystoneauth) we
have a number of functions like this that have a lot of optional arguments but
for future proofing we want to ensure people only pass them via keyword. So we
created the [positional](https://pypi.python.org/pypi/positional) library.

The [README](https://github.com/morganfainberg/positional) has a number of
detailed examples of use, but in general it provides a python 2 and 3
compatible way to ensure your users pass parameters via keyword.

```python
from positional import positional

@positional()
def function(arg, kw1=None, kw2=None):
    # do stuff

# must pass kw1 and kw2 as keyword arguments
function('a', kw1=1, kw2=2)
```

By default specifying only the positional decorator every argument with a
default value that is passed to the function must be passed via keyword.
Specifying a number in the decorator lets you control the number of arguments
that can be passed positionally to the function:

```python
@positional(2)
def function(arg, kw1=None, kw2=None):
    # do stuff

# kw1 has a default but can also be passed positionally
function('a', 1, kw2=2)
```

Later replacing this function with:


```python

@positional()
def worker(kw2=None, kw3=None):
    # do stuff

@positional(2)
def function(arg, kw1=None, **kwargs):
    worker(**kwargs)

function('a', 1, kw2=2)
```

is completely backwards compatible because there is no way a user could have
provided `kw2` as a positional argument.

I look forward to the day when we are all in a python 3 only world and
`@positional` is no longer required. Until then it has already allowed us to do
a number of library refactors that would have otherwise been much more
difficult.

Thanks to [Morgan Fainberg](https://twitter.com/MdrnStm) for the help and
upkeep on positional.
