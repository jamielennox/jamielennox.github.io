---
layout: post
title: "V3 authentication with auth_token middleware"
date: 2015-02-23 10:57:46 +1100
comments: true
categories: keystone openstack
---

Auth\_token is the middleware piece in OpenStack responsible for validating tokens and passing authentication and authorization information down to the services.
It has been a long time complaint of those wishing to move to the V3 identity API that auth\_token only supported the v2 API for authentication.

Then auth\_token middleware adopted authentication plugins and the people rejoiced!

Or it went by almost completely unnoticed.
Auth is not an area people like to mess with once it's working and people are still coming to terms with configuring via plugins.

The benefit of authentication plugins is that it allows you to use [any plugin you like for authentication](http://www.jamielennox.net/blog/2015/02/17/loading-authentication-plugins/) - including the v3 plugins.
A downside is that being able to load any plugin means that there isn't the same set of default options present in the sample config files that would indicate the new options available for setting.
Particularly as we have to keep the old options around for compatibility.

The most common configuration I expect for v3 authentication with auth\_token middleware is:

```ini
[keystone_authtoken]
auth_uri = https://public.keystone.uri:5000/
cafile = /path/to/cas

auth_plugin = password
auth_url = http://internal.keystone.uri:35357/
username = service
password = service_pass
user_domain_name = service_domain
project_name = project
project_domain_name = service_domain
```

The `password` plugin will query the `auth_url` for supported API versions and then use either v2 or v3 auth depending on what parameters you've specified.
If you want to save a round trip (once on startup) you can use the `v3password` plugin which takes the same parameters but requires a V3 URL to be specified in `auth_url`.

An unfortunate thing we've noticed from this is that there is going to be some confusion as most plugins present an `auth_url` parameter (used by the plugin to know where to authenticate the service user) along with the existing `auth_uri` parameter (reported in the headers of 403 responses to tell users where to authenticate).
This is a known issue we need to address and will likely result in changing the name of the `auth_uri` parameter as the concept of an `auth_url` is used by all existing clients and plugins.

For further proof that this works as expected checkout [devstack](https://github.com/openstack-dev/devstack/blob/5ce44cd63b6e2b53f08a6b4b87cb4ab11d1ade26/lib/keystone#L448) which has been operating this way for a couple of weeks.


_NOTE:_ Support for authentication plugins was released in keystonemiddleware 1.3.0 released 2014-12-18.
