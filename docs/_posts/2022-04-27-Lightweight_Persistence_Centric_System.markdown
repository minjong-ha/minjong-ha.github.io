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
At the time I graduated from the University with the research about the non-volatile memory, also called persistent memory, it was not a major trend in academia as far as I know.
Compared to the costs and efforts of applying non-volatile memory from the volatile-memory, the performance improvement did not look attractive.
Thus, the companies preferred to increase the memory physically, and the researchers start to study about the "CXL memory" instead.

However, I read an articles that the researchers in CAMEL lab (KAIST) implemented a integrated persistence centric system with the Intel OptaneDC Persistent Memory.

The paper, "LightPC: Hardware and Software Co-Design for Energy-Efficient Full System Persistence)", will be presented in ISCA'22 (International Symposium on Computer Architecture).
As soon as the paper is announced, I will read and analyze about it and update this posts!


## 1. Abstract

The following summarizes represent the contributions of the paper

### 1. Open-Channel architecture for PMEM

### 2. Transparent full persistence mechanism

### 3. Practical non-volatile computing and prototypes

[PDF](https://dl.acm.org/doi/pdf/10.1145/3470496.3527397)
