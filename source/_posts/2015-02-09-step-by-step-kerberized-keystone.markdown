---
layout: post
title: "Step-by-Step: Kerberized Keystone"
date: 2015-02-09 19:48:42 +1100
comments: true
categories: freeipa kerberos openstack
---

Authentication plugins in Keystoneclient have gotten to the point where they are sufficiently well deployed that we can start to do interesting additional forms of authentication.
As Kerberos is a commonly requested authentication mechanism here is a simple, single domain keystone setup using Kerberos authentication.
They are not necessarily how you would setup a production deployment, but should give you the information you need to configure that yourself.

They create:

 - A [FreeIPA](https://www.freeipa.org) server machine called `ipa.test.jamielennox.net`
 - A [Packstack](https://openstack.redhat.com/Quickstart) all in one deployment of OpenStack called `openstack.test.jamielennox.net`

Notes:

 - I use the realm `TEST.JAMIELENNOX.NET`. There is no meaning to this domain, it doesn't exist or make any difference to the deployment.
 - I am using a single domain deployment, so regular users and service users are intermingled.
   There is no great benefit to this because the Kerberos plugin only supports the Keystone v3 API (so you have to be domain aware), however it a couple of steps.
   Generally when doing a deployment like this I would put the IPA users in there own domain.

Disclaimer:

 - The goal of Kerberos authentication is obviously to enable both the command line and horizon interfaces to use your existing Kerberos ticket.
   At the time of writing the [openstackclient](http://docs.openstack.org/developer/python-openstackclient/) is really the only client capable of using plugins.
   I am also [working](https://review.openstack.org/#/c/153910/) [on](https://review.openstack.org/#/c/153174/) the patches required to enable horizon for SSO.
   There are obviously no promises that will be ready and accepted for the Kilo release, but I'm hoping so.

##Part 1 - Install FreeIPA

From a brand new centos 7 image:

{% codeblock lang:console %}
[root@ipa]# yum update -y
[root@ipa]# hostnamectl set-hostname ipa.test.jamielennox.net
{% endcodeblock %}

Now install the FreeIPA server.

{% codeblock lang:console %}
[root@ipa]# yum install ipa-server bind-dyndb-ldap -y
[root@ipa]# ipa-server-install
{% endcodeblock %}

I've ignored the details here, generally having correctly set the hostname the defaults are correct for me, the only thing I generally do is opt to configuring bind for DNS.
This is the reason for installing `bind-dyndb-ldap` and you can skip it if you don't install bind.


Pay particular attention to the username you give to the `admin` user as this user will be used from keystone later.
If you are unfamiliar with FreeIPA I suggest you test out the web interface, this is where your users will be registered.
As FreeIPA expects to be on a routable address you will need to add the hostname to `/etc/hosts` of the client machine to use the web service.

##Part 2 - Register as a FreeIPA Client

The first thing I like to do is register the machine as a FreeIPA client machine.
After registering the machine comes available for us to fetch Kerberos tickets from (and SSL certs, etc).

{% codeblock lang:console %}
[root@openstack]# yum update -y
[root@openstack]# hostnamectl set-hostname openstack.test.jamielennox.net
[root@openstack]# vim /etc/resolv.conf  # and update your DNS server IP of the ipa machine.
[root@openstack]# yum install ipa-client -y
[root@openstack]# ipa-client-install
{% endcodeblock %}

Again, having correctly set a hostname and DNS server the default options are correct for me.
You'll need to provide the admin user and password from setting up the FreeIPA server.

##Part 3 - Install Packstack

Packstack is a series of wrappers around the upstream puppet scripts that integrates well with RHEL/Centos, it will deploy the latest packaged version which is currently Juno.
You could use devstack or any alternative deployment here.

{% codeblock lang:console %}
[root@openstack]# setenforce 0  # :(
[root@openstack]# yum install -y https://rdo.fedorapeople.org/rdo-release.rpm
[root@openstack]# yum install -y openstack-packstack
[root@openstack]# packstack --allinone
{% endcodeblock %}

The `--allinone` option configures the entire OpenStack deployment on this machine, see [the docs](https://openstack.redhat.com/Docs) for more detailed deployments methods.

##Part 4 - Convert Keystone to Apache

To use kerberos authentication the keystone server uses the `mod_auth_kerb` Apache module. We therefore need to run the keystone server within Apache rather than from the script.
Ideally here we would have generated an answer file and told Packstack to use the `httpd` deployment method (which it can do) however this never seems to work for me, so I'll do it manually. YMMV.

Stop the existing keystone server and prevent it starting on boot:

{% codeblock lang:console %}
[root@openstack]# systemctl stop openstack-keystone
[root@openstack]# systemctl disable openstack-keystone
{% endcodeblock %}

There are many ways you could setup keystone within Apache. I don't think the [official docs](http://docs.openstack.org/developer/keystone/apache-httpd.html) explain this very well but you can deploy it as you would any other `mod_wsgi` python site.

{% codeblock lang:console %}
[root@openstack]# mkdir /var/www/keystone
[root@openstack]# ln -s /usr/share/keystone/keystone.wsgi /var/www/keystone/admin
[root@openstack]# ln -s /usr/share/keystone/keystone.wsgi /var/www/keystone/main
{% endcodeblock %}

This is the httpd config file I used initially:

{% codeblock lang:apache %}
Listen 5000
Listen 35357

<VirtualHost *:5000>
  ServerName openstack.test.jamielennox.net

  DocumentRoot "/var/www/keystone"

  <Directory "/var/www/keystone">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    Require all granted
  </Directory>

  ## Logging
  ErrorLog "/var/log/httpd/keystone_wsgi_main_error.log"
  ServerSignature Off
  CustomLog "/var/log/httpd/keystone_wsgi_main_access.log" combined
  WSGIDaemonProcess keystone_main group=keystone processes=1 threads=4 user=keystone
  WSGIProcessGroup keystone_main
  WSGIScriptAlias / "/var/www/keystone/main"
</VirtualHost>

<VirtualHost *:35357>
  ServerName openstack.test.jamielennox.net

  DocumentRoot "/var/www/keystone"

  <Directory "/var/www/keystone">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    Require all granted
  </Directory>

  ## Logging
  ErrorLog "/var/log/httpd/keystone_wsgi_admin_error.log"
  ServerSignature Off
  CustomLog "/var/log/httpd/keystone_wsgi_admin_access.log" combined
  WSGIDaemonProcess keystone_admin group=keystone processes=1 threads=4 user=keystone
  WSGIProcessGroup keystone_admin
  WSGIScriptAlias / "/var/www/keystone/admin"
</VirtualHost>
{% endcodeblock %}

If you restart Apache with this configuration you should see keystone start up and run with no apparent difference.

##Part 5 - Convert Keystone to LDAP:

First we create a keystone user.
This isn't required in a typical deployment, however we need to provide a user with which to connect to the ldap server.

{% codeblock lang:console %}
ipa user-add --first keystone --last openstack --random keystone
{% endcodeblock %}

`--random` sets a random password, you can omit it and set your own if you like.

Edit the `/etc/keystone/keystone.conf` file to use LDAP for the identity backend, and SQL for assignments.
This is the preferred deployment so whilst the users are in LDAP, projects and the permissions a user has on a project are managed in SQL.

{% codeblock lang:ini %}
[identity]
driver = keystone.identity.backends.ldap.Identity

[assignment]
driver = keystone.assignment.backends.sql.Assignment
{% endcodeblock %}

Still in `/etc/keystone/keystone.conf` we set the parameters for how to talk to the LDAP server that FreeIPA set up.
The password used here is the random one generated above.
You will need to replace the pattern `dc=test,dc=jamielennox,dc=net` with the similarly formatted realm that FreeIPA is configured for in your deployment.
This is the complete (comments removed) `[ldap]` section of the `keystone.conf` file.
This plus the `[identity]` block above are also exactly what would go into a [domain specific backend](http://docs.openstack.org/developer/keystone/configuration.html#domain-specific-drivers) configuration if you are deploying that way.

{% codeblock lang:ini %}
[ldap]
url=ldaps://ipa.test.jamielennox.net
user=uid=keystone,cn=users,cn=accounts,dc=test,dc=jamielennox,dc=net
password=D@yFxU6SZHV_
suffix=dc=test,dc=jamielennox,dc=net
user_tree_dn=cn=users,cn=accounts,dc=test,dc=jamielennox,dc=net
user_objectclass=person
user_id_attribute=uid
user_name_attribute=uid
user_mail_attribute=mail
user_allow_create=false
user_allow_update=false
user_allow_delete=false
group_tree_dn=cn=groups,cn=accounts,dc=test,dc=jamielennox,dc=net
group_objectclass=groupOfNames
group_id_attribute=cn
group_name_attribute=cn
group_member_attribute=member
group_desc_attribute=description
group_allow_create=false
group_allow_update=false
group_allow_delete=false
user_enabled_attribute=nsAccountLock
user_enabled_default=False
user_enabled_invert=true
{% endcodeblock %}

Make sure to restart keystone (via apache) to set this configuration.

{% codeblock lang:console %}
[root@openstack]# systemctl restart httpd
{% endcodeblock %}

##Part 6 - Recreate the LDAP users:

This is particularly hacky.
For my single domain deployment when I switch Keystone over to using the LDAP source it will loose access to the users that were set up via Packstack so I will need to recreate these users.
If you are using multiple domains you can skip this, leave the service users in the default domain and use your FreeIPA users via an alternative domain.

The easiest way I've found to do this is to remove the `[general]` tag at the top of the Packstack answers file that was created and then `source` it as bash environment variables.
This answers file is generally found in `/root/` and will be appended with a timestamp.

{% codeblock lang:console %}
[root@openstack]# sed -i -e "s/\[general\]/# [general]/" packstack-answers.txt
[root@openstack]# source packstack-answers.txt
{% endcodeblock %}

Then install the admin tools to allow you to add users via command line and `kinit` as the `admin` user to have permission to add users.

{% codeblock lang:console %}
[root@openstack]# yum install -y ipa-admintools
[root@openstack]# kinit admin
{% endcodeblock %}

Then recreate those users, I'll omit the shell prompt here so that it's easier to copy and paste, but you could script this fairly easily.
We are creating a FreeIPA user for each of the services, setting a password for the user, and then assigning the user a role on the services project.

I'm using the admin token to assign the roles as there are no longer any valid users that I can use to assign these roles.

__NOTE__: This is a fairly unsafe way of creating these users in IPA as they are given full home and login permissions to any IPA machines.
In a real deployment these should be system users and ensure they aren't able to login.

{% codeblock lang:console %}
export OS_ENDPOINT=http://localhost:35357/v2.0
export OS_TOKEN=$CONFIG_KEYSTONE_ADMIN_TOKEN

ipa user-add --first nova --last openstack nova && echo $CONFIG_NOVA_KS_PW | ipa passwd nova
keystone user-role-add --user nova --role admin --tenant services

ipa user-add --first ceilometer --last openstack ceilometer && echo $CONFIG_CEILOMETER_KS_PW | ipa passwd ceilometer
keystone user-role-add --user ceilometer --role admin --tenant services

ipa user-add --first cinder --last openstack cinder && echo $CONFIG_CINDER_KS_PW | ipa passwd cinder
keystone user-role-add --user cinder --role admin --tenant services

ipa user-add --first glance --last openstack glance && echo $CONFIG_GLANCE_KS_PW | ipa passwd glance
keystone user-role-add --user glance --role admin --tenant services

ipa user-add --first neutron --last openstack neutron && echo $CONFIG_NEUTRON_KS_PW | ipa passwd neutron
keystone user-role-add --user neutron --role admin --tenant services

ipa user-add --first swift --last openstack swift && echo $CONFIG_SWIFT_KS_PW | ipa passwd swift
keystone user-role-add --user swift --role admin --tenant services
{% endcodeblock %}

I add the keystone user to the services project as well - though in this case it won't really matter.

{% codeblock lang:console %}
keystone user-role-add --user keystone --role admin --tenant services
{% endcodeblock %}

Create the demo user and add the default roles to admin and demo so the `keystonerc_admin` and `keystonerc_demo` files still work as expected.

{% codeblock lang:console %}
ipa user-add --first demo --last openstack demo && echo $CONFIG_KEYSTONE_DEMO_PW  | ipa passwd demo
keystone user-role-add --user demo --role _member_ --tenant demo

keystone user-role-add --user admin --role admin --tenant admin
{% endcodeblock %}

##Part 7 - Kerberos:

Now we look to add Kerberos authentication.
First we have to register the service that you want to use on the IPA server.
For the current Kerberos plugin this _must_ be the HTTP service or the plugin won't catch it.
Then we fetch the keytab and put it somewhere apache can access it.

{% codeblock lang:console %}
[root@openstack]# ipa service-add HTTP/openstack.test.jamielennox.net@TEST.JAMIELENNOX.NET
[root@openstack]# ipa-getkeytab -s ipa.test.jamielennox.net -p HTTP/openstack.test.jamielennox.net@TEST.JAMIELENNOX.NET -k /etc/httpd/conf/httpd.keytab
[root@openstack]# chown apache: /etc/httpd/conf/httpd.keytab
{% endcodeblock %}

Install the `mod_auth_kerb` apache module which handles Kerberos Authentication, and enable it.
(Generally you don't have to enable module, this looks like a pattern that packstack has established)

{% codeblock lang:console %}
[root@openstack]# yum install -y mod_auth_kerb
[root@openstack]# ln -s /etc/httpd/conf.modules.d/10-auth_kerb.conf /etc/httpd/conf.d/10-auth_kerb.load
{% endcodeblock %}

Add the following snippet to _both_ the vhosts we created earlier, adjusting `/main` to `/admin` for the admin port.
The `WSGIScriptAlias / "/var/www/keystone/main"` is already present from the initial setup, this just shows that you need to insert the `/krb` reference _above_ it to make it take precedence.

This is creating a new route to the Keystone server code through which every access is Kerberos protected.
We will use this address later as our authentication URL.

{% codeblock lang:apache %}
WSGIScriptAlias /krb "/var/www/keystone/main"
WSGIScriptAlias / "/var/www/keystone/main"

<Location "/krb">
      LogLevel debug
      AuthType Kerberos
      AuthName "Kerberos Login"
      KrbMethodNegotiate on
      KrbMethodK5Passwd off
      KrbServiceName HTTP
      KrbAuthRealms TEST.JAMIELENNOX.NET
      Krb5KeyTab /etc/httpd/conf/httpd.keytab
      KrbLocalUserMapping on
      Require valid-user
      SetEnv REMOTE_DOMAIN Default
</Location>
{% endcodeblock %}

The SetEnv directive there is not required as we are deploying into the `Default` domain, however if you are deploying into a different domain you should set `REMOTE_DOMAIN` to the _name_ of the user's domain.

Restart Apache.

{% codeblock lang:console %}
[root@openstack]# systemctl restart httpd
{% endcodeblock %}

##Part 8 - Client:

Obviously you don't have to run the client on the same machine as the server.
You can start up another machine here and do an `ipa-client-install` to initialize it for testing if you like, or put it on your own desktop (you can add your kerberos server to `/etc/krb5.conf` to not register it as a FreeIPA client).

Install the openstack command line tool

{% codeblock lang:console %}
[root@openstack]# yum install -y python-openstackclient
{% endcodeblock %}

We also want to install the Kerberos client plugin from source.
This plugin is imminently due for its first release on PyPI however it will take a while to get to packages.

{% codeblock lang:console %}
[root@openstack]# yum groupinstall "Development Tools"
[root@openstack]# yum install -y krb5-devel
[root@openstack]# git clone https://github.com/openstack/python-keystoneclient-kerberos.git /opt/python-keystoneclient-kerberos
[root@openstack]# pip install -e /opt/python-keystoneclient-kerberos
{% endcodeblock %}

Now `kinit` as the demo user to get a Kerberos ticket to test with.

{% codeblock lang:console %}
[root@openstack]# su centos  # or another user
[centos@openstack]$ echo $CONFIG_KEYSTONE_DEMO_PW | kinit demo
{% endcodeblock %}

Finally, for the dramatic finale, let's get a token.
I'm putting the authentication parameters on the command line however they all exist as environment variables to create your own `keystonerc` file.

{% codeblock lang:console %}
[centos@openstack]$ export OS_AUTH_URL=http://openstack.test.jamielennox.net:5000/krb/v3
[centos@openstack]$ export OS_AUTH_TYPE=v3kerberos
[centos@openstack]$ export OS_PROJECT_NAME=demo
[centos@openstack]$ export OS_PROJECT_DOMAIN_ID=default
[centos@openstack]$ openstack token issue
{% endcodeblock %}

Notice the main authentication differences (and I'll use the CLI equivalent for demonstration):

 - `--os-auth-url http://openstack.test.jamielennox.net:5000/krb/v3` we need to specify the v3 `auth-url` and include the `/krb` path so that the authentication request goes to the Kerberos route.
 - `--os-auth-type v3kerberos` which tells the CLI to use the kerberos plugin appropriate for the v3 API.
 - `--os-project-domain-id` because we are in the V3 API now so you need to specify the domain-id if you use `--os-project-name`

# Finish

Getting to the point where we can support the `openstack` command line tool has taken a lot of foundational work, and due to the independence of the clients it may be some time before the existing CLIs come to accept an authentication plugin parameter.

# Thanks

Special thanks have to go to:

 - [Jose Castro Leon](https://twitter.com/josecastroleon) and [Marek Dennis](https://twitter.com/marekdenis) from CERN who did the initial Kerberos implementation and have been pushing the approach.
 - [Nathan Kinder](https://blog-nkinder.rhcloud.com/) who [scripted a setup](https://github.com/nkinder/rdo-vm-factory/blob/master/rdo-kerberos-setup/vm-post-cloud-init-rdo.sh) slightly more advanced and production worth that I cheated off for a number of settings.
