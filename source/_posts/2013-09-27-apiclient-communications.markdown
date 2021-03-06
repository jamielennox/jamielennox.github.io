---
layout: post
title: "APIClient Communications"
date: 2013-09-27 15:27
comments: true
categories: openstack keystone
---

There has been interest recently in porting novaclient's authentication plugin system to the rest of the OpenStack client libraries and moving the plugins into keystoneclient.
At a similar time [Alessio Ababilov](http://aababilov.wordpress.com) started trying to introduce the concept of a [common base client](http://openstackgd.wordpress.com/2013/01/12/preconditions-for-common-openstack-client-library/) into keystoneclient.
This is a fantastic idea and one that is well supported by the Keystone, Oslo and I'm sure other teams.
I've been referring to this move as APIClient as that is the name of the folder in Oslo code.
At its core is a change in how clients communicate that will result in some significant changes to the base client objects and incorporate these plugins.

Keystone is interested in handling how communication is managed within OpenStack, not just for tokens but as we bring in client certificate and kerberos authentication it will need to have influence over the requests being sent.
After discussing the situation with Alessio he agreed to let me take his base work and start the process of getting these changes into keystoneclient with the intent that this pattern be picked up by other OpenStack clients.
This has unfortunately been a slower process than I would have liked and I think it is hampered by a lack of clear understanding in what is trying to be achieved, which I hope to address with this post.
What follows is in the vein of Alessio's ideas and definitely a result of his work but is my own interpretation of the problem and the implementation has been rewritten from that initial work.


Most OpenStack clients have the concept of a HTTPClient which abstracts the basic communication with a server, however projects differ in what this object is and how it is used.
Novaclient creates an instance of a HTTPClient object which it saves as ``self.client`` (for yet another candidate for what a client object is).
Much of what the novaclient object does then in terms of setting and using authentication plugins is simply a wrapper around calls to the HTTPClient object.
Managers (the part of client responsible for a resource eg user, project etc) are provided with a reference to the base client object (this time saved as ``api``) and so make requests in the form ``self.api.client.get``.
Keystoneclient subclasses HTTPClient and managers make calls in the form ``self.api.get``.
Other projects can go either way depending on which client they were using as reference.

My *guess* here is that when keystoneclient was initially split out from novaclient the subclassing of HTTPClient was intentional, such that keystoneclient would provide an authenticated HTTPClient that novaclient would use.
Keystoneclient however has its own managers and requirements and the projects have sufficiently diverged so that it no longer fits into this role.
To this day novaclient does not use keystoneclient (in any way) and introduced authentication plugins instead.

If there is going to be a common communication framework then there must be a decision between:

* Standardizing on a common base client class that is capable of handling communication (as keystoneclient does).
* Create a standalone communication object that clients make use of (as novaclient does).

The APIClient design goes for the latter.
We create a communication object that can be used by any type of client _and_ be reused by different instances of clients (which novaclient does not currently allow).
This communication object is passed between clients deprecating some of the ever increasing list of parameters passed to clients and changes the flow from authenticating a client to authenticating a channel that clients can make use of.
This centralizes authentication and token fetching (including kerberos and client certs), catalog management and endpoint selection and will let us address caching, HTTP session management etc in the future.

In the initial APIClient this object was the new HTTPClient, however this term was so abused I am currently using ClientSession (as it is built on the requests library and is similar to the requests.Session concept) but debate continues.

This is where authentication plugins will live so that any communication through a ClientSession object can request a token added from the plugin.
Maintaining the plugin architecture is preferred here to simply having multiple ClientSession subclasses to allow independent storing and caching of authentication, plugin discovery, and changing or renewing authentication.

So an example of the new workflow is:

```python
from keystoneclient.auth.identity import v3_auth
from keystoneclient import session
from keystoneclient.v3 import client as v3_client
from novaclient.v1_1 import client

auth = v3_auth.Auth(username='username',
                    password='password',
                    project_id='project_id',
                    auth_url='https://keystone.example.com')
client_session = session.ClientSession(auth)

keystone_client = v3_client.Client(client_session)
nova_client = client.Client(client_session)
```

It is obviously a little longer than the current method but I'm sure that the old syntax can be maintained for when you only need a single client.

Implementations of this are starting to go into review on keystoneclient.
For the time being some features from nova such as authentication plugins specifying CLI arguments are not being considered until we can ensure that the new system meets at least the current functionality.

The major problem found so far is maintaining API compatibility.
Much of what is currently on keystoneclient that will be moved is defined publicly and cannot simply be thrown away even though they are typically attributes and abstract functions that a user should have no need of.

Hopefully this or something very similar will be coming to the various OpenStack clients soon.
