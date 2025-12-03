## Explain the difference between class attributes and instance attributes.

**Class Attributes**: Variables that are owned by the class itself. They are shared by all instances of that class. They are defined directly inside the class body (outside __init__).

**Instance Attributes**: Variables that are owned by a specific instance of the class. They are defined inside methods (usually __init__) using self.

**Key Distinction**: `Memory`. A class attribute is stored once in memory. Instance attributes are stored separately for every object created.


## What is the Method Resolution Order (MRO) in Python? Explain how the C3 Linearization algorithm works.

MRO stands for `Method Resolution Order`. It is the order in which Python looks for a method in a hierarchy of classes. MRO follows a specific sequence to determine which method to invoke when it encounters `multiple classes` with the same method name.. For Example :

```python
class A:
    def rk(self):
        print("In class A")

class B:
    def rk(self):
        print("In class B")

# classes ordering
class C(A, B):
    def __init__(self):
        print("Constructor C")

r = C()
```
In the above example the methods that are invoked is from class B but not from class A, and this is due to Method Resolution Order(MRO). 

> To get the method resolution order of a class we can use either `__mro__` attribute or `mro()` method. By using these methods we can display the order in which methods are resolved.

### The C3 Linearization Algorithm:
We use the C3 Linearization algorithm to determine the MRO. It ensures three important properties:

- Children precede parents: A subclass is always checked before its superclass.

- Parent order is preserved: If a class inherits from A and B `(class Child(A, B))`, A is checked before B.

- Monotonicity: The MRO of a subclass implies the MRO of its superclasses (no unexpected shuffling of order when you derive a new class).

## What is the significance of @property decorator?
The `@property` is a built-in decorator for the property() function in Python. It is used to give "special" functionality to certain methods to make them act as `getters`, `setters`, or `deleters` when we define properties in a class.

```python
class House:

    def __init__(self, price):
        self._price = price

    @property
    def price(self):
        # getter
        return self._price

    @price.setter
    def price(self, new_price):
        if new_price > 0 and isinstance(new_price, float):
            self._price = new_price
        else:
            print("Please enter a valid price")

    @price.deleter
    def price(self):
        del self._price
```
Specifically, you can define three methods for a property:

- A getter - to access the value of the attribute.
- A setter - to set the value of the attribute.
- A deleter - to delete the instance attribute.

## What does __ call __ do?
The `__ call __` method in Python is a special method (a "dunder" method) that allows an instance of a class to be treated and executed as if it were a function.

If a class implements `__ call __`, its instances become callable objects.

### How it Works
When you define `__ call __(self, *args, **kwargs)` within a class, you define the behavior that occurs when you use the function call syntax—parentheses ()—on an instance of that class.

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, number):
        """
        The method that is executed when the instance is called like a function.
        """
        return number * self.factor

# 1. Instantiate the class (creates the object)
double = Multiplier(2) 
triple = Multiplier(3)

# 2. Call the instance like a function (executes __call__)
print(double(10)) # Output: 20 (Executed double.__call__(10))
print(triple(10)) # Output: 30 (Executed triple.__call__(10))
```

**Creating a Counter Function**: Lets's create a counter object using __call__:

```python
class C:
    def __init__(self):
        self.count = 0
​
    def __call__(self):
        self.count += 1
        return self.count
​
count = C()
print(count()) -> 1
print(count()) -> 2
print(count()) -> 3
```