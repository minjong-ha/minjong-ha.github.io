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


## .service Files

<!-- Details about service.file-->

### 1. Unit


### 2. Service


### 3. Install


## Appendix

### systemd-analyze
<!-- systemd-analyize critical-chain -->

### What about starting service after the other service finished?
<!-- about FDE: dev-mapper-extra2.device file case-->




