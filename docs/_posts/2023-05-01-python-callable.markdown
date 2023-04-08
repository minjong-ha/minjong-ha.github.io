---
layout: posts
title:  "Exploring Python's Interesting Usages"
author: Minjong Ha
published: false
date:   2022-05-01 17:13:43 +0900
---

In this post, I share my experience in python about useful implementation.
I discuss the concept of a callable funtion and example that demonstrating the `do_call()`.
It also presents the practical examples with `do_call()`.

I learned about these skills from my co-worker's codes.

## `do_call()`

The `do_call()` utility is designed to handle both synchronous and asynchronous funtion calls seamlessly.
`do_call()` automatically determines the type of function (coroutine or regular funtion).

## `do_call_if_callable()`

In some cases, you might need to check whether the function is callable.
`do_call_if_callable()` utility address this needs by verifying the callabllity of a give funtion and invoking `do_call()` if the funtion is callable.

## Other Usages

When dealing the funtion calls, it is important to ensure error handling and execution management.
