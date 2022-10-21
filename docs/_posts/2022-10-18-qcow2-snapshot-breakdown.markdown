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
"nb_snapshots" and "snapshots_offset" represent the number of snapshot that image has and the address.
"image_backing_file" represents the path of backing file that image references.

Since the describes about the internal and external snapshots in QEMU related documents<sup>[1](#footnote_1)</sup> and difference between qcow1 and qcow2<sup>[1](#footnote_2)</sup>, I misundertood that internal and external snapshot are added features in qcow2.

## Qcow2 Snapshot

## References

## Footnotes

<a name='footnote_1'><1></a>: qcow2 presents two snapshot: internal and external (https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html)

<a name='footnote_2'><2></a>: The difference from the original version is that qcow2 supports multiple snapshots using a newer, more flexible model for storing them. (https://en.wikipedia.org/wiki/Qcow)
