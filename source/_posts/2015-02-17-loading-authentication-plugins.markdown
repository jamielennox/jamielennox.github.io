---
layout: post
title: "Loading authentication plugins"
date: 2015-02-17 09:08:03 +1100
comments: true
categories: keystone openstack
---

I've been pushing a lot on the authentication plugins aspect of keystoneclient recently.
They allow us to generalize the process of getting a token from OpenStack such that we can enable new mechanisms like [Kerberos](https://github.com/openstack/python-keystoneclient-kerberos) or client certificate authentication - without having to modify all the clients.

For most people hardcoding credentials into scripts is not an option, both for security and for reusability reasons.
By having a standard loading mechanism for this selection of new plugins we can ensure that applications we write can be used with future plugins.
I am currently working on getting this method into the existing services to allow for more extensible service authentication, so this pattern should become more common in future.

There are two loading mechanisms for authentication plugins provided by keystoneclient:

 - Loading from an [oslo.config](http://docs.openstack.org/developer/oslo.config) CONF object.
 - Loading from an [argparse](https://docs.python.org/2/library/argparse.html) command line

## Loading from CONF

We can define a plugin from CONF like:

```ini
[somegroup]
auth_plugin = v3password
auth_url = http://keystone.test:5000/v3
username = user
password = pass
user_domain_name = domain
project_name = proj
project_domain_name = domain
```

The initially required field here is `auth_plugin` which specifies the name of the plugin to load.
All other parameters in that section are dependant on the information that plugin (in this case v3password) requires.

To load that plugin from an application we do:

{% gist 3b26bfb8e80fa48133e9 test-conf.py %}

Then create `novaclient`, `cinderclient` or whichever client you wish to talk to with that session as normal.

You can also use an `auth_section` parameter to specify a different group in which the authentication credentials are stored.
This allows you to reuse the same credentials in multiple places throughout your configuration file without copying and pasting.

```ini
[somegroup]
auth_section = credentials

[othergroup]
auth_section = credentials

[credentials]
auth_plugin = v3password
auth_url = http://keystone.test:5000/v3
username = user
password = pass
user_domain_name = domain
project_name = proj
project_domain_name = domain
```

The above loading code for `[somegroup]` or `[othergroup]` will load separate instances of the same authentication plugin.


## Loading from the command line

The options present on the command line are very similar to that presented via the config file, and follow a pattern familiar to the existing openstack CLI applications.
The equivalent options as specified in the config above would be presented as:

```sh
./myapp --os-auth-plugin v3password \
        --os-auth-url http://keystone.test:5000/v3 \
        --os-username user \
        --os-password pass \
        --os-user-domain-name domain \
        --os-project-name proj \
        --os-project-domain-name domain
        command
```

Or

```sh
export OS_AUTH_PLUGIN=v3password
export OS_AUTH_URL=http://keystone.test:5000/v3
export OS_USERNAME=user
export OS_PASSWORD=pass
export OS_USER_DOMAIN_NAME=domain
export OS_PROJECT_NAME=proj
export OS_PROJECT_DOMAIN_NAME=domain

./myapp command
```

This is loaded from python via:

{% gist 4e22049c5bc57f4b68ec test-cli.py %}

**NOTE**: I am aware that the syntax is wonky with the command for session loading and auth plugin loading different.
This was one of those things that was 'optimized' between reviews and managed to slip through.
There is a review out to standardize this.

This will also set `--help` appropriately, so if you are unsure of the arguments that this particular authentication plugin takes you can do:

```sh
./myapp --os-auth-plugin v3password --help

usage: myapp [-h] [--os-auth-plugin <name>] [--os-auth-url OS_AUTH_URL]
             [--os-domain-id OS_DOMAIN_ID] [--os-domain-name OS_DOMAIN_NAME]
             [--os-project-id OS_PROJECT_ID]
             [--os-project-name OS_PROJECT_NAME]
             [--os-project-domain-id OS_PROJECT_DOMAIN_ID]
             [--os-project-domain-name OS_PROJECT_DOMAIN_NAME]
             [--os-trust-id OS_TRUST_ID] [--os-user-id OS_USER_ID]
             [--os-user-name OS_USERNAME]
             [--os-user-domain-id OS_USER_DOMAIN_ID]
             [--os-user-domain-name OS_USER_DOMAIN_NAME]
             [--os-password OS_PASSWORD] [--insecure]
             [--os-cacert <ca-certificate>] [--os-cert <certificate>]
             [--os-key <key>] [--timeout <seconds>]

optional arguments:
  -h, --help            show this help message and exit
  --os-auth-plugin <name>
                        The auth plugin to load
  --insecure            Explicitly allow client to perform "insecure" TLS
                        (https) requests. The server's certificate will not be
                        verified against any certificate authorities. This
                        option should be used with caution.
  --os-cacert <ca-certificate>
                        Specify a CA bundle file to use in verifying a TLS
                        (https) server certificate. Defaults to
                        env[OS_CACERT].
  --os-cert <certificate>
                        Defaults to env[OS_CERT].
  --os-key <key>        Defaults to env[OS_KEY].
  --timeout <seconds>   Set request timeout (in seconds).

Authentication Options:
  Options specific to the v3password plugin.

  --os-auth-url OS_AUTH_URL
                        Authentication URL
  --os-domain-id OS_DOMAIN_ID
                        Domain ID to scope to
  --os-domain-name OS_DOMAIN_NAME
                        Domain name to scope to
  --os-project-id OS_PROJECT_ID
                        Project ID to scope to
  --os-project-name OS_PROJECT_NAME
                        Project name to scope to
  --os-project-domain-id OS_PROJECT_DOMAIN_ID
                        Domain ID containing project
  --os-project-domain-name OS_PROJECT_DOMAIN_NAME
                        Domain name containing project
  --os-trust-id OS_TRUST_ID
                        Trust ID
  --os-user-id OS_USER_ID
                        User ID
  --os-user-name OS_USERNAME, --os-username OS_USERNAME
                        Username
  --os-user-domain-id OS_USER_DOMAIN_ID
                        User's domain id
  --os-user-domain-name OS_USER_DOMAIN_NAME
                        User's domain name
  --os-password OS_PASSWORD
                        User's password

```

To prevent polluting your CLI's help only the 'Authentication Options' for the plugin you specified by '--os-auth-plugin' are added to the help.

Having explained all this one of the primary application currently embracing authentication plugins, [openstackclient](https://github.com/openstack/python-openstackclient), currently handles its options slightly differently and you will need to use `--os-auth-type` instead of `--os-auth-plugin`

## Available plugins

The [documentation](http://docs.openstack.org/developer/python-keystoneclient/authentication-plugins.html) for plugins provides basic features and parameters however it's not always going to be up to date with all options, especially for plugins not handled within keystoneclient.
The following is a fairly simple script that lists all the plugins that are installed on the system and their options.

{% gist 7f5cfabd64a6922e643c list-plugins.py %}

Which for the `v3password` plugin we've been using returns:

```
...
v3password:
    auth-url: Authentication URL
    domain-id: Domain ID to scope to
    domain-name: Domain name to scope to
    project-id: Project ID to scope to
    project-name: Project name to scope to
    project-domain-id: Domain ID containing project
    project-domain-name: Domain name containing project
    trust-id: Trust ID
    user-id: User ID
    user-name: Username
    user-domain-id: User's domain id
    user-domain-name: User's domain name
    password: User's password
...
```

From that it's pretty simple to determine the correct format for parameters.

 - When using the CLI you should prefix `--os-`, e.g. `auth-url` becomes `--os-auth-url`.
 - Environment variables are upper-cased, and prefix `OS_` and replace `-` with `_`, e.g. `auth-url` becomes `OS_AUTH_URL`.
 - Conf file variables replace `-` with `_` eg. `auth-url` becomes `auth_url`.
