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

```python
async def do_call(function, *args, **kwargs):
    #   https://stackoverflow.com/a/52422903
    def is_coroutine_function(function) -> bool:
        while isinstance(function, functools.partial):
            function = function.func
        return inspect.iscoroutinefunction(function)

    if is_coroutine_function(function):
        return await function(*args, **kwargs)

    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(
        None, functools.partial(function, *args, **kwargs)
    )
```

## `do_call_if_callable()`

In some cases, you might need to check whether the function is callable.
`do_call_if_callable()` utility address this needs by verifying the callabllity of a give funtion and invoking `do_call()` if the funtion is callable.

```python
async def do_call_if_callable(function, *args, **kwargs):
    if not function or not callable(function):
        return None

    return await do_call(function, *args, **kwargs)
```

## Other Usages

When dealing the funtion calls, it is important to ensure error handling and execution management.

```python
async def safe_call(function, *args, **kwargs):
    try:
        return await do_call(function, *args, **kwargs)
    except Exception as e:
        print(f"Error while calling {function.__name__}: {e}")

```

```python
async def timeout_call(function, timeout, *args, **kwargs):
    try:
        return await asyncio.wait_for(do_call(function, *args, **kwargs), timeout)
    except asyncio.TimeoutError:
        print(f"Function {function.__name__} timed out after {timeout} seconds")
    except Exception as e:
        print(f"Error while calling {function.__name__}: {e}")
```
