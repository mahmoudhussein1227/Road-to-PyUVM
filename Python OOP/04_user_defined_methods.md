# Road to PyUVM, Part 4: User-Defined Methods

So far our classes hold data: class attributes, object attributes, and a constructor that sets them up. Methods are what give objects **behavior**. A method is how an object does something: sends a packet, checks a value, prints itself. This part covers what a method is, the rule that its first parameter is `self`, and how to call methods.

**Topics covered**

1. [What is a method?](#1-what-is-a-method)
2. [The `self` rule](#2-the-self-rule)
3. [Calling a method](#3-calling-a-method)
4. [Parameters and return values](#4-parameters-and-return-values)
5. [Methods that change the object's state](#5-methods-that-change-the-objects-state)
6. [Calling one method from another](#6-calling-one-method-from-another)
7. [Method vs function](#7-method-vs-function)
8. [Common pitfalls](#8-common-pitfalls)
9. [Why this matters for PyUVM](#9-why-this-matters-for-pyuvm)

---

## 1. What is a method?

A **method is a function defined inside a class**. It is written with the same `def` syntax as a normal function, but it lives in the class body and operates on the objects of that class.

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data

    def show(self):                       # a user-defined method
        print(f"addr={self.addr}, data={self.data}")

    def is_write(self):                   # another one
        return self.data != 0
```

"User-defined" means methods that **you** write, as opposed to special methods that Python calls automatically, such as `__init__` and `__str__` from Part 3. (Strictly, `__init__` is also a method; the difference is who calls it.)

From Part 1: method code is stored **once, in the class**, and shared by every object. What differs from call to call is which object the method works on.

---

## 2. The `self` rule

**The first parameter of every instance method must be `self`.** It receives the object the method is called on.

```python
class Packet:
    def show(self):             # self is the first parameter
        print(self.addr)
```

Why is it required? Because a method needs a way to reach the object's own data. Without a parameter to receive the object, the method would have no idea which object's `addr` to read.

```python
p1.show()    # self is p1  -> prints p1's addr
p2.show()    # self is p2  -> prints p2's addr
```

Same code, different object, different result.

Key points:

- `self` must come **first**, before any other parameter: `def send(self, delay)`.
- You **do not pass it yourself** when calling. Python passes the object automatically.
- Through `self` you read and write object attributes (`self.addr`) and call other methods (`self.show()`).
- `self` is not a reserved word. It is the universal naming convention for this first parameter, and every Python and PyUVM code base uses it. Always name it `self`.

---

## 3. Calling a method

Use the object, a dot, the method name, and parentheses:

```python
p = Packet(0x10, 0xFF)
p.show()                  # addr=16, data=255
print(p.is_write())       # True
```

### What Python does behind the scenes

`p.show()` is translated to:

```python
Packet.show(p)
```

The object before the dot becomes the first argument, which `self` receives. Both forms do the same thing, but the first is the one you write.

This also explains the classic error message. The call `p.show()` passes **one** argument (the object) even though you typed none:

```python
class Packet:
    def show():               # forgot self
        print("hi")

p = Packet()
p.show()    # TypeError: show() takes 0 positional arguments but 1 was given
```

### Bound methods

When you access `p.show` without calling it, Python gives you a **bound method**: the function already tied to the object `p`.

```python
f = p.show          # no parentheses: not called yet
f()                 # works, and self is still p
```

That is how the object travels along with the method.

---

## 4. Parameters and return values

After `self`, methods take parameters and return values like any function.

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data

    def matches(self, other_addr):            # extra parameter after self
        return self.addr == other_addr

    def to_hex(self):                         # returns a value
        return f"0x{self.addr:X}:0x{self.data:X}"
```

```python
p = Packet(0x10, 0xFF)
print(p.matches(0x10))     # True
print(p.to_hex())          # 0x10:0xFF
```

Remember: `self` is **not** counted among the arguments you pass. `p.matches(0x10)` passes one argument; `self` is filled in for you.

Default values and keyword arguments work normally:

```python
def set_fields(self, addr=0, data=0):
    self.addr = addr
    self.data = data

p.set_fields(data=0xAA)    # addr keeps its default
```

A method with no `return` statement returns `None`.

---

## 5. Methods that change the object's state

Methods are the clean way to modify an object's attributes.

```python
class Counter:
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1

    def reset(self):
        self.value = 0
```

```python
c = Counter()
c.increment()
c.increment()
print(c.value)    # 2
c.reset()
print(c.value)    # 0
```

The change is stored on the object (`self.value`), so it persists after the method returns. A plain local variable inside the method would disappear.

Two objects keep independent state:

```python
a = Counter()
b = Counter()
a.increment()
print(a.value, b.value)    # 1 0
```

---

## 6. Calling one method from another

Inside a class, a method calls another method of the same object through `self`:

```python
class Packet:
    def __init__(self, addr, data):
        self.addr = addr
        self.data = data

    def to_hex(self):
        return f"0x{self.addr:X}:0x{self.data:X}"

    def show(self):
        print("Packet", self.to_hex())     # self.to_hex(), not just to_hex()
```

Writing `to_hex()` without `self.` fails with `NameError`, because Python looks for a plain function named `to_hex`, not the method on the current object.

This is also how you break a long task into small methods: one big method calls several smaller ones.

---

## 7. Method vs function

| | Function | Method |
|---|---|---|
| Defined | At module level | Inside a class |
| Called | `name(args)` | `obj.name(args)` |
| First parameter | Any | `self` (the object) |
| Has access to object data | Only if you pass it in | Yes, through `self` |
| Belongs to | The module | The class |

```python
def show_packet(p):          # function: receives the object explicitly
    print(p.addr)

class Packet:
    def show(self):          # method: receives the object as self
        print(self.addr)

show_packet(p)    # function call
p.show()          # method call
```

The idea is the same; methods just bundle the behavior with the data it works on. This is the core of object-oriented design: data and the operations on it live together.

### A quick note on other kinds of methods

Everything above is an **instance method**: it works on one object through `self`. Python also has `@classmethod` (receives the class as `cls`) and `@staticmethod` (receives neither). They are outside this series' scope, and you will not need them to start with PyUVM.

---

## 8. Common pitfalls

### Pitfall 1: forgetting `self` in the definition

```python
def show():           # TypeError when called: takes 0 positional arguments but 1 was given
```

Fix: `def show(self):`.

### Pitfall 2: forgetting `self.` inside the method

```python
class Packet:
    def show(self):
        print(addr)        # NameError: name 'addr' is not defined
```

Fix: `print(self.addr)`. Without `self.`, Python looks for a local or global variable, not an object attribute.

### Pitfall 3: passing `self` when calling

```python
p.show(p)     # TypeError: show() takes 1 positional argument but 2 were given
```

Python already passes the object. Just write `p.show()`.

### Pitfall 4: forgetting the parentheses

```python
p.show        # not an error, but nothing runs: this is just the bound method object
p.show()      # this calls it
```

### Pitfall 5: forgetting to `return`

```python
def to_hex(self):
    f"0x{self.addr:X}"        # computed, then thrown away

print(p.to_hex())             # None
```

Fix: `return f"0x{self.addr:X}"`. This is the same mistake as printing instead of returning in `__str__`.

### Pitfall 6: forgetting `self.` when calling another method

```python
def show(self):
    to_hex()                  # NameError
    self.to_hex()             # correct
```

---

## 9. Why this matters for PyUVM

Nearly everything you write in a PyUVM testbench is a method on a component class. The UVM phases are methods that the framework calls for you, in order, with `self` as the first parameter:

```python
class MyDriver(uvm_driver):
    def build_phase(self):
        self.count = 0

    async def run_phase(self):          # async methods: covered in the cocotb section
        ...
```

- `build_phase`, `connect_phase`, `run_phase` and the other phases are **user-defined methods** that you fill in.
- Helper methods such as `drive_item(self, item)` keep a driver readable, with `self` giving access to the interface handle and counters.
- Inside any method, `self.something` is how components reach their own state, exactly as in Parts 2 and 3.
- The `async def` methods you will see in `run_phase` follow the same `self` rule. They are the bridge to the cocotb coroutines coming up in the next section of the series.

---

**Next up:** Part 5, Accessing class members.
