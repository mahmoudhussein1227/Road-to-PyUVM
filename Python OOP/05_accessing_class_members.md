# Road to PyUVM, Part 5: Accessing Class Members

We now know how to build a class: attributes, `self`, a constructor, `__str__`, and methods. This last Python OOP part is about **using** what we built: how to reach an object's attributes and methods from the outside, and what Python does (and does not) do to protect them.

**Topics covered**

1. [The dot operator](#1-the-dot-operator)
2. [Accessing attributes](#2-accessing-attributes)
3. [Accessing methods](#3-accessing-methods)
4. [Accessing through the class vs through an object](#4-accessing-through-the-class-vs-through-an-object)
5. [Python does not enforce encapsulation](#5-python-does-not-enforce-encapsulation)
6. [Naming conventions](#6-naming-conventions)
7. [Name mangling with a double underscore](#7-name-mangling-with-a-double-underscore)
8. [Which one should you use?](#8-which-one-should-you-use)
9. [Common pitfalls](#9-common-pitfalls)
10. [Why this matters for PyUVM](#10-why-this-matters-for-pyuvm)

---

## 1. The dot operator

Everything inside a class or an object, whether data or behavior, is reached with the **dot operator** (`.`):

```
object.member
```

The same operator is used for attributes and for methods. The only visible difference is that a method is followed by parentheses.

```python
class Packet:
    protocol = "AXI"                        # class attribute

    def __init__(self, addr, data):
        self.addr = addr                    # object attributes
        self.data = data

    def show(self):                         # method
        print(f"addr={self.addr}, data={self.data}")

p = Packet(0x10, 0xFF)

print(p.addr)        # attribute: no parentheses
p.show()             # method: with parentheses
```

Inside the class, the same dot is used through `self` (`self.addr`, `self.show()`), as we saw in Parts 2 and 4.

---

## 2. Accessing attributes

### Read

```python
print(p.addr)          # 16
print(p.protocol)      # AXI  (class attribute, found through the object)
```

### Write (update)

```python
p.addr = 0x20
print(p.addr)          # 32
```

### Add a new attribute

Python lets you create an attribute on an object at any time just by assigning to it:

```python
p.comment = "first packet"
print(p.comment)       # first packet
```

This works, but only `p` gets it. Other `Packet` objects do not. It is legal, though usually better practice is to define all fields in the constructor so every object has the same shape.

### Delete

```python
del p.comment
print(hasattr(p, "comment"))    # False
```

### Accessing by name (as a string)

When the attribute name is only known at runtime, use the built-in functions:

```python
getattr(p, "addr")             # same as p.addr
getattr(p, "missing", None)    # returns None instead of raising an error
setattr(p, "data", 0xAA)       # same as p.data = 0xAA
hasattr(p, "addr")             # True
```

### Listing what is available

```python
print(p.__dict__)    # object attributes: {'addr': 32, 'data': 255}
print(dir(p))        # everything reachable from p, including methods and class attributes
```

### Reading something that does not exist

```python
print(p.size)    # AttributeError: 'Packet' object has no attribute 'size'
```

---

## 3. Accessing methods

A method is accessed with the same dot, then called with parentheses:

```python
p.show()                 # call the method
```

Without parentheses you get the bound method object instead of running it:

```python
f = p.show               # not called yet
f()                      # call it later, self is still p
```

Methods can take arguments after `self` and can return values:

```python
class Packet:
    def matches(self, addr):
        return self.addr == addr

print(p.matches(0x20))   # True
```

Remember from Part 4 that `p.show()` is translated into `Packet.show(p)`.

---

## 4. Accessing through the class vs through an object

| Member | Through the class | Through an object |
|---|---|---|
| Class attribute (`protocol`) | `Packet.protocol` works | `p.protocol` works |
| Object attribute (`addr`) | Not available (`Packet.addr` fails) | `p.addr` works |
| Method (`show`) | `Packet.show(p)` (you pass the object) | `p.show()` works |

```python
print(Packet.protocol)    # AXI
print(Packet.addr)        # AttributeError: type object 'Packet' has no attribute 'addr'
```

Class attributes live in the class, so both routes find them. Object attributes live in each object, so the class does not know about them.

Reminder from Part 2: **reading** `p.protocol` falls back to the class, but **assigning** `p.protocol = "APB"` creates an object attribute that hides the class one. To change the shared value, assign through the class: `Packet.protocol = "APB"`.

---

## 5. Python does not enforce encapsulation

In languages like C++, Java, or SystemVerilog you mark members as `private`, `protected`, or `local`, and the compiler stops outside code from touching them.

**Python has no such enforcement.** Every attribute and method can be read, changed, or called from anywhere:

```python
class BankAccount:
    def __init__(self, balance):
        self.balance = balance

acct = BankAccount(100)
acct.balance = -5000        # allowed, nothing stops this
```

Python's philosophy is often summed up as "we are all consenting adults here": the language trusts the programmer and relies on **conventions** to communicate intent, instead of walls to prevent access.

---

## 6. Naming conventions

Since the language will not stop you, the name itself signals how a member is meant to be used.

| Name style | Meaning | Example |
|---|---|---|
| `name` | **Public**: part of the class's interface, use freely | `self.addr`, `show()` |
| `_name` | **Internal/"protected"** by convention: for use inside the class and its subclasses, outside code should leave it alone | `self._queue`, `_check_fifo()` |
| `__name` | **Strongly private** by convention, with name mangling (next section) | `self.__secret` |
| `__name__` | **Dunder**: special names reserved by Python | `__init__`, `__str__` |

```python
class Driver:
    def __init__(self):
        self.name = "drv"          # public
        self._queue = []           # internal: please don't touch from outside
        self.__token = 123         # private by convention + name mangling

    def send(self, item):          # public method
        self._queue.append(item)
        self._log(item)

    def _log(self, item):          # internal helper method
        print("sending", item)
```

A single leading underscore is **only a hint**. This works fine:

```python
d = Driver()
print(d._queue)    # works: Python doesn't block it, but you are ignoring the convention
```

Tools also respect the convention: for example, `from module import *` skips names that start with an underscore, and many editors and linters warn when outside code uses them.

---

## 7. Name mangling with a double underscore

A name starting with **two** underscores (and not ending with two) gets rewritten by Python inside the class, to reduce accidental clashes, especially with subclasses.

```python
class Driver:
    def __init__(self):
        self.__token = 123

d = Driver()
print(d.__token)          # AttributeError: 'Driver' object has no attribute '__token'
```

It looks private, but it is not. Python simply renamed the attribute to `_ClassName__name`:

```python
print(d._Driver__token)   # 123  -> still reachable
print(d.__dict__)         # {'_Driver__token': 123}
```

So `__name` makes accidental access harder, but it is still not real privacy. Its main purpose is avoiding name collisions between a class and its subclasses, not hiding data. Dunder names that **end** with two underscores (`__init__`) are not mangled.

---

## 8. Which one should you use?

- Default to **public** names (`self.addr`). Most Python code does this.
- Use a single underscore (`self._queue`) for attributes and helper methods that are implementation details and not meant to be used from outside.
- Use the double underscore rarely, only when you specifically need to avoid name clashes in inheritance.
- If you need to control or validate access to an attribute, Python offers `@property` (getters and setters that look like normal attributes). That is an optional next step beyond this series.

```python
# the usual Python style: simple public attributes, underscores for internals
class Monitor:
    def __init__(self):
        self.count = 0           # public
        self._seen = []          # internal

    def sample(self, item):      # public
        self._seen.append(item)
        self.count += 1
```

---

## 9. Common pitfalls

### Pitfall 1: forgetting the parentheses when calling a method

```python
p.show        # bound method object, nothing runs
p.show()      # runs it
```

### Pitfall 2: adding parentheses to an attribute

```python
p.addr()      # TypeError: 'int' object is not callable
```

Attributes are values, not functions.

### Pitfall 3: typos create new attributes silently

```python
p.adrr = 0x30        # typo: creates a NEW attribute, p.addr is unchanged
print(p.addr)        # still the old value
```

Because Python lets you create attributes by assignment, a misspelled name does not raise an error. This is a good reason to define every field in the constructor and be careful when writing to attributes.

### Pitfall 4: treating `_name` as a lock

A leading underscore is a request, not a barrier. Do not rely on it for protection, and do not ignore it in other people's code unless you have a good reason.

### Pitfall 5: expecting `__name` to be unreachable

Mangling renames the attribute, it does not hide it. `obj._Class__name` still works.

### Pitfall 6: accessing an object attribute through the class

```python
Packet.addr    # AttributeError: only objects have addr, the class does not
```

---

## 10. Why this matters for PyUVM

- All component code uses the dot operator constantly: `self.dut.clk`, `self.ap.write(item)`, `self.logger.info(...)`, `item.addr`.
- Because Python does not enforce privacy, testbench code can reach into any component. The **conventions** are what keep a testbench organized: public names for the interface other components use, and `_names` for internals.
- If you are coming from SystemVerilog, where `local` and `protected` are enforced by the compiler, expect more freedom in Python and more responsibility on you to follow the convention.
- Typos in attribute names do not raise errors on assignment (pitfall 3), so a misspelled field in a driver can silently create a new attribute and cause a confusing bug. Defining all fields in `__init__` or `build_phase` keeps this under control.

---

**Next up:** Part 6, Inheritance.

