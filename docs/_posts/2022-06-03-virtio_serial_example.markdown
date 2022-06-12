---
layout: posts
title:  "virtio-serial Linux Host - Windows Guest"
author: Minjong Ha
published: false
date:   2022-06-03 16:13:43 +0900
---

# Virtio-Serial Appeareance in Linux Host and Windows Guest
<!-- What is the virtio-serial?-->
Virtio presents the communication channels between the Host and Guest.
Host and Guest share the memory space and send notification to each others to notify that there are the data that the Host or Guest should read.
For example, if the Host tries to send some data to the Guest, it writes the data it want to send in the virtqueue (v-ring) located in the shared memory and gives a notification to the Guest through the KVM.
The Guest's virtio driver now knows that there are data it should read.
In this operations, virtio provides the communication API using the virtqueue.
Since the Host and Guest both require communication handler, they should install the virtio-aware driver based on the virtio APIs.

Implementing drivers for both Host and Guest is the time consuming works.
Fortunately, Virtio also presents the ready-to-use drivers for the various purposes: virtio-blk, virtio-pci, virtio-serial, and etc.
In this post, I will explain about the virtio-serial drivers.

# Preparation

```xml
<channel type='unix'>
  <source mode='bind' path='/var/lib/libvirt/qemu/f16x86_64.agent'/>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```

There are two steps for using virtio-serial: Install virtio-serial driver, and specify virtio-serial channel in the libvirt domain xml.
Unlike the virtio-serial driver is already installed in the Linux as a kernel configuration, Windows does not support virtio frontend drivers in the initial state.
The user should install the virtio drivers in manually at [here](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)
Also, the user should notifies that the vm has virtio devices to the hypervisor.
Since I manage the VMs in my machine through libvirt, I explain how to add the virtio device on the hypervisor based on it.
Above xml codes represents the example of qemu-guest-agent specification, which is the one of the virtio-serial based guest agent.
(oVirt guest agent also uses virtio-serial channel for communication)

In the xml codes, we can see the two components: source and target.
'source' represents the socket information on the Linux Host.
Application in the Host can connect to the socket in the 'source' and works as a client.
'target' represents the name of port in the Windows Guest.
I will explain details about it through the last sections.


# Linux Host socket connection with QEMU
In the Linux Host, QEMU presents a socket for a channel to communicate with the Guest.
QEMU hypervisor performs a role as a server, and a process which tries to connect to the socket is a client.
Since the QEMU only accept one client, it is impossible connecting multiple processes to the socket unless modifies the source codes of QEMU (It is not sure because I did not try it).

# Windows Guest port connection with WIN32 API
In the Windows Guest, QEMU presents a port for a channel to communicate with the Guest.
In fact, QEMU also provides a port for a channel in the Linux Guest either.
However, in this post, I only explain about the Windows Guest since there are many references for the Linux Guest.
<!-- with characteristics compare with orninary port in WIN32 API -->


