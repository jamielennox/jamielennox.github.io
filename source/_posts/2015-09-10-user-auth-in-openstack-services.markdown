---
layout: post
title: "User auth in OpenStack services"
date: 2015-09-10 15:19:24 +1000
comments: true
categories: keystone openstack
---

With auth plugins we are trying to ensure that an individual OpenStack service (like Nova or Glance) should never have to deal with the details of authentication.
One of the improvements we've made that has gone largely unnoticed is the addition of the __keystone.token_auth__ authentication plugin that is passed down in a request's environment variables from auth_token middleware.
This object is a full authentication plugin that uses the token and service catalog of the user that was just validated so that the service does the right thing without having to figure out keystone's token format.

This means that service to service communication is as simple as:

```python
from glanceclient import client
import json
from keystoneclient import session
from keystonemiddleware import auth_token
from oslo_config import cfg
import webob.dec
from wsgiref import simple_server

cfg.CONF(project='testservice')

session.Session.register_conf_options(cfg.CONF, 'communication')
SESSION = session.Session.load_from_conf_options(cfg.CONF, 'communication')


@webob.dec.wsgify
def app(req):
    glance = client.Client('2',
                           session=SESSION,
                           auth=req.environ['keystone.token_auth'])

    return webob.Response(json.dumps([i.name for i in glance.images.list()]))


app = auth_token.AuthProtocol(app, {})
server = simple_server.make_server('', 8000, app)
server.serve_forever()
```

This is a full service that responds to every request with a JSON formatted list of image names in your project which is not all that useful but proves a point. There are some things to notice:

####The session is global.

There are two ways to use a session with authentication.

* If you are writing something like a CLI application then you will want to use the same authentication for the lifetime of the program and it can be easier to just pass an auth plugin to the sesion constructor and forget about it.
* If you are writing a service that wants to use many different authentications over its lifetime you can pass the auth directly to the client that will consume it.

In our case we want to re-use the session for benefits like connection pooling and caching, but we will often change the authentication being used so we pass the plugin to the client directly.
The session is thread safe and is able to be reused across requests like this.
Consider this to be splitting the application context and the request context.

####We create the glanceclient just in time.

As a very small application this isn't obvious however because all the caching and authentication logic is being handled by the session and plugin there is no reason to keep a client around.
Clients become very cheap to create and so in most situations you can use a client object within a function and then discard it.

####We never entered a URL for Glance.

At no point did we have to provide a URL for glance in the config file.
If on any project you encounter you have to enter a fixed URL to communicate with another service please file a bug.
Keystone tokens have a service catalog in them so that all requests made on behalf of a user go to the appropriate URL.
In the past this was a relatively ugly affair involving parsing the information from dictionaries, however this is all encapsulated into the auth plugin now.

####There is additional information on the plugin

Whilst not shown in the example the auth\_plugin has the following attributes:

* user.auth\_token
* user.user\_id
* user.user\_domain\_id
* user.project\_id
* user.project\_domain\_id
* user.trust\_id
* user.role\_names

If you are storing the auth plugin in a context using these accessors can be much easier that trying to figure out the variables that auth_token middleware also set.

####You can't serialize the auth plugin.

In the case of Nova and others the auth\_token middleware check is performed on the API service however most service communication is done in a backend service.
We currently have no way of serializing the plugin to an oslo.context so it is reconstructed on the backend.
This is something we are working on.

####It's available now

Going back to look at the initial review it is 5 days shy of 1 year old (merged 2014-09-15).
There have been improvements since then however the basic functionality has been out for a while and is available in the current minimum global requirements.
Glanceclient on the other hand has only had session support since the 1.0 release (2015-08-31) so you will need a recent version to test the example.

##Conclusion

We are doing all we can to prevent services ever having to deal with the details of authentication in OpenStack.
If your project has still not adopted plugins please come find us in #openstack-keystone on freenode as it's currently making your life more difficult.
