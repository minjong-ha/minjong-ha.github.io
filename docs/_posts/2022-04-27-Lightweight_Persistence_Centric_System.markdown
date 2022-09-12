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
At the time I graduated from the university with the research about the non-volatile memory, also called persistent memory, it was not a major trend in academia as far as I know.
Compared to the costs and efforts of applying non-volatile memory from the volatile-memory, the performance improvement did not look attractive.
Thus, the companies preferred to increase the memory physically, and the researchers start to study about the "CXL memory" instead.

However, I read an articles that the researchers in CAMEL lab (KAIST) implemented a integrated persistence centric system with the Intel OptaneDC Persistent Memory.

In this post, I will summarize their contributions, explain their architecture, and evaluate their experiment.
Followings are my questions before I read the paper

* How could they maintain the consistency among the each operations?

* How could they reduce the overhead coming from amplification?

* How could they leaverage the write performance?


## 1. Abstract

Persistent Memory (PMEM) was released by Intel in 2019, and it can persist the data even if there were a blackout.
However, although its characteristics such as x10 times capacity and non-volatile, it also has limitations since its lower performance than volatile memory with hard to use.
In this paper, the researchers suggest LightPC: easy to use non-volatile computing platform consist of dedicated OS.

The following summarizes represent the contributions of the paper

* Open-Channel architecture for PMEM

* Transparent full persistence mechanism

* Practical non-volatile computing and prototypes


## 2. Introduction

Long-running applications such as server-class domains require a stable, robust system against the failures since their characteristics.
They run more than a few month to year at once, and take a few hours for MTBF (Meat Time Between Failure).
In addition, it is necessary to recover the system from failure.
Many long-running applications supports persistence mechanism.
They periodically flush the volatile data in the memory device to the non-volatile storage.

<!-- this is my personal opinion-->
<!--
However, volatile memory has limitations.
They have small capacity relative to the storage.
There is a gap between the flushes and it is the potential data missing point.
-->







## References

[PDF](https://dl.acm.org/doi/pdf/10.1145/3470496.3527397)
[Slides](https://www.iscaconf.org/isca2022/slides/isca22-kwon.pdf)
