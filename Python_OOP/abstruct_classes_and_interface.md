## What are ABCs (Abstract Base Classes) in Python?
Abstract base classes are tools used to create a `blueprint` for other classes. They do this by defining abstract methods that must be implemented by all classes that inherit the abstract base class. 

This helps to ensure common behavior across a group of classes. To denote abstract methods, we'll use the `abstractmethod` decorator from the `abc` module; more on this to come. 

- Abstract base classes are only meant to be inherited, and are never instantiated themselves. 
- Not every method in an abstract base class needs to be abstract. These methods are known as concrete methods, and they're inherited like normal. 

Let's say we were were defining a MiddleSchool and a HighSchool class; we'd want both to follow a similar "blueprint", such as allowing students to enroll, as well as add a course. In the School abstract base class, we could define two abstract methods; enroll and add_course. Any class then wishing to inherit School (such as MiddleSchool or HighSchool) must provide it's own implementation of enroll and add_course. Both classes would be free to define other methods, and may even inherit a concrete method from School.

### Creating an abstract base class
```python
from abc import ABC, abstractmethod

class School(ABC):
    @abstractmethod
    def enroll(self):
        # this method must be implemented in classes that inherit from it
        pass

    def graduate(self):
        # this concrete methods will be enherited but not mandatory.
        print("Congratulation!")


class HighSchool(School):
    def enroll(self):
        # must have method
        print("Welcome to high school!")

```
- Inherit from the `ABC` class in the `abc` module
- pass
- Any class that enherits School `must` impement `enroll()` method
- Can also implement concrete methods
- Will raise `TypeError` if enroll is defined in the `Highschool` class


## Interfaces in Python
### What is an interface?
An interface defines a contract between a class and its users. It specifies a set of methods that a class must implement in order to be considered compatible with the interface. In Python, interfaces can be implemented using abstract base classes (ABCs).

For example, suppose we have a Shape interface that specifies a single method, area(). We can define the Shape interface in Python using the abc module as follows:

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass
```