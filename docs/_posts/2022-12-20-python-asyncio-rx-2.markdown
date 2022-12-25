---
layout: posts
title:  "Python: RxPY with asyncio - 2"
author: Minjong Ha
published: false
date:   2022-12-17 12:00:00 +0900
---

<!--- Lets study about python asyncio and rx with event_loop() --->

## RxPY

RxPy is a python library for reactive programming.
Rx(Reactive Extensions) provides asynchronous API with "Observable" interface to implement FRP(Functional Reactive Programming).
It is a stream that creating and emitting a event for state change of "Observable" objects.

Observer subscribes observable objects and discover the events that observable makes.
Simply, Observable can call a method of Observer.

```python
class PrintObserver(Observer):
    def on_next(self, value):
        print(value)
    
    def on_completed(self):
        print("DONE")

    def on_error(self, error):
        print(error)

...

source = Observable.from_(["python", "java", "c", "rust"])
source.subscribe(PrintObserver())
```


## Appendix

- [Repository for Python asyncio and RxPY study](https://github.com/minjong-ha/python-asyncio-study)
