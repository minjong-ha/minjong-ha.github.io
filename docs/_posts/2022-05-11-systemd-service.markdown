---
layout: posts
title:  "systemd: How can I manage the services in Linux?""
author: Minjong Ha
published: false
date:   2022-05-11 19:38:22 +0900
---

In this post, I share what I learned about systemd.

"systemd" is a suite of basic building blocks for a Linux system[Wiki](https://www.freedesktop.org/wiki/Software/systemd/).
It starts the services / daemons, keeps track of precesses with control groups, maintains mount/unmount points, and provide dependency-based service control logic.
"systemd" involves the Linux system widely, but I focus the starting services and dependency-based service control logics in this post.

## Backgrounds

<!-- about services... what are they and where we use them-->
"systemd" always has PID 1.
When the system booted, the services start following the orders with parallelization by the systemd.

There are two directories for systemd services: "/lib/systemd/" and "/etc/systemd/".
There is no strict rules that deciding which service should be where; it is a policy.
The services in "/lib/systemd/" path are usually installed by default package manager (i.e. apt in Debians, yum in CentOS).
On the other hand, "/etc/systemd/" path generally has manually installed, modified services and symbolic links of systemd services.
If there are two service files having different contents with same title, the service in "/etc/systemd/" path has precedence.

Also there are "/system" and "/user" directories under the "/lib/systemd/" and "/etc/systemd".
The services under the "/system" directory are system services.
They starts when the system booted, and follow the order that systemd pre-defined.
For example, libvirtd-related services include multi-user.target in their service files (I will explain more details about it in later section).
It starts when the Linux enters in multi-user.target; it is usually considered as CLI based with tty.
On the other hand, gnome-related services include graphical.target in their service files.
It starts when the GUI is provided by the Linux.
Thus, gnome-related services never start before the libvirtd-related services since services belong to graphical.target start after the services belong to multi-user.target start (multi-user.target -> graphical.target).


## ".service" Files

<!-- Details about service.file-->

### 1. Unit


### 2. Service


### 3. Install


## Appendix

### systemd-analyze
<!-- systemd-analyize critical-chain -->

### What about starting service after the other service finished?
<!-- about FDE: dev-mapper-extra2.device file case-->

### "override.conf"




