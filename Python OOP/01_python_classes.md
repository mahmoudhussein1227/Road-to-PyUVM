# Road to PyUVM, Part 1: Python Classes

Every PyUVM testbench is built from classes: drivers, monitors, agents, and environments are all Python classes. Before touching UVM, it helps to know what a class is and what happens in memory when you create an object.

**Topics covered**

1. [Class declaration](#1-class-declaration)
2. [Creating objects and what happens in memory](#2-creating-objects-and-what-happens-in-memory)
3. [Names are references: assignment is not copying](#3-names-are-references-assignment-is-not-copying)
4. [Why this matters for PyUVM](#6-why-this-matters-for-pyuvm)


---

## 1. Class declaration

A class is a blueprint. It describes what its objects will look like and what they can do.

### Minimal class

```python
class Packet:
    pass
```

- `class` is the keyword, followed by the class name.
- By convention, class names use `CamelCase`.
- The body is indented. `pass` is a placeholder for an empty body.

### A class with content

```python
class Packet:
    protocol = "AXI"          # class attribute, shared by all objects

    def show(self):           # a method
        print("I am a packet")
```

Everything the class defines (attributes and methods) lives inside its body. Attributes, `self`, constructors, and methods get their own posts later in the series.


## 2. Creating objects and what happens in memory

Creating an object (also called an *instance*) looks like a function call:

```python
p1 = Packet()
p2 = Packet()
```

### Step 1: the class itself is an object

When Python executes a `class` statement, it builds a **class object** in memory. That object holds the class attributes and the methods, and the name `Packet` points to it.

```python
print(type(Packet))    # <class 'type'>
print(type(p1))        # <class '__main__.Packet'>
```

Classes are themselves instances of `type`, which is why they can be passed around, stored in variables, and created dynamically. The UVM factory relies on exactly this when it creates components by class.

### Step 2: calling the class creates an instance

`Packet()` triggers two steps, in order:

1. `__new__` allocates a fresh, empty object in memory.
2. `__init__` initializes that object (this is the constructor you usually write).

```python
class Packet:
    def __new__(cls):
        print("1. __new__: allocating memory")
        return super().__new__(cls)

    def __init__(self):
        print("2. __init__: initializing the object")

p = Packet()
# 1. __new__: allocating memory
# 2. __init__: initializing the object
```

You will rarely override `__new__`. In day-to-day code you only write `__init__`, which is covered in the constructors post.

### Step 3: the variable stores a reference

`p1` does not contain the object. It is a **name bound to a reference** to the object, which lives elsewhere in memory.

### Each call makes a separate object

```python
print(id(p1))      # e.g. 140234871234
print(id(p2))      # e.g. 140234871298  -> different address
print(p1 is p2)    # False
```

`id()` returns the object's identity. In CPython this is the object's memory address. `is` compares identities, while `==` compares values (and by default, for your own classes, `==` falls back to identity until you define `__eq__`).

---

## 3. Names are references: assignment is not copying

```python
p3 = p1
print(p3 is p1)    # True
```

No new object is created. `p3` is a second name pointing to the same object.

```python
p1.protocol = "APB"   # creates an instance attribute on the object
print(p3.protocol)    # APB  -> same object, so the change is visible
```

To get an independent object you must create one explicitly, either by calling the class again or by copying:

```python
import copy

p4 = copy.copy(p1)       # shallow copy: new object, same inner references
p5 = copy.deepcopy(p1)   # deep copy: new object, inner objects copied too
print(p4 is p1)          # False
```

### Common pitfall: shared mutable state

```python
def modify(pkt):
    pkt.protocol = "CHANGED"

modify(p1)
print(p1.protocol)   # CHANGED -> the function received a reference, not a copy
```

Passing an object to a function (or to another component) passes the reference. The callee can modify the original.

---

## 4. Why this matters for PyUVM

- Drivers, monitors, sequencers, agents, and environments are all classes. `class MyDriver(uvm_driver)` is the class-declaration syntax from section 1, plus inheritance.
- The UVM factory creates components as objects, using the class-as-object idea from section 2.
- Components pass handles to each other (for example, a monitor's analysis port connecting to a scoreboard). Because names are references (section 3), all of them see the same object. Modifying a transaction in one place changes it everywhere that holds the handle, so copy it when you need an independent one.
- Class attributes (section 4) are shared by every instance, so avoid storing per-object data there.

---

**Next up:** Part 2, Attributes and `self`.
