---
layout: posts
title:  "Host-Guest Communication: Full vs Para Virtualization - 2"
author: Minjong Ha
published: false
date:   2022-04-25 19:13:43 +0900
---

## Introduction
In previous post, we analyzed how the vCPU thread working and understoodd the concept of the vCPU in the GUEST_MODE under the QEMU-KVM hypervisor.
Now, lets look around how Para-Virtualization (virtio) works and compare it to the Full-Virtualization.
Since we already understand the detail operations, comparison between the Full and Para Virtualization is much easier.

## Full-Virtualization
Assume that we try to send network packet inside the Guest.
In the Guest, the application requests to the Guest's Kernel and hands over the data to make it as a packet and send.
However, in Full-Virtualization, Guest does not know that the network device is a emulated, virtual device.
The GUEST's kernel will behave as if the device were a real physical device, which means it will write the data to the MMIO area of the device.
Since the device is not a real, The host's help is required to transfer the guest's data to the actual, physical device.
It requires work on a host that can access the physical device.

<!-- It is not sure-->
To handle it, QEMU-KVM implements the BIOS to trigger the __vmexit__ when the write is made to the memory area allocated to the virtual device.
As I already mentioned in the previous post, the KVM returned from the GUEST_MODE through the vmexit tries to figure out the reason of it.
Soon, it discovers that a device working for the Guest has a data to handle.
KVM __copies__ the data inside the device's memory area to the kernel space (1st user-kernel data copy).
However, the device itself is a thread in the user area, KVM (kernel) could not handle it over to the device.
KVM returns the result of the ioctl() that the QEMU had called.
Now, QEMU looks up the reason of vmexit.
It figures out the vmexit is triggered since the device request, and copies the data from the kernel area (2nd user-kernel data copy).
Finally, QEMU passes over the data to the native kernel drive and it causes 3rd user-kernel data copy.


## Para-Vitualization
Not only the vmexits, user-kernel data copy is the huge overheads.
It is because that the Guest does not know the device is a virtual, emulated device.


## Experiment

## Conclusion


