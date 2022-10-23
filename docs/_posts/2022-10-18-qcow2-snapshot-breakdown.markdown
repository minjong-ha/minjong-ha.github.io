---
layout: posts
title:  "QCoW2 Snapshot Breakdown"
author: Minjong Ha
published: false
date:   2022-10-18 16:59:00 +0900
---

## Introduction
I already wrote a post about VM image snapshot in ${url_link}.
However, there was a chance that analyzing more detail actions of qcow2 snapshot with source code.
In this post, I will explain how exactly qcow2 and qemu generate snapshot (overlay image), especially focusing on the copy-on-write.

## Qcow2 Architecture
I already explained the architecture of qcow2 image in ${url_link}.
To understand more details about I/O and snapshots, I analyzed qcow2 header and source codes based on qemu-7.0.0.

Qcow2 header contains several arguments for its action.
"nb_snapshots" and "snapshots_offset" represent the number of snapshot and offsets that image has.
"image_backing_file" represents the path of backing file that image references.

Since the describes about the internal and external snapshots in QEMU related documents<sup>[1](#footnote_1)</sup> and difference between qcow1 and qcow2<sup>[1](#footnote_2)</sup>, I misundertood that internal and external snapshot are added features in qcow2.

However, based on the name and source codes, qemu does not consider the external snapshot as a snapshot.
I think its original name is backing file, and qemu support it from qcow (version 1).
Of course it can be used as a snapshot, however, it is more likely to use it as a template in oVirt.
oVirt supports template feature, which means that you can create a VM template based on the VM image.
Usually, the size of the template is thin-provisioned VM image.
If you generate new VM based on the template, new VM image only has a few MB size and grows.
oVirt uses backing file feature to implement template.

Since qcow2, qemu supports its new feature: snapshot (internal snapshot).
Unlike the backing file, internal snapshot contains every snapshot (overlay) architecture within the single file.
Qemu can switch the VM between the snapshots during the runtime of VM.
Moreover, it also provide memory state snapshot either, so it can reload the VM state when the snapshot created.

## Qcow2 Snapshot

## References

## Footnotes

<a name='footnote_1'><1></a>: qcow2 presents two snapshot: internal and external (https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html)

<a name='footnote_2'><2></a>: The difference from the original version is that qcow2 supports multiple snapshots using a newer, more flexible model for storing them. (https://en.wikipedia.org/wiki/Qcow)
