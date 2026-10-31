# Road to PyUVM, Part 3: Constructors and the `__str__` Dunder Method

In Part 2 we created object attributes by assigning them from outside the class or from ordinary methods, and we saw the trap of reading an attribute that was never set. The **constructor** solves that: it is the standard place to give every object its fields as soon as it is created. This part also introduces **dunder methods** and one of the most useful ones, `__str__`, which turns an object into readable text.

**Topics covered**

1. [The constructor: `__init__`](#1-the-constructor-__init__)
2. [When is it called?](#2-when-is-it-called)
3. [Parameters and default values](#3-parameters-and-default-values)
4. [Dunder methods](#4-dunder-methods)
5. [`__str__`: object to string](#5-__str__-object-to-string)
6. [`__str__` vs `__repr__`](#6-__str__-vs-__repr__)
7. [Common pitfalls](#7-common-pitfalls)
8. [Why this matters for PyUVM](#8-why-this-matters-for-pyuvm)
9. [Quick exercises](#9-quick-exercises)

---

## 1. The constructor: `__init__`

The constructor is the method Python calls on a new object to **initialize its fields**. In Python it is always named `__init__`.

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data
```

Creating an object now passes the values straight in:

```python
p1 = Packet(0x10, 0xFF)
p2 = Packet(0x20, 0xAA)

print(p1.addr, p1.data)    # 16 255
print(p2.addr, p2.data)    # 32 170
```

Compare this with Part 2: every object is guaranteed to have `addr` and `data` from the moment it exists, so no more `AttributeError` from a field that was never set.

Notes:

- The first parameter is `self`, the object being initialized (see Part 2).
- The other parameters are the values you pass when creating the object: `Packet(0x10, 0xFF)`.
- `self.addr = addr` creates an **object attribute** and stores the value in it.
- `__init__` should not return anything (it returns `None`). Returning another value raises a `TypeError`.

---

## 2. When is it called?

You never call `__init__` yourself. Python calls it automatically, **right after the object has been created**, as the first method that runs on the new object.

Recall from Part 1 that `Packet(...)` does two steps:

1. `__new__` allocates a new, empty object in memory.
2. `__init__` runs on that object to fill in its fields.

```python
class Packet:
    def __init__(self, addr, data):
        print("__init__ is running")
        self.addr = addr
        self.data = data

print("before")
p = Packet(0x10, 0xFF)      # prints "__init__ is running"
print("after")
```

Output:

```
before
__init__ is running
after
```

`__new__` creates the object, while `__init__` is where you set it up. In day-to-day code you only write `__init__`.

### Without a constructor

If a class does not define `__init__`, Python uses a default one that does nothing, so `Packet()` still works but the object starts with no object attributes. That is the situation we had in Parts 1 and 2.

---

## 3. Parameters and default values

Constructor parameters work like any function parameters, including defaults and keyword arguments.

```python
class Packet:
    def __init__(self, addr=0, data=0, protocol="AXI"):
        self.addr = addr
        self.data = data
        self.protocol = protocol

p1 = Packet()                         # all defaults
p2 = Packet(0x10, 0xFF)               # positional
p3 = Packet(addr=0x20, protocol="APB")  # keyword, data keeps its default
```

You can also compute fields instead of just copying parameters:

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data
        self.size = len(bin(data)) - 2    # derived field
        self.history = []                 # each object gets its own list
```

Creating the list in `__init__` fixes the shared-list pitfall from Part 2: every object gets a fresh list.

---

## 4. Dunder methods

A **dunder method** (short for "double underscore", also called a *magic method* or *special method*) is a method whose name starts and ends with two underscores, such as `__init__` or `__str__`.

You do not call them directly. Python calls them for you when you use certain syntax or built-in functions:

| You write | Python calls |
|---|---|
| `Packet(0x10, 0xFF)` | `__init__` (after `__new__`) |
| `print(p)` or `str(p)` | `__str__` |
| `repr(p)` | `__repr__` |
| `p1 == p2` | `__eq__` |
| `len(p)` | `__len__` |

`__init__` is the dunder you have already met. `__str__` is the next one, and probably the most useful one for debugging.

---

## 5. `__str__`: object to string

By default, printing an object tells you almost nothing:

```python
p = Packet(0x10, 0xFF)
print(p)    # <__main__.Packet object at 0x7f3c2a1b5d30>
```

`__str__` lets you decide what that text is. It should **return a string** that describes the object's fields.

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data

    def __str__(self):
        return f"Packet(addr=0x{self.addr:X}, data=0x{self.data:X})"
```

```python
p = Packet(0x10, 0xFF)
print(p)                    # Packet(addr=0x10, data=0xFF)
```

Python calls `__str__` in all of these cases:

```python
print(p)                    # Packet(addr=0x10, data=0xFF)
text = str(p)               # "Packet(addr=0x10, data=0xFF)"
print(f"Sending {p}")       # Sending Packet(addr=0x10, data=0xFF)
```

Rules:

- It must **return** a string, not print one. Printing inside `__str__` and returning nothing causes a `TypeError` when you call `print(p)`.
- It should describe the object's current field values, which makes logs and error messages much easier to read.

---

## 6. `__str__` vs `__repr__`

`__repr__` is the sibling method meant for developers: an unambiguous description, ideally one that shows how to recreate the object.

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data

    def __str__(self):                  # readable, for users and logs
        return f"Packet(addr=0x{self.addr:X}, data=0x{self.data:X})"

    def __repr__(self):                 # precise, for developers and debugging
        return f"Packet({self.addr}, {self.data})"
```

| | `__str__` | `__repr__` |
|---|---|---|
| Used by | `print()`, `str()`, f-strings | `repr()`, the interactive prompt, **printing containers** |
| Goal | Readable | Unambiguous |
| If missing | Falls back to `__repr__` | Default `<... object at 0x...>` |

A common surprise is printing a list of objects. Lists display their items using `__repr__`, not `__str__`:

```python
packets = [Packet(1, 2), Packet(3, 4)]
print(packets)    # [Packet(1, 2), Packet(3, 4)]   -> uses __repr__
```

If you only define `__str__`, the list will show the ugly default form. Defining `__repr__` as well fixes it. A reasonable habit is to define `__str__` at least, and add `__repr__` when you print collections.

---

## 7. Common pitfalls

### Pitfall 1: forgetting `self.` in the constructor

```python
class Packet:
    def __init__(self, addr, data):
        addr = addr        # only a local variable, nothing is stored on the object
        data = data
```

Then `Packet(1, 2).addr` raises `AttributeError`. Use `self.addr = addr`.

### Pitfall 2: misspelling the dunder name

```python
class Packet:
    def __innit__(self, addr):    # typo: Python never calls this
        self.addr = addr

p = Packet(5)    # TypeError: Packet() takes no arguments
```

The name must be exactly `__init__`, with two underscores on each side.

### Pitfall 3: printing instead of returning in `__str__`

```python
def __str__(self):
    print(f"addr={self.addr}")    # wrong: returns None
```

```python
def __str__(self):
    return f"addr={self.addr}"    # correct
```

### Pitfall 4: mutable default arguments

```python
class Monitor:
    def __init__(self, seen=[]):    # the same list is reused by every call
        self.seen = seen
```

Default values are evaluated once, when the function is defined, so every `Monitor()` shares one list. Use `None` and create the list inside:

```python
class Monitor:
    def __init__(self, seen=None):
        self.seen = [] if seen is None else seen
```

---

## 8. Why this matters for PyUVM

- Every PyUVM component and transaction has a constructor. Components take a `name` and a `parent`, and you set up your own fields there:

```python
class MyDriver(uvm_driver):
    def __init__(self, name, parent):
        super().__init__(name, parent)    # inheritance: covered later in the series
        self.count = 0
```

- Sequence items (transactions) define their fields in `__init__`, so every transaction is created with a known set of attributes.
- `__str__` is how you make a transaction printable. Logging `f"Driving {item}"` is only useful if `__str__` returns the field values.
- Debugging a testbench usually means reading logs, so a good `__str__` on your transactions pays off quickly.

---

## 9. Quick exercises

1. Write a `Transaction` class whose constructor takes `addr` and `data` and stores them as object attributes. Create two objects and print their fields.
2. Add default values so `Transaction()` works with no arguments.
3. Print a `Transaction` object before defining `__str__`, then add `__str__` and print again.
4. Add a `__repr__` and print a list of three transactions. Compare the output with and without it.
5. Put a `print` statement in `__init__` and observe exactly when it runs relative to the surrounding code.
6. Reproduce the mutable default argument bug with a list, then fix it using `None`.
7. Add a derived field, for example `self.is_write = (data != 0)`, computed inside the constructor.

---

**Next up:** Part 4, User-defined methods.
