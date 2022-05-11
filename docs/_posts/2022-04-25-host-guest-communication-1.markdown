---
layout: posts
title:  "Host-Guest Communication: Full vs Para Virtualization - 1"
author: Minjong Ha
published: false
date:   2022-04-25 17:13:43 +0900
---

About Host-Guest communication!
Currently writing content....

In this post, I share my research and analysis about the data communication between the host and guest in case of the device request and focus on the differences in the full and para virtualized machine.
I also explain about the vCPU as a background; what is the vCPU and how it works.


## Introduction

A virtual machine in "Full-Virtualization" does not know that it is operating in a virtualized environment.
On the other hand, a VM in "Para-Virtualization" knows that it is since the hypervisor who emulates the device notifies it.
Since the VM already knows that the device is for the virtualization, it has different device request logic.
The para virtualization reduces the data copy between the user and the kernel compared to full virtualization which is one of the most largest overhead in the Host-Guest communication

One of the most famous para-virtualization is "Virtio".
"Virtio" presents the devices and drivers specially dedicated for the virtualization.
If the user specifies the virtio device to the hypervisor, VM tries to connect to the device with virtio driver.
Since the virtio driver is installed in Linux as default, it can be activated through kernel config.
However, in Windows, it is not included in the Windows kernel and the user should install virtio driver manually in the VM.

For example, you can configurate your disk device to the normal SATA emulation or the virtio device disk.
In my personal experiments, virtio outperforms the SATA emulation approximately 30-50% in the sysbench - FIO test.


## Background

### ioctl()

The ioctl() system call manipulates the underlying device parameters of special files([reference](https://man7.org/linux/man-pages/man2/ioctl.2.html)).
Usually, it is used to configurate and send requests to the device.
For instance, user could configurate the printer device via ioctl(); such as what is the font it using and what is the size of the paper in the device.
We can see the information about the ioctl()s that KVM supports in __qemu/linux-headers/linux/kVM.h__


### vCPU()

QEMU-KVM hypervisor supports vCPU, executing the code in the guest directly on the physical CPU.
When the QEMU-KVM hypervisor emulates the CPU for the VM, the emulated CPU is supported by vCPU in the KVM.
When the vCPU enters to the GUEST_MODE, the guest can use it exclusively.
<!-- Otherwise, it should runs operations that incur large overhead, such as code generation for instructions through the QEMU. -->
If there were no KVM support, which means in the QEMU only hypervisor, every guest code should be handled by the QEMU.
QEMU translates the guest code to the suitable instructions for the physical CPUs in the host and the cost is tremendous.
However ,thanks to vCPU, the QEMU-KVM hypervisor leaverages the overall performance of the VM.



## Full-Virtualization

## Para-Vitualization

## Experiment

## Conclusion


[Notion Document: Full-Virtualization(QEMU-KVM) vs Para-Virtualization(Virtio)](https://seen-fact-e72.notion.site/Full-Virtualization-vs-Para-Virtualization-cd4933792f6a4a2b871a385f58592955)
