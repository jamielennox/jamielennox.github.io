---
layout: post
title: "Token Scoping"
date: 2014-08-08 13:41
comments: true
categories: keystone openstack
---

It has been mentioned to me a number of times now that people don't always understand the difference between projects and domains and generally how the token scoping process works within OpenStack.
I'm sure others have tried to provide explanations but this is my attempt.

Domains, Tenants and Projects
-----------------------------

Firstly, if you are having trouble between V2 and V3 of the identity API terminology what was called a tenant in V2 is now called a project in V3 and you will hear people use the terms interchangeably.
I wasn't involved in keystone at the time of that decision but my rationale for the (fairly disruptive) change is that the V3 API brings multitenancy to OpenStack and the term tenant in v2 does not mean the same as the multi-tenancy concept.

Looking at [wikipedia](https://en.wikipedia.org/wiki/Multitenancy):

> Multitenancy refers to a principle in software architecture where a single instance of the software runs on a server, serving multiple client-organizations (tenants). Multitenancy contrasts with multi-instance architectures where separate software instances (or hardware systems) operate on behalf of different client organizations. With a multitenant architecture, a software application is designed to virtually partition its data and configuration, and each client organization works with a customized virtual application.

Multitenancy, the ability to virtually partition OpenStack, is required to allow resellers to segregate there cloud for there customers rather than running a new cloud deployment for every customer.
The often quote example is that you should be able to support Coke and Pepsi on the same resources and be assured of the same level of privacy as if they were each on there own cloud.

To provide this isolation we created domains.
Users and projects all belong to a domain.
Administrators of a domain can handle their own users and create and destroy projects however they have no privileges outside of that domain.
So a Coke domain administrator should not be even able to determine if Pepsi is in the same cloud let alone access information from them.

As you can hopefully see this pattern doesn't reflect what V2 calls tenants at all and persisting the term tenants to the V3 API so that tenants exist in a domain is really bad terminology.
So tenants got renamed to projects, domains provide the concept of multitenancy, and everyone involved with keystone tries their best to remove the word "tenant" from the vocabulary.

Now when migrating between V2 and the V3 API almost everything that the V2 API provides in terms of creating projects, managing users etc all maps neatly into a single domain.
We therefore have a default domain and everything that happens against the V2 API is implicitly happening within the default domain.

Token Scoping
-------------

Roles are typically assigned to a user on a specific project so that they can have different roles on different projects.
For example a user might have permission to spin up and destroy instances on the development project, but not on the production project.

When you get a token for a project that token identifies the project you are operating within and contains all the roles and information that keystone knows about for that user on that project.
Because of this that token may only be used when accessing resources within that project.
If a user wishes to perform operations within a different project they must fetch a new token for that project.
So if the user presents a token for 'Project A' when trying to perform an operation within 'Project B' the request will be rejected even if the user has the required roles on the other project.
Because of this relationship we call them "project scoped tokens".

With the V2 API when you wanted to perform operations that were on the domain (like create a project) we checked for a very coarse 'admin' role that a user would have on any project.
In V3 this maps into having that same role on the domain.
Therefore to perform operations related to the domain we need to fetch a "domain scoped token" which provides the user's roles and information on that domain.

These are currently the only two things that a token can be scoped to in OpenStack.
Because the domain is something we can scope to and manage independently we no longer need to have those global roles and so roles on the domain do not automatically give you any roles within the projects in that domain.
It also means that to be used a role must be provided to a user on either a domain or a project, the concept of globally assigning a user to a specific role does not make sense.

The last concept regarding scoping is what is referred to as the "unscoped token" which are not scoped to anything and would not be allowed to perform any operations on regular services.
They are useful only within keystone to allow a user to login and list the projects or domains that they can receive a scoped token.

What does this all mean to other services?
------------------------------------------

So the primary reason for me to try and explain these concepts is that it seems to confuse the other OpenStack services as to what they have to do for these tokens.
In theory the answer is nothing, in practice we still have not managed to make V3 the default API.
The reasons for this are more technical than this post but essentially come down to the fact that between the V2 and V3 API the token format was changed and there were lots of services that relied on this format.
It does not have anything to do with the introduction of domains or the renaming of tenants.

The minimum unit that OpenStack services should work on is the project.
All resources in OpenStack are associated with a project and a user's access to those resources is restricted by the user's roles in that project.

So an unscoped token which has no roles is only usable by keystone to inform the user of there permissions.
It will be rejected by all other services.

A domain scoped token is used for managing users and projects - concepts which are completely managed within keystone.
The OpenStack services that are providing access to resources within a project are allowed to be completely unaware of concepts happening outside of a project.
They will therefore reject domain scoped tokens.
