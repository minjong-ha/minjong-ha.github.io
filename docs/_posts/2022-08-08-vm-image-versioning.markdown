---
layout: posts
title:  "VM Management: Implement Recovery and Checkpoint for Image"
author: Minjong Ha
published: true
date:   2022-08-08 14:37:00 +0900
---

## Introduction

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

## Qcow2 Architecture

<img data-action="zoom" src='{{ "../assets/images/posts/2022-08-08-vm-image-versioning/qcow_struct.png" | relative_url }}' alt='relative'>

Above image represents the header (metadata) of qcow2 file.
"qcow2" image can be divided to seven sectors: Header, L1 Table, Refcount Table, More Refcount Blocks, Snapshot Headers, L2 Tables and Data Reserved (data cluster).

Header includes metadata of qcow2 image such as number of snapshot it has, actual(current) size, virtual(maximum) size, and etc.
Snapshot header also represents the information of snapshot that the image has.

L1 Table, L2 Table and Refcount Table, More Refcount Blocks are two-level tables for managing data allocation with Copy-on-Write (CoW).
L1, L2 Tables are corresponding to multi-level page tables.
It indicates the data cluster, where the data are actually allocated.
On the other hand, refcount are corresponding to represent the attribute of each data clusters.
There are three types of refcount: 0, 1, and more than 2.
It represents the number of referenced of data cluster.
For example, when the refcount is 0, it represents it is free, non-allocated data cluster (no L2 table indicates it).
Refcount 1 means it is allocated, used data cluster (there is a L2 table exists indicating the data cluster).
Application can overwrite the data cluster having refcount 1.
Refcount more than 2 is similar to 1.
It means that the data cluster is used, allocated.
However, the difference is that the data clusters having refcount more than 2 should be allocated to new data cluster, just like Copy-on-Write.
I will explain more details about it in later section.

## Qcow2 Data Allocation

<img data-action="zoom" src='{{ "../assets/images/posts/2022-08-08-vm-image-versioning/qcow_image_achitecture.png" | relative_url }}' alt='relative'>

Above image represents the architecture of qcow2 file when it writes the data.
qcow2 image manages the data with L1,L2 tables and Refcount tables like page table.
Each table entries are corresponding to the data clusters and each data clusters having 64KB default size (512B - 2MB).

```bash
qemu-img create -f qcow -b original_image_name new_overlay_image
```

qcow2 image supports "overlay" image which is CoW based data allocation.
If the user execute above command, new_overlay_image is created and indicates the original_image as a backing file.
When the application tries to read data from new_overlay_image, it returns the value referencing its data cluster or the backing file.
If the application tries to write new data or overwrite data cluster, overlay_image allocates new data clusters and writes the data.
new_overlay_image copies L1,L2 tables and refcount tables from the original_image when generated.
However, unlike it copies L1,L2 tables, it copies refcount tables with increment.
If the original refcount table has the values 0,1,1,0,1, then new_overlay_image has the values 1,2,2,1,2 (overlay image only has 1 or more than 2 refcount).
When the application tries to write data, it checks the refcount of the data cluster.
If the refcount is 1, it means the original data cluster is free, non-allocated state.
Since the original_image does not have allocated data cluster, it generates new data cluster and write the data.
If the refcount is more than 2, it means the original data cluster is exist.
Hence, overlay_image allocates new data cluster and update the L1,L2 table.
The difference between refcount 1 and more than 2 is the prior does not change refcount.
However, the later updates L1,L2 table and refcount at the same time, since the CoW happened.
With the CoW based data cluster management, qcow2 image can hold the original data and overlap the new data at the same time.

## Image Overlay

<!-- Overlay Image -->

```text
.--------------.    .-------------.    .-------------.    .-------------.
|              |    |             |    |             |    |             |
| RootBase     |<---| Overlay-1   |<---| Overlay-1A  <--- | Overlay-1B  |
| (raw/qcow2)  |    | (qcow2)     |    | (qcow2)     |    | (qcow2)     |
'--------------'    '-------------'    '-------------'    '-------------'

-------------------------------------------------------------------------------------

.-----------.   .-----------.   .------------.  .------------.  .------------.
|           |   |           |   |            |  |            |  |            |
| RootBase  |<--- Overlay-1 |<--- Overlay-1A <--- Overlay-1B <--- Overlay-1C |
|           |   |           |   |            |  |            |  | (Active)   |
'-----------'   '-----------'   '------------'  '------------'  '------------'
   ^    ^
   |    |
   |    |       .-----------.    .------------.
   |    |       |           |    |            |
   |    '-------| Overlay-2 |<---| Overlay-2A |
   |            |           |    | (Active)   |
   |            '-----------'    '------------'
   |
   |
   |            .-----------.    .------------.
   |            |           |    |            |
   '------------| Overlay-3 |<---| Overlay-3A |
                |           |    | (Active)   |
                '-----------'    '------------'
```

Above image represents the overview of the overlays.
Since the changes are only updated to the overlay images, it is possible to create two different images based on the same original image.

```bash
qemu-img commit ${overlay_image}
```

Qcow2 image also presents commit feature.
For example, if I commit Overlay-1A in the first figure, every data clusters and L1/L2 tables, and refcounts are merged to the Overlay-1.
After the merging completed, Overlay-1A now has a clear, vanila overlay image.
Since it merging the data from 1A to 1, additional storage is required (maximum Overlay-1A size).

As you can see, qcow2 overlay is very similar to git.
We can change the disk image of VM with libvirt, and create new branch, commit.
The only difference is qcow2 is not available for recovering old history.

## Conclusion?

qcow2 presents useful snapshot features and backing logic motivated by CoW.
It makes managing the versions of VMs convinient.
However, it is still the problem that write amplification.
Even if the OS looks like idle, it continuously performs its tasks.
One of the task is executed with file system.
Since it updates or writes its system files, thin-provisioned qcow2 file grows up more than 1 GB only with the booting.
For this reason, managing VM versions requires amount of storage.

## Apendix

### Capacity Coverage for QCow2 image

A single L1 or L2 table can hold maximum 262,144 entries (2^18).
And the size of data cluster can be 512 bytes to 2 MB (2^9 to 2^21).
Since there is only one L1 table for qcow2 image, the maximum number of L2 entries are 2^18 x 2^18 = 2^36.
In the case when the data cluster is 512 bytes, qcow2 image can handle maximum 2^36 x 2^9 = 2^45 bytes (32 TB).
In the case when the data cluster is 2 MB, qcow2 image can handle maximum 2^36 x 2^21 = 2^57 (128 PB).

### How libvirt communicates with QEMU?

"libvirt" only manages qemu-related information as XML format and provides additional features to users.
"libvirt" itself only perform XML managing.
QEMU-related works are performed by executing command through g_loop asymmetrically.
For example, libvirt only generates xmls for a snapshot.
Actual snapshot image is created by QEMU through "qemu-img" command from libvirt.

## References

[Qcow2 Overlay Images](https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html)
