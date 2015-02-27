---
layout: post
title: "Setting up s4u2proxy"
date: 2015-02-27 18:48:11 +1100
comments: true
categories: freeipa kerberos
---

## Motivation:

Kerberos authentication provides a good experience for allowing users to connect to a service.
However this authentication does not allow the user to take the received ticket and further communicate with another service.

The canonical example of this is when authenticating to a web service we want to use the same user credentials to authenticate with an LDAP service, rather than require credentials for the service itself.

In my specific case if I have a [kerberized keystone](http://www.jamielennox.net/blog/2015/02/12/step-by-step-kerberized-keystone/) then when the user talks to Horizon I want to forward the user's ticket to authenticate with keystone.

The mechanism that allows us to forward these Kerberos tickets is called Service-for-User-to-Proxy or S4U2Proxy.
To mitigate some of the security issues with delegating user tickets there are strict controls over which services are allowed to forward tickets and to whom which have to be configured.

For a more in-depth explanation check out the [further reading](#furtherreading) section at the end of this post.

## Scenario:

I intend this guide to be a step by step tutorial into setting up a basic S4U2 proxying service that we can verify and give you enough information to go about setting up more complex delegations.
If you are just looking for the raw commands you can jump down to [Setting up the Delegation](#delegation).

I created 3 Centos 7 virtual machines on a private network:

 - An IPA server at `ipa.s4u2.jamielennox.net`
 - A service provider at `service.s4u2.jamielennox.net` that will provide the target service.
 - An S4U2 proxy service at `proxy.s4u2.jamielennox.net` that will accept a Kerberos ticket and forward it to `service.s4u2.jamielennox.net`

For this setup I am creating a testing realm called `S4U2.JAMIELENNOX.NET`.
I will post the setup that works for my environment and leave it up to you to recognize where you should use your own service names.

## Setting up IPA

```
hostnamectl set-hostname ipa.s4u2.jamielennox.net
yum install -y ipa-server bind-dyndb-ldap
ipa-server-install
```

I pick the option to enable DNS as I think it's easier, you can skip that but then you'll need to make `/etc/hosts` entries for each of the hosts.


## Setting up the Service

We start by doing the basic configuration of the machine and setting it up as an IPA client machine.

```sh
hostnamectl set-hostname service.s4u2.jamielennox.net
yum install -y ipa-client
vim /etc/resolv.conf  # set DNS server to IPA IP address
ipa-client-install
yum install -y httpd php mod_auth_kerb
rm /etc/httpd/conf.d/welcome.conf  # a stub page that gets in the way
```

Register that we will be exposing a HTTP service on the machine:

```sh
yum install -y ipa-admintools
kinit admin
ipa service-add HTTP/service.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET
```

Fetch the Kerberos keytab from IPA and make it accessible to Apache:

```sh
ipa-getkeytab -s ipa.s4u2.jamielennox.net -p HTTP/service.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET -k /etc/httpd/conf/httpd.keytab
chown apache: /etc/httpd/conf/httpd.keytab
```

Create a simple site that will display the environment variables the server has received.
I share most people's opinion of PHP, however for a simple diagnostic site it's hard to beat `phpinfo()`:

```
mkdir /var/www/s4u2
echo "<?php phpinfo(); ?>" > /var/www/s4u2/index.php
```

Configure Apache to serve our simple PHP site behind Kerberos authentication.

{% codeblock lang:apache /etc/httpd/conf.d/s4u2-service.conf %}
<VirtualHost *:80>
  ServerName service.s4u2.jamielennox.net

  DocumentRoot "/var/www/s4u2"

  <Directory "/var/www/s4u2">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    Require all granted
  </Directory>

  <Location "/">
    AuthType Kerberos
    AuthName "Kerberos Login"
    KrbMethodNegotiate on
    KrbMethodK5Passwd off
    KrbServiceName HTTP
    KrbAuthRealms S4U2.JAMIELENNOX.NET
    Krb5KeyTab /etc/httpd/conf/httpd.keytab
    KrbSaveCredentials on
    KrbLocalUserMapping on
    Require valid-user
  </Location>

  DirectoryIndex index.php
</VirtualHost>
{% endcodeblock %}

Finally restart Apache to bring up the service site:

```sh
systemctl restart httpd
```

## Setting up my local machine

You could easily test all this using curl, however particularly as we are setting up HTTP to HTTP delegation the obvious use is going to be via the browser, so at this point I like to configure firefox to allow Kerberos negotiation.

I don't want my development machine to be an IPA client so I just configure the Kerberos KDC so that I can get a ticket on my machine with `kinit`.

Edit `/etc/krb5.conf` to add:

{% codeblock lang:ini /etc/krb5.conf %}
[realms]
 S4U2.JAMIELENNOX.NET = {
  kdc = ipa.s4u2.jamielennox.net
  admin_server = ipa.s4u2.jamielennox.net
 }

[domain_realms]
 .s4u2.jamielennox.net = S4U2.JAMIELENNOX.NET
 s4u2.jamielennox.net = S4U2.JAMIELENNOX.NET
{% endcodeblock %}

And because I don't want to rely on the DNS provided by this IPA server I'll need to add the service IPs to `/etc/hosts`:

{% codeblock lang:text /etc/hosts %}
10.16.19.24     service.s4u2.jamielennox.net
10.16.19.100    proxy.s4u2.jamielennox.net
10.16.19.101    ipa.s4u2.jamielennox.net
{% endcodeblock %}

In firefox open the config page (type `about:config` into the URL bar) and set:
```ini
network.negotiate-auth.delegation-uris = .s4u2.jamielennox.net
network.negotiate-auth.trusted-uris = .s4u2.jamielennox.net
```
These are comma seperated values so you can configure this in addition to any existing realms you might have configured.

To test get a ticket:
```sh
kinit admin@S4U2.JAMIELENNOX.NET
```

I can now point firefox to `http://service.s4u2.jamielennox.net` and we see the `phpinfo()` dump of environment variables.
This means we have successfully set up our service host.

Interesting environment variables to check for to ensure this is correct are:

 * `REMOTE_USER admin` shows that the ticket belonged to the admin user.
 * `AUTH_TYPE Negotiate` indicates that the user was authenticated via the Keberos mechanism.

## Create Proxy Service

When you register the service you have to mark it as allowed to delegate credentials.
You can do this anywhere you have an admin ticket or via the web UI, however there's less options to provide if you use one of the ipa client machines.

```sh
ipa service-add HTTP/proxy.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET --ok-as-delegate=true
```
or to modify an existing service:
```sh
ipa service-mod HTTP/proxy.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET --ok-as-delegate=true
```

## <a name="delegation"></a>Setting up the Delegation

Unfortunately FreeIPA has no way to manage S4U2 delegations via the command line or GUI yet and so we must resort to editing LDAP directly.
The s4u2 access permissions are defined from a group of services (`groupOfPrincipals`) onto a group of services.

You can see existing delegations via:

```
ldapsearch -Y GSSAPI -H ldap://ipa.s4u2.jamielennox.net -b "cn=s4u2proxy,cn=etc,dc=s4u2,dc=jamielennox,dc=net" "" "*"
```

This delegation is how the FreeIPA web service is able to use the user's credentials to read and write from the LDAP server so there is at least 1 existing rule that you can copy from.

A delegation consists of two parts:

 - A target group with a list of services (`memberPrincipal`) that are allowed to receive delegated credentials.
 - A group (type `objectclass=ipaKrb5DelegationACL`) with a list of services (`memberPrincipal`) that are allowed to delegate credentials _AND_ the target groups (`ipaAllowedTarget`) that they can delegate to.

{% codeblock lang:text delegate.ldif  %}
# test-http-delegation-targets, s4u2proxy, etc, s4u2.jamielennox.net
dn: cn=test-http-delegation-targets,cn=s4u2proxy,cn=etc,dc=s4u2,dc=jamielennox,dc=net
objectClass: groupOfPrincipals
objectClass: top
cn: test-http-delegation-targets
memberPrincipal: HTTP/service.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET

# test-http-delegation, s4u2proxy, etc, s4u2.jamielennox.net
dn: cn=test-http-delegation,cn=s4u2proxy,cn=etc,dc=s4u2,dc=jamielennox,dc=net
objectClass: ipaKrb5DelegationACL
objectClass: groupOfPrincipals
objectClass: top
cn: test-http-delegation
memberPrincipal: HTTP/proxy.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET
ipaAllowedTarget: cn=test-http-delegation-targets,cn=s4u2proxy,cn=etc,dc=s4u2,dc=jamielennox,dc=net
{% endcodeblock %}

Write it to LDAP:

```sh
ldapmodify -a -H ldaps://ipa.s4u2.jamielennox.net -Y GSSAPI -f delegate.ldif
```

And that's the hard work done, the `HTTP/proxy.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET` service now has permission to delegate a received ticket to `HTTP/service.s4u2.jamielennox.net@S4U2.JAMIELENNOX.NET`.

## Proxy

Registering the proxy machine is very similar.

```sh
hostnamectl set-hostname proxy.s4u2.jamielennox.net
yum install -y ipa-client
vim /etc/resolv.conf  # set DNS server to IPA IP address
setenforce 0
```
Because the easiest way I know to test a Kerberos endpoint is with curl I am also going to write the proxy service directly in bash:

{% codeblock lang:bash /var/www/s4u2/index.sh %}
#!/bin/sh

echo "Content-Type: text/html; charset=UTF-8"
echo ""
echo ""

# simply dump the information from the service page
curl -s --negotiate -u :  http://service.s4u2.jamielennox.net
{% endcodeblock %}

This works because the cgi-bin sets the request environment into the shell environment, so `$KRB5CCNAME` is set.
If you are using mod\_wsgi or other then you would have to set that into your shell environment before executing any Kerberos commands.

I'm going to skip the IPA client setup and fetching the keytab - this is required and done exactly the same as for the service.

The apache configuration for the proxy is very similar to the configuration of the service except we add:

```apache
KrbConstrainedDelegation on
```

Within the apache vhost config file to enable it to delegate a Kerberos credential.

The final config file looks like:

{% codeblock lang:apache /etc/httpd/conf.d/s4u2-proxy.conf %}
<VirtualHost *:80>
  ServerName proxy.s4u2.jamielennox.net

  DocumentRoot "/var/www/s4u2"

  <Directory "/var/www/s4u2">
    Options Indexes FollowSymLinks MultiViews ExecCGI
    AllowOverride None
    AddHandler cgi-script .sh
    Require all granted
  </Directory>

  <Location "/">
    AuthType Kerberos
    AuthName "Kerberos Login"
    KrbMethodNegotiate on
    KrbMethodK5Passwd off
    KrbServiceName HTTP
    KrbAuthRealms S4U2.JAMIELENNOX.NET
    Krb5KeyTab /etc/httpd/conf/httpd.keytab
    KrbSaveCredentials on
    KrbLocalUserMapping on
    Require valid-user
    KrbConstrainedDelegation on
  </Location>

  DirectoryIndex index.sh
</VirtualHost>
{% endcodeblock %}

Restart apache to have your changes take effect:

```sh
systemctl restart httpd
```

## Voila

After all that aiming firefox at `http://proxy.s4u2.jamielennox.net` gives me the same `phpinfo` page I got from when I talked to the service host directly.
You can verify from this site also that the `SERVER\_NAME service.s4u2.jamielennox.net` and that `REMOTE_USER` is `admin`.

## <a name="furtherreading"></a>Further Reading

There are a couple of sites that this guide is based on:

 * [Adam Young](http://adam.younglogic.com/2014/05/s4u2proxy-horizon/) - who initially prototyped a lot of the work for horizon which we hope to have ready soon.
 * [Alexander Bokovoy](https://vda.li/en/posts/2013/07/29/Setting-up-S4U2Proxy-with-FreeIPA/) - who is the actual authority that Adam and I are relying upon.
 * [Simo Sorce](https://ssimo.org/blog/id_011.html) - explaining the rationale and uses for the S4U2 delegation mechanisms.
