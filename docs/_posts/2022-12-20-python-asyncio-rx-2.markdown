---
layout: posts
title:  "Python: RxPY with asyncio - 2"
author: Minjong Ha
published: false
date:   2022-12-17 12:00:00 +0900
---

<!--- Lets study about python asyncio and rx with event_loop() --->

## RxPY

### Basic Features

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

There are three methods in Observable: on\_next, on\_complete, on\_error.
"on\_next" method notifies that tere is a data will be used next.
"on\_error" method notifies that there is an error in observable.
"on\_complete" method notifies that there is no data will be used next.

In the example, I subscribe a list of string and subscribe with PrintObserver I implemented.
List will be converted to observable by "Observable.from_" function.
Since I subscribe it, PrintObserver will traverse the list and print the value by "on\_next".
"rust" will be printed at last, and PrintObserver will call on\_completed() function.


The features I explained are useful but not a killer.
The reason that RxPy can be killer is it can implement state based reactive application.


### Pipe

You can work with multiple operators with pipe() method.
It allows the programmer can chaining multiple operators together.
For example:

```python
test = of(1, 2, 3) # Observables
subscriber = test.pipe(
        op1(),
        op2(),
        op3()
)
```

In the above example, There is an observable by of() method and it takes the values: 1, 2, and 3.
For each values, 'op1()', 'op2()', and 'op3()' will perform while 'op' means operators of RxPy.
The execution of operators will go on sequential.

```python
from rx import of, operators as op

test = of(1,2,3,4,5,6,7,8,9,10)
sub1 = test.pipe(
        op.filter(lambda s: s%2=0),
        op.reduce(lambda acc, x: acc + x)
)

sub1.subscribe(lambda x: print("Sum of Even numbers is {0}".format(x)))
```

There is a list having 1 - 10 integers.
First, rx traverses the list and filters even numbers with 'filter()'.
Then, add all even numbers in 'x' with 'reduce()' operator.

Since there are many operators in RxPy, reference [here](https://www.tutorialspoint.com/rxpy/rxpy_operators.htm)


## Appendix

- [Repository for Python asyncio and RxPY study](https://github.com/minjong-ha/python-asyncio-study)
- [Tutorial for RxPy](https://www.tutorialspoint.com/rxpy/rxpy_operators.htm)
