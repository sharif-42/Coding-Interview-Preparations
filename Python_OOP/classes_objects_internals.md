## What is a Metaclass in Python? How does it differ from a standard class, and when would you actually use one?

In Python, everything is an object, including classes themselves.

- Class: A blueprint to create instances (objects).
- Metaclass: A blueprint to create classes.

A metaclass in Python is a "class of a class." Just as a class serves as a blueprint for creating objects (instances), a metaclass serves as a blueprint for creating classes themselves. When you define a class in Python, it is implicitly an instance of a metaclass, with type being the default metaclass for all standard classes.

**Difference from a Standard Class**:
- Role: A standard class defines the structure and behavior of objects, while a metaclass defines the structure and behavior of classes.
- Instantiation: A standard class is instantiated to create objects, whereas a metaclass is used to create classes.
- Control Level: Standard classes control the attributes and methods of their instances. Metaclasses control the creation process of classes, including their attributes, methods, and inheritance hierarchy.

**When to Use a Metaclass**:
- Metaclasses are a powerful, advanced feature primarily used in frameworks and libraries for metaprogramming, where you need to modify or customize the class creation process dynamically. Concrete use cases include:
- Automatic Registration of Classes: Frameworks can use metaclasses to automatically register newly defined classes with a central registry, like a plugin system or ORM.
- Enforcing Design Patterns: Metaclasses can enforce design patterns like Singleton, ensuring only one instance of a class can ever exist.
- API Generation and Validation: They can be used to generate methods or attributes based on class definitions, or to validate that classes adhere to specific API contracts.
- Abstract Base Classes (ABCs): The abc module in Python uses metaclasses to define abstract base classes, ensuring that subclasses implement required abstract methods.
- Adding Common Functionality: Metaclasses can inject common methods or attributes into all classes that use them, reducing boilerplate code.


## What is duck typing and how does Python encourage it?
Duck Typing is a type system used in dynamic languages. For example, Python, Perl, Ruby, PHP, Javascript, etc. `where the type or the class of an object is less important than the method it defines`. Using Duck Typing, we do not check types at all. Instead, we check for the presence of a given method or attribute. The name Duck Typing comes from the phrase:
> "If it looks like a duck and quacks like a duck, it's a duck"

### How Python Encourages Duck Typing

Python is dynamically typed and lacks compulsory interface declarations, which naturally encourages duck typing through several language design choices:

- Lack of Compile-Time Type Checking: `Python checks types only at runtime. This means the interpreter doesn't care what class an object belongs to until it actually tries to execute a method on it`.

- Protocol-Based Design: Many built-in functions rely on "protocols" (sets of methods) rather than explicit class types:
    - Iterable Protocol: Any object with an __iter__ or __getitem__ method can be used in a for loop.

    - Context Manager Protocol: Any object with __enter__ and __exit__ methods can be used in a with statement.

- Sized Protocol: Any object with a __len__ method can be passed to the len() function.

- Emphasis on Composition over Inheritance (Elegance): Developers often design functions to accept any object that meets the necessary behavioral requirements, simplifying the class hierarchy and promoting flexible code.