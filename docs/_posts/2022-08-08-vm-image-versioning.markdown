---
layout: posts
title:  "VM Management: Recovery and Checkpoint Images in Limited Storage Capacity"
author: Minjong Ha
published: false
date:   2022-08-08 14:37:00 +0900
---

Qemu hypervisor presents qcow2 format for disk image.
Guest considers the qcow2 image as a physical disk, and qemu hypervisor handles the I/O requests from the VM.
It is efficient since the disk size increases depending on the usage of the disk.
For example, assume that there is a qcow2 image having 100 GB virtual size.
Guest OS can see the 100 GB storage device through the qemu hypervisor.
If the VM only uses 10 GB on the storage, qcow2 image only grows up to 10 GB (+ a).
If the VM uses more than 10 GB on the storage, qcow2 image also expands to 20 GB (+ a) 

qcow2 format image also supports snapshot feature.
There are two snapshot methods: internal and external.
Internal snapshot contains its snapshot data in the original base image file.
On the other hand, external snapshot creates seperate so called overlay image.
In this post, I will explain about external snapshot.

<!-- overlay image overview image -->

<!-- L1/L2 table + refcount table-->



