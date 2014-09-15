---
layout: post
title: "How to use keystoneclient Sessions"
date: 2014-09-15 09:13:36 +1000
comments: true
categories: keystone openstack
---

In the last post I did on keystoneclient sessions there was a lot of hand waving about how they should work but it's not merged yet.
Standardizing clients has received some more attention again recently - and now that the sessions are more mature and ready it seems like a good opportunity to explain them and how to use them again.

For those of you new to this area the clients have grown very organically, generally forking off some existing client and adding and removing features in ways that worked for that project.
Whilst this is in general a problem for user experience (try to get one token and use it with multiple clients without reauthenticating) it is a nightmare for security fixes and new features as they need to be applied individually across each client.

Sessions are an attempt to extract a common authentication and communication layer from the existing clients so that we can handle transport security once, and keystone and deployments can add new authentication mechanisms without having to do it for every client.

The Basics
----------

Sessions and authentications are user facing objects that you create and pass to a client, they are public objects not a framework for the existing clients.
They require a change in how you instantiate clients.

The first step is to create an authentication plugin, currently the available plugins are:

- ``keystoneclient.auth.identity.v2.Password``
- ``keystoneclient.auth.identity.v2.Token``
- ``keystoneclient.auth.identity.v3.Password``
- ``keystoneclient.auth.identity.v3.Token``
- ``keystoneclient.auth.token_endpoint.Token``

For the primary user/password and token authentication mechanisms that keystone supports in v2 and v3 and for the test case where you know the endpoint and token in advance.
The parameters will vary depending upon what is required to authenticate with each.

Plugins don't need to live in the keystoneclient, we are currently in the process of setting up a new repository for kerberos authentication so that it will be an optional dependency.
There are also some plugins living in the contrib section of keystoneclient for federation that will also likely be moved to a new repository soon.

You can then create a session with that plugin.

```python

from keystoneclient import session as ksc_session
from keystoneclient.auth.identity import v3
from keystoneclient.v3 import client as keystone_v3
from novaclient.v1_1 import client as nova_v2

auth = v3.Password(auth_url='http://keystone.host/v3',
                   username='user',
                   password='password',
                   project_name='demo',
                   user_domain_name='default',
                   project_domain_name='default')

session = ksc_session.Session(auth=auth,
                              verify='/path/to/ca.cert')

keystone = keystone_v3.Client(session=session)
nova = nova_v2.Client(session=session)
```

Keystone and nova clients will now share an authentication token fetched with keystone's v3 authentication.
The clients will authenticate on the first request and will re-authenticate automatically when the token expires.

This is a fundamental shift from the existing clients that would authenticate internally to the client and on creation so by opting to use sessions you are acknowledging that some methods won't work like they used to.
For example keystoneclient had an ```authenticate()``` function that would save the details of the authentication (user\_id etc) on the client object.
This process is no longer controlled by keystoneclient and so this function should not be used, however it also cannot be removed because we need to remain backwards compatible with existing client code.

In converting the existing clients we consider that passing a Session means that you are acknowledging that you are using new code and are opting-in to the new behaviour.
This will not affect 90% of users who just make calls to the APIs, however if you have got hacks in place to share tokens between the existing clients or you overwrite variables on the clients to force different behaviours then these will probably be broken.

Per-Client Authentication
-------------------------

The above flow is useful for users where they want to have there one token shared between one or more clients.
If you are are an application that uses many authentication plugins (eg, heat or horizon) you may want to take advantage of using a single session's connection pooling or caching whilst juggling multiple authentications.
You can therefore create a session without an authentication plugin and specify the plugin that will be used with that client instance, for example:

```python
global SESSION

if not SESSION:
    SESSION = ksc_session.Session()

auth = get_auth_plugin()  # you could deserialize it from a db,
                          # fetch it based on a cookie value...
keystone = keystone_v3.Client(session=SESSION, auth=auth)
```

Auth plugins set on the client will override any auth plugin set on the session - but I'd recommend you pick one method based on your application's needs and stick with it.

Loading from a config file
--------------------------

There is support for loading session and authentication plugins from and oslo.config CONF object.
The documentation on exactly what options are supported is lacking right now and you will probably need to look at code to figure out everything that is supported.
I promise to improve this, but to get you started you need to register the options globally:

```python
group = 'keystoneclient'  # the option group
keystoneclient.session.Session.register_conf_options(CONF, group)
keystoneclient.auth.register_conf_options(CONF, group)
```

And then load the objects where you need them:

```python
auth = keystoneclient.auth.load_from_conf_options(CONF, group)
session = ksc_session.Session.load_from_conf_options(CONF, group, auth=auth)
keystone = keystone_v3.Client(session=session)
```

Will load options that look like:

```ini
[keystoneclient]
cacert = /path/to/ca.cert
auth_plugin = v3password
username = user
password = password
project_name = demo
project_domain_name = default
user_domain_name = default
```

There is also support for transitioning existing code bases to new option names if they are not the same as what your application uses.

Loading from CLI
----------------

A very similar process is used to load sessions and plugins from an argparse parser.

```python

parser = argparse.ArgumentParser('test')

argv = sys.argv[1:]

keystoneclient.session.Session.register_cli_options(parser)
keystoneclient.auth.register_argparse_arguments(parser, argv)

args = parser.parse_args(argv)

auth = keystoneclient.auth.load_from_argparse_arguments(args)
session = keystoneclient.session.Session.load_from_cli_options(args,
                                                               auth=auth)
```

This produces an application with the following options:

```bash
python test.py --os-auth-plugin v3password
usage: test [-h] [--insecure] [--os-cacert <ca-certificate>]
            [--os-cert <certificate>] [--os-key <key>] [--timeout <seconds>]
            [--os-auth-plugin <name>] [--os-auth-url OS_AUTH_URL]
            [--os-domain-id OS_DOMAIN_ID] [--os-domain-name OS_DOMAIN_NAME]
            [--os-project-id OS_PROJECT_ID]
            [--os-project-name OS_PROJECT_NAME]
            [--os-project-domain-id OS_PROJECT_DOMAIN_ID]
            [--os-project-domain-name OS_PROJECT_DOMAIN_NAME]
            [--os-trust-id OS_TRUST_ID] [--os-user-id OS_USER_ID]
            [--os-user-name OS_USERNAME]
            [--os-user-domain-id OS_USER_DOMAIN_ID]
            [--os-user-domain-name OS_USER_DOMAIN_NAME]
            [--os-password OS_PASSWORD]
```

There is an ongoing effort to create a standardized CLI plugin that can be used by new clients rather than have people provide an --os-auth-plugin every time.
It is not yet ready, however clients can create and specify there own default plugins if --os-auth-plugin is not provided.


For Client Authors
------------------

To make use of the session in your client there is the ```keystoneclient.adapter.Adapter``` which provides you with a set of standard variables that your client should take and use with the session.
The adapter will handle the per-client authentication plugins, handle ```region_name```, ```interface```, ```user_agent``` and similar client parameters that are not part of the more global (across many clients) state that sessions hold.

The basic client should look like:

```
class MyClient(object):

    def __init__(self, **kwargs):
        kwargs.set_default('user_agent', 'python-myclient')
        kwargs.set_default('service_type', 'my')
        self.http = keystoneclient.adapter.Adapter(**kwargs)
```

The adapter then has ```.get()``` and ```.post()``` and other http methods that the clients expect.

Conclusion
----------

It's great to have renewed interest in standardizing client behaviour, and I'm thrilled to see better session adoption.
The code has matured to the point it is usable and simplifies use for both users and client authors.

In writing this I kept wanting to link out to official documentation and realized just how lacking it really is.
Some explanation is available on the [official python-keystoneclient docs pages](http://docs.openstack.org/developer/python-keystoneclient/using-sessions.html), there is also [module documentation](http://docs.openstack.org/developer/python-keystoneclient/api/keystoneclient.auth.identity.html) however this is definetly an area in which we (read I) am a long way behind.
