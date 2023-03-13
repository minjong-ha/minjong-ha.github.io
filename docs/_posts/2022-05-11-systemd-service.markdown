---
layout: posts
title:  "systemd: How can I manage the services in Linux?"
author: Minjong Ha
published: true
date:   2022-05-11 19:38:22 +0900
---

## Introduction

In this post, I share what I learned about systemd.

"systemd" is a suite of basic building blocks for a Linux system[Wiki](https://www.freedesktop.org/wiki/Software/systemd/).
It starts the services / daemons, keeps track of precesses with control groups, maintains mount/unmount points, and provide dependency-based service control logic.
"systemd" involves to the Linux system widely, but I focus on the launching services and dependency-based service control logics in this post.

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

```bash
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

Above codes represent the contents of the service file.

### 1. Unit

Unit section defines the relations with other services.
Suppose there is a service "A":

* Description
  * Short describes what service it is.

* Requires
  * The services belong to "Requires" should be active to activate service "A".

* Wants
  * When the service "A" executes, the services belong to "Wants" also be executed. Unlike "Requires", "A" can be active even if the service in "Wants" fails to run.

* BindsTo:
  * If the services in "BindsTo" become inactive, "A" also be inactive.

* PartOf:
  * If the services in "PartOf" try to restart or inactive, "A" follows their action.

* Before:
  * Services in "Before" will not be executed until the "A" executed == Service "A" should be executed before the services in "Before".

* After:
  * Services in "After" will be executed first then "A" == Service "A" wait its execution after the services in "After".

* Conflicts:
  * Services in "Conflicts" will not executed at the same time with "A". If "A" is active, "Confilicts" will be inactive, and vice versa.

### 2. Service

Service section defines the execution related tasks for service "A".
One thing important is there are multiple types for define the active of service "A".

* Type = simple, forking, oneshot, dbus, notify
  * "Type" represents the conditions for active. If the service satisfies the condition in "Type", its state changes to active from activating.
    * simple: Executing the service is considered as active.
    * forking: When the child process of the service is created, it is considred as active.
    * oneshot: Executing the service is considered as active, but it is considered that the main process of the service exits.
    * dbus: When the service acquires the dbus it defined, it is considered as active.
    * notify: When the service sends message through sd_notify, it is considered as active.

* Exec
  * Execution command or file for the service.

* ExecStart Post and ExecStart Pre
  * Execution command or file before or after the service starts.

* ExecStop Post and ExecStop Pre
  * Execution command or file before or after the service stops.

### 3. Install

Install section contains the information when the service "A" enabled (installed) by systemd.
Usually, install section includes ".target" dependency using WantedBy or RequiredBy.
When the system is booted, systemd launches the services following the target level: init.target -> basic.target -> ... -> multi-user.target -> graphical.target.
If the service has multi-user.target dependency in Install section, systemd launches it at the multi-user.target turn.

It is also can be used to manage the services as a group.

* WantedBy and RequiredBy
  * Define the services that should be enabled when the service "A" is enabled == Enable them with "A".

* Alias
  * Create symbolic link for the service "A" in the name in "Alias".

* Also
  * Define the services that should be enabled or disabled when the service "A" is enabled or disabled.

## Appendix

### "override.conf"

In ".service file" section, I explained there are two paths for systemd: /lib/systemd and /etc/systemd.
If there are two same service files both in /lib/systemd and /etc/systemd, the file in /etc/systemd has prioirity.
As a result, it is possible to customize the systemd service with /etc/systemd.

However, if the original service file in /lib/systemd is updated and its contents changes, service file in /etc/systemd might causes unintended behavior.
It is high complexity task that managing service file in /etc/systemd whenever the original service file is updated.
"systemd" supports override.conf feature for this situation.
For example, suppose that I want to add dependency for libvirtd.service.
I can create a directory called /etc/systemd/system/libvirtd.service.d and write a "override.conf" file and its content is:

```bash
[Unit]
After=dev-mapper-extra.device
```

I add a dependency for libvirtd.service using override.conf file.
With the override.conf, service file in /lib/systemd dynamically add the dependency when the systemd define the order between the services.

Following represents the order of priority that systemd has:
/lib/systemd/ -> /etc/systemd -> /etc/systemd/${unit_name}.d/override.conf

With override.conf, it is easy to manage the unit files with less dependencies.
