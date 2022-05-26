---
layout: posts
title:  "Host-Guest Communication: Full vs Para Virtualization - 2"
author: Minjong Ha
published: false
date:   2022-04-25 19:13:43 +0900
---

## Introduction
In previous post, we analyzed how the vCPU thread working and understand the concept of the GUEST_MODE in the QEMU-KVM hypervisor.
Now, lets look around Para-Virtualization (virtio) works and compare it to the Full-Virtualization.
Since we already understand the detail operations, comparison between the Full and Para Virtualization is much easier.

## Full-Virtualization
Assume that we try to send network packet inside the Guest.
In the Guest, the application requests to the Guest's Kernel and hands over the data to make it as a packet and send.
However, in Full-Virtualization, Guest does not know that the network device is a emulated, virtual device.
The GUEST's kernel will behave as if the device were a real physical device, which means it will write the data to the MMIO area of the device.

Since the device is not a real, The host's help is required to transfer the guest's data to the actual, physical device.
It requires work on a host that can access the physical device.

## Para-Vitualization

## Experiment

## Conclusion


