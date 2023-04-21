---
layout: posts
title:  "Python: Abstract Base Classes (ABC)"
author: Minjong Ha
published: false
date:   2022-05-08 17:13:43 +0900
---

Python is a versatile programming language that provides a lot of features to make your code more structured and maintainable.
One such feature is the support for **Abstract Base Classes (ABCs)** with the `abc` module.
In this post, I will explain about the concept of Abstract Base Classes and learn how to use the `abc` module in Python.

## Abstract Base Classess (ABC)

In object-oriented programming, an abstract base class (ABC) is a class that cannot be instantiated and is meant to be subclassed by other classes.
It defines a common interface that all derived classes must adhere to, ensuring that specific methods are always implemented.
This can help improve code readability and reduce the chance of runtime errors.

## Usage

To use the `abc` module in Python, you need to import the `ABC` class and the `@abstractmethod` decorator:

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

    @abstractmethod
    def perimeter(self):
        pass
```

Since I define a `Shape` abstract base class with two abstract method `area()` and `perimeter()`, every classes inherits from `Shape` must have two method.

```python
class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return 3.1415926535 * self.radius * self.radius

    def perimeter(self):
        return 2 * 3.1415926535 * self.radius
```

`Rectangle` and `Circle` have different formular for area and periment.
If a class does not implement abstract methods, python will raise a `TypeError` when the object of the class instantiates.
