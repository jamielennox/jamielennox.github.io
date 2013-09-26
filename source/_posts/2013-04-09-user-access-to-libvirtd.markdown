---
layout: post
title: "User access to libvirtd"
date: 2013-04-09 11:50
comments: true
categories: fedora openstack
---

To enable access to libvirtd without sudo:

1. Create a group for privileged users, I called mine `libvirt` and add your users to it.
1. Create a new file in `/etc/polkit-1/rules.d/` i called mine `50-libvirt-group.rules`
1. Add the following function:
```
polkit.addRule(function(action, subject) {
    if (action.id == "org.libvirt.unix.manage" &&
        subject.isInGroup('libvirt') ) {
        return polkit.Result.YES;
    }
});
```

