---
layout: post
title: "PKI tokens don't give better security"
date: 2014-12-01 12:55:29 +1000
comments: true
categories: keystone openstack
---

This will be real quick.

Every now and then I come across something that mentions how you should use PKI tokens in keystone as the cryptography gives it better security.
It happened today and so I thought I should clarify:

__There is no added security benefit to using keystone with PKI tokens over UUID tokens.__

There are advantages to PKI tokens:

 - Token validation without a request to keystone means less impact on keystone.

And there are disadvantages:

 - Larger token size.
 - Additional complexity to set up.

However the fundamental model, that this opaque chunk of data in the 'X-Auth-Token' header indicates that this request is authenticated does not change between PKI and UUID tokens.
If someone steals your PKI token you are just as screwed as if they stole your UUID token.
