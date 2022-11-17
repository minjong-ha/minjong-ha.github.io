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

<!-- some description about systemd -->
<!-- overview? -->


## ".service" Files

<!-- Details about service.file-->

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

"systemd" service has three state: inactive, activating, and active.
"inactive" represents that the service is not executed.
"systemd" runs the services following the relations between of them, and make their state to "activating".
If the service satisfies the active contidion, its state becomes "active".

Above is the part of "libvirtd.service" file.
There are three sections in ".service" file: Unit, Service, and Install.

```
# libvirtd.service

[Unit]
Description=Virtualization daemon
Requires=virtlogd.socket
...
After=xencommons.service
Conflicts=xendomains.service
...

[Service]
Type=notify
ExecStart=/usr/sbin/libvirtd $LIBVIRTD_ARGS
...

[Install]
WantedBy=multi-user.target
Also=virtlockd.socket
...
```


### 1. Unit
Unit section defines the relations with other services.
Suppose there is a service "A":

- Description
> * Short describes what service it is.

- Requires
> * The services belong to "Requires" should be active to activate service "A".

- Wants
> * When the service "A" executes, the services belong to "Wants" also be executed. Unlike "Requires", "A" can be active even if the service in "Wants" fails to run.

- BindsTo:
> * If the services in "BindsTo" become inactive, "A" also be inactive.

- PartOf:
> * If the services in "PartOf" try to restart or inactive, "A" follows their action.

- Before:
> * Services in "Before" will not be executed until the "A" executed == Service "A" should be executed before the services in "Before".

- After:
> * Services in "After" will be executed first then "A" == Service "A" wait its execution after the services in "After".

- Conflicts:
> * Services in "Conflicts" will not executed at the same time with "A". If "A" is active, "Confilicts" will be inactive, and vice versa.


### 2. Service

- Type = simple | forking | oneshot | dbus | notify
> * "Type" represents the conditions for active. If the service satisfies the condition in "Type", its state changes to active from activating.
>> * simple: Executing the service is considered as active.
>> * forking: When the child process of the service is created, it is considred as active.
>> * oneshot: Executing the service is considered as active, but it is considered that the main process of the service exits.
>> * dbus: When the service acquires the dbus it defined, it is considered as active.
>> * notify: When the service sends message through sd_notify, it is considered as active.


### 3. Install


## Appendix

### systemd-analyze
<!-- systemd-analyize critical-chain -->

### What about starting service after the other service finished?
<!-- about FDE: dev-mapper-extra2.device file case-->

### "override.conf"




