---
layout: post
title: "Keystone Token Binding"
date: 2013-10-18 13:52
comments: true
categories: keytone openstack
---

With the release of Havana Keystone gains the ability to issue and verify what we have been calling "bound tokens".
To understand the reason for this feature we need to first consider the security model of the current token architecture.

OpenStack tokens are what we call "bearer tokens".
The term seems to have come out of the oauth movement but means that whoever has the token has all the rights associated with that person (so bearer in a right to bear arms kind of way).
This is not an uncommon situation on the Internet, it is the way username and passwords work, cookies and session ids and one of the reasons that SSL is so important when authenticating against a website.
If an attacker was to get your token then they have all the rights of that token for as long as it is valid, including permission to reissue a token or change your password.

As OpenStack grows and this token is presented to an ever increasing list of services the vulnerability of this mechanism increases.
So what can we do about it?
The typical answer, particularly for the enterprise, is to use Kerberos or x509 client certificates.
This is a great solution but we don't want to have each service dealing with different authentication mechanisms, that's what Keystone does.

##What is a bound token?

They are regular keystone token with some additional information that indicates that the token may only be used in conjunction with the specified external authentication mechanism.
Taking the example of Kerberos, when a token is issued Keystone embeds the name of the Kerberos principle into the token.
When this token is then presented to another service the service notices the bind information and ensures that Kerberos authentication was used and that the same user is making the request.

So how does this help to protect token hijacking?
To give an example:

 1. Alice connects to Keystone using her Kerberos credentials and gets a token.
    Embedded within this token is her Kerberos principal name `alice@ACME.COM`.
 2. Alice authenticates to HaaS (hacked as a service) using her token and Kerberos credentials and is allowed to perform her operations.
 3. Bob, who has privileged access to HaaS, records the token that Alice presented to the service (or otherwise gets Alice's token)
 4. Bob attempts to connect to Keystone as Alice to change her password.
    He connects to keystone with his own Kerberos credentials `bob@ACME.COM`.
    Because these credentials do not match the ones that were present when the token was created his access is disallowed.

It does not necessarily mean that the user initially authenticated themselves by there Kerberos credentials, they may have used there regular username and password.
It simply means that the user who created the token has said that they are also the owner of this Kerberos principal (note: that it is tied to the principal, not a ticket so it will survive ticket re-issuing) and the token should not be authenticated in future without it present.

##What is implemented?

Currently tokens issued from Keystone can be bound to a Kerberos principal.
Extending this mechanism to x509 client certificates should be a fairly simple exercise but will not be included in the Havana release.

A patch to handle bind checking in auth\_token middleware is currently under review to bring checking to other services.

There are however a number of problems with enforcing bound tokens today:

 - Kerberos authentication is not supported by the eventlet http server (the server that drives most of the OpenStack web services), and so there is no way to authenticate to the server to provide the credentials.
   This essentially restricts bind checking to services running in httpd, which to the best of my knowledge is currently only keystone and swift.
 - None of the clients currently support connecting with Kerberos authentication.
   The option was added to Keystoneclient as a proof of concept but I am hoping that this can be solved across all clients by standardizing the way they communicate rather than having to add and maintain support in each individual client.
   There will also be the issue of how to configure the servers to use these clients correctly.
 - Kerberos tickets are issued to users, not hosts, and typically expire after a period of time.
   To allow unattended servers to have valid Kerberos credentials requires a way of automatically refreshing or fetching new tickets.
   I am told that there is support for this scenario coming in Fedora 20 but I am not sure what it will involve.

##Configuring Token Binding

The new argument to enable token binding in `keystone.conf` is:

    [token]

    # External auth mechanisms that should add bind information to token.
    # eg kerberos, x509
    bind = kerberos

As mentioned currently only the value kerberos is currently supported here.


To enable token bind authentication in `keystone.conf` is:

    [token]
    # Enforcement policy on tokens presented to keystone with bind information.
    # One of disabled, permissive, strict, required or a specifically required bind
    # mode e.g. kerberos or x509 to require binding to that authentication.
    enforce_token_bind = permissive

As illustrated by the comments the possible values here are:

 - `disabled`: Disables token bind checking.
 - `permissive`: Token bind information will be verified if present.
    If there is bind information for a token and the server does not know how to verify that information then it will be ignored and the token will be allowed.
    This is the new default value and should have no effect on existing systems.
 - `strict`: Like permissive but if unknown bind information is present then the token will be rejected.
 - `required`: Tokens will only be allowed if bind information is present and verified.
 - A specific form of bind information is present and verified.
   For example the only currently available value here is `kerberos` indicating that a token must be bound to a Kerberos principal to be accepted.

##In Conclusion

For a deployment with access to a Kerberos or x509 infrastructure token binding will dramatically increase your user's security.
Unfortunately the limitations of Kerberos within OpenStack don't really make this a viable deployment option in Havana.
Watch this space however as we add x509 authentication and binding, and improve Kerberos handling throughout.
