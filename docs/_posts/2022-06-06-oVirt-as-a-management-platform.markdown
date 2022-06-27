---
layout: posts
title:  "Deploy oVirt as a VM Image Management Platform"
author: Minjong Ha
published: false
date:   2022-06-16 10:48:00 +0900
---

# Abstract
oVirt is an open-source distributed virtualization solution, designed to manage your entire enterprise infrastructure. oVirt uses the trusted KVM hypervisor and is built upon several other community projects, including libvirt, Gluster, PatternFly, and Ansible. (https://www.ovirt.org/)
Usually, oVirt presents a distributed system for the VMs and helps to deploy the cloud infrastructure.
In this post, I will explain how to deploy the oVirt and describe the architecture.
Then I will talk about why I tried to deploy oVirt as an effective VM image footpring management.

# Introduction
Managing VM images is important since the users requires various image shape.
Differences in images may appear in the OS, in versions within the same OS, or in applications depending on usage within the same version.
VM service manager performs numerous major and minor updates and it causes fragmentation of images.
For this reason, We need to use an infrastructure to manage and update the images effectively.

I choosed oVirt as an infrastructure for image management for some reasons.
First, it is open-source software.
There is no additional cost for running after I prepare the machines to deploy.
Second, it presents web-console GUI.
Through the web-console, I can easily create VMs and templates and run the VM itself using web-presented vnc.
It makes VM image update easy and helps the manager in convinient.
Third, it is easy to add hosts and storage.
It is also possible via web-console to add hosts and storage domains.
I can add NFS or glusterFS based storage domain when I need more storage.

There are many VM supports in oVirt as an DaaS, however, I focused on the features for VM image management and describe the reason in the above paragraph.
Especially, what caught my eye in particular is the templated image.
It looks attractive to managing the image versions as a template.

# oVirt: Architecture
I deployed a CentOS 7 as a hosted engine, and three oVirt-nodes (4.3) as host and storage domains.
Also, to manage FQDNs for each nodes, I implemented docker-dnsmasq on the another machine as an internal DNS server.

# Deployment

# Result

