---
layout: post
title: identity_uri in Auth Token Middleware
date: 2014-05-21 14:54
comments: true
categories: keystone openstack
---

As part of the 0.8 release of keystoneclient (2014-04-17) we made an update to the way that you configure auth\_token middleware in OpenStack.

Previously you specify the path to the keystone server as a number of individual parameters such as:

```
[keystone_authtoken]
auth_protocol = http
auth_port = 35357
auth_host = 127.0.0.1
auth_admin_prefix =
```

This made sense in code when using httplib for communication where you use each of those independent pieces.
However we removed httplib a number of releases ago and now simply reconstruct the full URL in code in the form:

```
%(auth_protocol)s://%(auth_host)s:%(auth_port)d/%(auth_admin_prefix)s
```

This format is much more intuitive for configuration and so should now be used with the key **identity\_uri**. e.g.

```
[keystone_authtoken]
identity_uri = http://127.0.0.1:35357
```

Using the original format will continue to work but you'll see a deprecation message like:

```
WARNING keystoneclient.middleware.auth_token [-] Configuring admin URI using auth fragments. This is deprecated, use 'identity_uri' instead.
```
