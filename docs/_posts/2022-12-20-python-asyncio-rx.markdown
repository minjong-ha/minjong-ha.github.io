---
layout: posts
title:  "Python: RxPY with asyncio"
author: Minjong Ha
published: false
date:   2022-12-15 12:00:00 +0900
---

<!--- Lets study about python asyncio and rx with event_loop() --->


## Introduction

When you want to implement reactive, passive-working application, asynchronous programming is an essential option.
Asyncio is a python library that supports async features: event loop, coroutine, task, and etc.

Asyncio usually works together with callback functions.
RxPY is a python library that presents observable object for callback.

In this post, I will explain about the basic concept and usage of them.


## Asyncio

### Event Loop


<img data-action="zoom" src='{{ "../assets/images/posts/2022-12-20-python-asyncio-rx/async_eventloop.jpg" | relative_url }}' alt='relative'>
(image from [here](https://hackersandslackers.com/intro-to-asyncio-concurrency/))

Event-loop can execute async tasks and callbacks in python.
Usually, it requires the command such as asyncio.run()

Using only one event loop in the entire application.
asyncio.run(main()) will return one event loop.


If there are tasks having long latency for return such as network communication, asyncio is an effective choice.
It can handle multiple tasks concurrently with a single thread.


## Appendix

- [Repository for Python asyncio and RxPY study](https://github.com/minjong-ha/python-asyncio-study)
