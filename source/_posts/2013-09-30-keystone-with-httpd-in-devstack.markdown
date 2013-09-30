---
layout: post
title: "Keystone with HTTPd in devstack"
date: 2013-09-30 13:53
comments: true
categories: openstack devstack keystone
---

Keystone has been slowly pushing away from being deployed with [Eventlet](http://eventlet.net) and the ``keystone-all`` script in favour of the more traditional httpd mod\_wsgi application method.
There has been discussion of Eventlet's place in OpenStack [before](http://davidhadas.wordpress.com/2012/05/14/asynchronousio/) and its (mis)use has led to numerous subtle bugs and problems, however from my opinion in Keystone the most important reasons to move away from Eventlet are:

* Eventlet does not support Kerberos authentication.
* pyOpenSSL only releases the GIL around some SSL verification commands.
  This leads to a series of hacks to prevent long running crypto commands blocking Eventlet threads and thus the entire Keystone process.
* There are already a lot of httpd authentication/authorization plugins that we could make use of in Keystone.
* It's faster to have things handled by httpd modules in C than in Python.

Keystone has shipped with sample WSGI scripts and httpd configuration files since Foslom and documentation for how to use them [is available](http://docs.openstack.org/developer/keystone/apache-httpd.html) however most guides and service wrappers (upstart, systemd etc) will use the ``keystone-all`` method.

To get some wider adoption and understanding of the process I've just added Keystone with httpd support into devstack.
Simply set:
```
APACHE_ENABLED_SERVICES=key
```
in your localrc or environment variables and re-run ``./stack.sh`` to try it out.

P.S. Swift can also be deployed this way by adding ``swift`` to the (comma separated) services list.
