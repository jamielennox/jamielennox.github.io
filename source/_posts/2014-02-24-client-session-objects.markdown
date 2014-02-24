---
layout: post
title: "Client Session Objects"
date: 2014-02-24 13:32
comments: true
categories: openstack
---

Keystoneclient has recently introduced a _Session_ object.
The concept was discussed and generally accepted at the Hong Kong Summit that keystoneclient as the root of authentication (and arguably security) should be responsible for transport (HTTP) and authentication across all the clients.

The majority of the functionality in this post is written and up for review but has __not yet been committed__.
I write this in an attempt to show the direction of clients as there is currently a lot of talk around projects such as the [OpenStack-SDK](https://wiki.openstack.org/wiki/SDK-Development).

When working with clients you would first create an authentication object, then create a session object with that authentication and then re-use that session object across all the clients you instantiate.

```python

from keystoneclient.auth.identity import v2
from keystoneclient import session
from keystoneclient.v2_0 import client

auth = v2.Password(auth_url='https://localhost:5000/v2.0',
                   username='user',
                   password='pass',
                   tenant_name='demo')

sess = session.Session(auth=auth,
                       verify='/path/to/ca.pem')

ksclient = client.Client(session=sess,
                         region_name='RegionOne')
# other clients can be created sharing the sess parameter

```

Now whenever you want to make an authenticated request you just indicated it as part of the request call.

```python

# requests with authenticated are sent with a token
users = sess.get('http://localhost:35357/v2.0/users',
                 authenticated=True)

```

This was pretty much the extent of the initial proposal, however in working with the plugins I have come to realize that authentication is responsible for much more than simply getting a token.

A large part of the data in a keystone token is the service catalog.
This is a listing of the services known to an OpenStack deployment and the URLs that we should use when accessing those services.
Because of the disjointed way in which clients have been developed this service catalog is parsed by each client to determine the URL with which to make API calls.

With a session object in control of authentication and the service catalog there is no reason for a client to know its URL, just what it wants to communicate.

```python

users = sess.get('/users',
                 authenticated=True,
                 service_type='identity',
                 endpoint_type='admin',
                 region_name='RegionOne')

```

The values of `service_type` and `endpoint_type` are well known and constant to a client, `region_name` is generally passed in when instantiating (if required).
Requests made via the client object will have these parameters added automatically, so given the client from above the following call is exactly the same:

```python

users = ksclient.get('/users')

```

Where I feel that this will really begin to help though is in dealing with the transition between API versions.

Currently deployments of OpenStack put a versioned endpoint in the service catalog eg for identity `http://localhost:5000/v2.0`.
This made sense initially however now as we try to transition people to the V3 identity API we find that there is no backwards compatible way to advertise both the v2 and v3 services.
The agreed solution long-term is that entries in the service catalog should not be versioned eg. `http://localhost:5000` as the root path of a service will list the available versions.
So how do we handle this transition across the 8+ clients?
Easy:

```python

try:
    users = sess.get('/users',
                     authenticated=True,
                     service_type='identity',
                     endpoint_type='admin',
                     region_name='RegionOne',
                     version=(2, 0))  # just specify the version you need
except keystoneclient.exceptions.EndpointNotFound:
    logging.error('No v2 identity endpoint available', exc_info=True)

```

This solution also means that when we have a suitable hack for the transition to unversioned endpoints it needs only be implemented in one place.

Reliant on this is a means to discover the available versions of all the OpenStack services.
Turns out that in general the projects are similar enough in structure that it can be done with a few minor hacks.
For newer projects there is now a definitive specification [on the wiki](https://wiki.openstack.org/wiki/VersionDiscovery).

A major advantage of this common approach is we now have a standard way of determining whether a version of a project is available in this cloud.
Therefore we get client version discovery pretty much for free:


```python

if sess.is_available(service_type='identity',
                     version=(2,0)):
    ksclient = v2_0.client.Client(sess)
else:
    logging.error("Can't create a v2 identity client")

```

That's a little verbose as a client knows that information, so we can extract a wrapper:

```python

if v2_0.client.Client.is_available(sess):
    ksclient = v2_0.client.Client(sess)

```

or simply:

```python

ksclient = keystoneclient.client.Client(session=sess,
                                        version=(2,0))
if ksclient:
    # do stuff
```

So the session object has evolved from a pure transport level object and this departure is somewhat concerning as I don't like mixing layers of responsibility.
However in practice we have standardized on the [requests](http://www.python-requests.org) library to abstract much of this away and the Session object is providing helpers around this.

So, along with standardizing transport, by using the session object like this we can:

- reduce the basic client down to an object consisting of a few variables indicating the service type and version required.
- finally get a common service discovery mechanism for all the clients.
- shift the problem of API version migration onto someone else - probably me.

####Disclaimers and Notes

- The examples provided above use keystoneclient and the 'identity' service purely because this is what has been implemented so far.
In terms of CRUD operations keystoneclient is essentially the same as other client in that it retrieves its endpoint from the service catalog and issues requests to it, so the approach will work equally well.

- Currently none of the other clients rely upon the session object, I have been waiting on the inclusion of authentication plugins and service discovery before making this push.

- Region handling is still a little awkward when using the clients.
I blame this completely on the fact that region handling is awkward on the servers.
In Juno we should have hierarchical regions and then it may make sense to allow `region_name` to be set on a session rather than per client.
