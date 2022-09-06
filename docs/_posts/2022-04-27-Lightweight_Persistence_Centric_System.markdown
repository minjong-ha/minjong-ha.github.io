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

The following summarizes represent the contributions of the paper

* Open-Channel architecture for PMEM

* Transparent full persistence mechanism

* Practical non-volatile computing and prototypes


## References

[PDF](https://dl.acm.org/doi/pdf/10.1145/3470496.3527397)
