---
layout: posts
title:  "Lightweight Persistence Centric System"
author: Minjong Ha
published: false
date:   2022-04-27 09:00:00 +0900
---


## 0. What is LightPC?


[Non-volatile computing demo (LightPC) English version](https://www.youtube.com/watch?v=HzYe_xooOKk&feature=emb_title)

[LightPC Presents a Resilient System Using Only Non-Volatile Memory](https://news.kaist.ac.kr/newsen/html/news/?GotoPage=1&list_e_date=&list_s_date=&mng_no=20111&mode=V&skey=&sval=)

"A KAIST research team has developed hardware and software technology that ensures both data and execution persistence."

This is very interesting research.

## 1. Abstract

Persistent Memory (PMEM) was released by Intel in 2019, and it can persist the data even if there were a blackout.
However, although its characteristics such as x10 times capacity and non-volatile, it also has limitations since its lower performance than volatile memory with hard to use.
In this paper, the researchers suggest LightPC: easy to use non-volatile computing platform consist of dedicated OS.

The following summarizes represent the contributions of the paper:

* Open-Channel architecture for PMEM

* Transparent full persistence mechanism

* Practical non-volatile computing and prototypes


## 2. Introduction

Long-running applications such as server-class domains require a stable, robust system against the failures since their characteristics.
They run more than a few month to year at once, and take a few hours for MTBF (Meat Time Between Failure).
For this reason, most of the long-running applications support persistence mechanism.
They periodically flush the volatile data in the memory device to the non-volatile storage for recovery.

<!-- this is my personal opinion-->
However, volatile memory has limitations.
They have small capacity relative to the storage and there are gap between the flushes which is the potential data missing point.
For example, performance critical application, such as in-memory databases, have to pay expensive cost for large capacity.
And they should periodically flush their memories to the storages.
It is the chanllenge for the memory-intensive workload that leverages its performance.

## 3. PMEM (Persistent Memory)

<!-- maybe the first and the last...-->
Intel Optane DC Persistent Memory (DCPM, PMEM) is the first commercial product released by Intel in 2019.
For better understanding, I can describe it as a SSD installed in DIMM slot.
It has larger capacity than ordinary memory and also has persistency.
And since the device is installed on the DIMM, PMEM has better read, write performance than normal storage devices.
However these advantages are also disadvantages.
It has smaller capacity than normal storage device, and higher latency than volatile memory.

In addition, the paper explain more limitations.

<!-- limitaion 1 slides-->
First, it requires source code modification.
Since PMEM is installed through DIMM, it is possible to use memory-intensive application on it.
However, difference between PMEM and memory, which is volatility, I cannot convince that the characteristics of PMEM are fully exploited simply by changing the device.
PMDK (Persistent Memory Development Kit) supports PMEM aware application modification.

<!-- limitation 2 slides-->
Second, it has unpredicatable latency

<!-- limitation 3 slides-->
Third, complicated PMEM's DIMM-level architecture which causes non-deterministic read latency

For these reasons, it is very challenable that utilize PMEM for leaveraging system performance.


## 4. Design (LightPC)

## 5. Experiment

## 6. Conclusion







## References

[PDF](https://dl.acm.org/doi/pdf/10.1145/3470496.3527397)
[Slides](https://www.iscaconf.org/isca2022/slides/isca22-kwon.pdf)
[pmem.io](https://pmem.io/)

