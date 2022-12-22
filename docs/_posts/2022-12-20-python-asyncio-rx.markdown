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

### Coroutine

Coroutine is a basic execution unit in asyncio.
Event loop executes multiple coroutines at the same time like context switch.
However, coroutine only supports concurrency but not parallelism.
Coroutine can be represented as an async function like:

```python
async def coroutine_func():
    # Do something
```

I will explain more details about it on "Event Loop" section.


### Task

Task is used to assign multiple tasks at the same time.
By the task, coroutines can have parallelism.
Task can be assign the coroutines like:

```python
asyncio.create_task(coroutine_func_1())
asyncio.create_task(coroutine_func_2())
asyncio.create_task(coroutine_func_3())
```

I will explain more details about it on "Event Loop" section


### Event Loop

<img data-action="zoom" src='{{ "../assets/images/posts/2022-12-20-python-asyncio-rx/async_eventloop.jpg" | relative_url }}' alt='relative'>
(image from [here](https://hackersandslackers.com/intro-to-asyncio-concurrency/))

Event-loop can execute async tasks and callbacks in python.
And you can assign coroutines and tasks into the event loop.

If there is a callback function to wait for response, you can run event loop forever with "loop.run\_forever()"
It keeps alive event loop until one of the task call loop.stop().

If there are tasks having long latency for return such as network communication, asyncio is an effective choice.
It can handle multiple tasks concurrently with a single thread.

```
               Eventloop
                   │
  Coroutine 1      │
                   │
┌─────────────┐    │
│             │    │
│             │    │
│await        │    │      Coroutine 2
└─────────────┘    │    ┌─────────────┐
       :           │    │             │
       :           │    │             │
       :           │    │await        │
┌─────────────┐    │    └─────────────┘
│             │    │           :
│             │    │           :
│await        │    │           :
└─────────────┘    │           :
       :           │    ┌─────────────┐
       :           │    │             │
       :           │    │             │
       :           │    │return       │
┌─────────────┐    │    └─────────────┘
│             │    │
│             │    │
│return       │    │
└─────────────┘    │
                   │
                   ▼
```

Above image represents the working flow of event loop with two coroutines.
Each coroutines can be assigned to event loop by "create\_task(coroutine\_1()) and create\_task(coroutine\_2())

Folloings are simple examples

```python
import asyncio

def _handle_exception(self, loop, context):
    exception = context.get("exception")
    logging.error(exception)
    loop.stop()

# async function can be considred as a coroutine
async def your_func(_arg):
    # 3. wait until job_1 returns
    await job_1()
    # 4. wait until job_1 returns
    await job_2()
    # 5. wait until job_1 returns
    await job_3()

    # 6. run three tasks at the same time
    asyncio.create_task(task_1())
    asyncio.create_task(task_2())
    asyncio.create_task(task_3())


# 1. Create event loop
loop = asyncio.get_event_loop()

# (optional) assign exception handler
loop.set_exception_handler(_handle_exception)

# 2. Assign tasks into the loop (you can assign multiple tasks)
loop.create_task(your_func(_arg))

# 7. Make loop run forever
loop.run_forever()
```

Above codes create a event loop, and create task.
"your\_func(\_arg)" is called and it calls three await jobs and three "asyncio_create_task" tasks.
"await" and "asyncio.create_task" are both async functions that representing coroutines.
However, while "asyncio.create_task" can share the execution with main, await demands waiting the job finished.
In codes, three awaited jobs will be executed sequentially while three tasks run at the same time.
It is important to choose proper asyncio function.


## RxPY

RxPy is a python library for reactive programming.
Rx(Reactive Extensions) provides asynchronous API with "Observable" interface to implement FRP(Functional Reactive Programming).
It is a stream that creating and emitting a event for state change of "Observable" objects.


## Appendix

- [Repository for Python asyncio and RxPY study](https://github.com/minjong-ha/python-asyncio-study)
