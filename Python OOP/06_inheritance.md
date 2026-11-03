# Road to PyUVM, Part 6: Inheritance

In Parts 1 to 5 we learned to build a class and use its objects. Inheritance is the last piece of Python OOP you need before PyUVM, because **every PyUVM component you write inherits from a class provided by the library**: `class MyDriver(uvm_driver)`. This part explains what that line means, how `super()` works, and how a child class changes what it inherits.

**Topics covered**

1. [What is inheritance?](#1-what-is-inheritance)
2. [Declaring a child class](#2-declaring-a-child-class)
3. [Inherited attributes and methods](#3-inherited-attributes-and-methods)
4. [Extending the constructor with `super()`](#4-extending-the-constructor-with-super)
5. [Method overriding](#5-method-overriding)
6. [Extending a parent method with `super()`](#6-extending-a-parent-method-with-super)
7. [Checking relationships: `isinstance` and `issubclass`](#7-checking-relationships-isinstance-and-issubclass)
8. [How Python looks up a member](#8-how-python-looks-up-a-member)
9. [Multiple inheritance: supported in Python, unlike SystemVerilog](#9-multiple-inheritance-supported-in-python-unlike-systemverilog)
10. [Common pitfalls](#10-common-pitfalls)
11. [Why this matters for PyUVM](#11-why-this-matters-for-pyuvm)

---

## 1. What is inheritance?

Inheritance lets one class **reuse and extend another class** instead of rewriting it.

- The class being inherited from is the **parent** (also called *base class* or *superclass*).
- The class that inherits is the **child** (also called *derived class* or *subclass*).

If you come from SystemVerilog, this is the same idea as `class Child extends Parent`. The big difference, covered in section 9, is that Python allows a class to have **more than one** parent.

The relationship is an **"is-a"** relationship: a `WriteTransaction` *is a* `Transaction`, just a more specialized one. The child automatically gets everything the parent has, and can then add new things or change existing ones.

---

## 2. Declaring a child class

Put the parent class name in parentheses after the child's name:

```python
class Transaction:                       # parent
    def __init__(self, addr):
        self.addr = addr

    def show(self):
        print(f"addr=0x{self.addr:X}")

class WriteTransaction(Transaction):     # child: inherits from Transaction
    pass
```

Even with an empty body, the child already works:

```python
w = WriteTransaction(0x10)
print(w.addr)     # 16
w.show()          # addr=0x10
```

`WriteTransaction` has no constructor and no `show` of its own, yet both work, because they are inherited from `Transaction`.

Every Python class ultimately inherits from the built-in class `object`, even when you do not write it. `class Transaction:` is the same as `class Transaction(object):`.

---

## 3. Inherited attributes and methods

The child gets the parent's **methods** and **class attributes** directly. Object attributes (the ones created through `self`) appear on the child's objects when the parent's constructor runs.

```python
class Transaction:
    bus = "AXI"                         # class attribute

    def __init__(self, addr):
        self.addr = addr                # object attribute

    def show(self):
        print(f"addr=0x{self.addr:X}")

class WriteTransaction(Transaction):
    pass

w = WriteTransaction(0x10)
print(w.bus)       # AXI    (inherited class attribute)
print(w.addr)      # 16     (set by the inherited constructor)
w.show()           # inherited method
```

The child can also add its own members:

```python
class WriteTransaction(Transaction):
    def is_write(self):                 # new method, only on the child
        return True
```

---

## 4. Extending the constructor with `super()`

Usually the child needs extra fields of its own. If you define `__init__` in the child, it **replaces** the parent's constructor, so the parent's fields will not be created unless you call it explicitly. That is what `super()` is for.

`super()` gives access to the parent class from inside the child. (Like `self`, it is not a reserved keyword: `super` is a built-in function, but it is used so universally that it reads like one.)

```python
class WriteTransaction(Transaction):
    def __init__(self, addr, data):
        super().__init__(addr)          # run the parent's constructor first
        self.data = data                # then add the child's own field

w = WriteTransaction(0x10, 0xFF)
print(w.addr)     # 16   (set by Transaction.__init__)
print(w.data)     # 255  (set by WriteTransaction.__init__)
```

Rules of thumb:

- Call `super().__init__(...)` and pass the arguments the parent expects.
- Do **not** pass `self`: `super()` handles it.
- Calling it first is the usual style, so the parent's fields exist before the child uses them.

---

## 5. Method overriding

A child can **redefine** a method it inherited. When the method is called on a child object, Python uses the child's version.

```python
class Transaction:
    def show(self):
        print(f"addr=0x{self.addr:X}")

class WriteTransaction(Transaction):
    def show(self):                      # overrides Transaction.show
        print(f"WRITE addr=0x{self.addr:X}, data=0x{self.data:X}")
```

```python
t = Transaction(0x10)
w = WriteTransaction(0x20, 0xFF)

t.show()    # addr=0x10
w.show()    # WRITE addr=0x20, data=0xFF
```

Same method name, different behavior depending on the object's class. This is the basis of **polymorphism**: code that calls `item.show()` works for any kind of transaction without checking which one it has.

The same applies to dunder methods. Overriding `__str__` in the child changes how `print(w)` looks (Part 3).

---

## 6. Extending a parent method with `super()`

Sometimes you do not want to replace the parent's behavior, only add to it. Call the parent's version through `super()` inside the override:

```python
class WriteTransaction(Transaction):
    def __init__(self, addr, data):
        super().__init__(addr)
        self.data = data

    def show(self):
        super().show()                   # parent's version: prints addr
        print(f"data=0x{self.data:X}")   # then add the child's part
```

```python
w = WriteTransaction(0x20, 0xFF)
w.show()
# addr=0x20
# data=0xFF
```

This **override-and-extend** pattern is exactly what you do with UVM phases in PyUVM.

---

## 7. Checking relationships: `isinstance` and `issubclass`

```python
w = WriteTransaction(0x20, 0xFF)

print(isinstance(w, WriteTransaction))   # True
print(isinstance(w, Transaction))        # True   (a child object IS a parent type)
print(isinstance(t, WriteTransaction))   # False  (the reverse is not true)

print(issubclass(WriteTransaction, Transaction))   # True
```

`isinstance` follows the inheritance chain, which is why it is preferred over comparing `type(w) == Transaction`. That comparison would be `False` for a child object.

You can also inspect the chain:

```python
print(WriteTransaction.__mro__)
# (<class 'WriteTransaction'>, <class 'Transaction'>, <class 'object'>)
```

---

## 8. How Python looks up a member

In Part 2 we saw the lookup order for `obj.name`: the object, then its class. With inheritance, the search continues up the chain of parents:

1. The object itself (`obj.__dict__`)
2. The object's class
3. The parent class, then its parent, and so on up to `object`

```
 w.show()
    │
    ├─ in w.__dict__ ?                no
    ├─ in WriteTransaction ?          yes -> use it (override wins)
    └─ (otherwise) in Transaction ?   then in object ?
```

The first match wins. That is why an overriding method in the child hides the parent's method of the same name, and why `super()` is needed to reach the parent's version.

---

## 9. Multiple inheritance: supported in Python, unlike SystemVerilog

A Python class can inherit from **more than one parent**. List them all in the parentheses:

```python
class Logger:
    def log(self, msg):
        print(f"[LOG] {msg}")

class Checker:
    def check(self, ok):
        print("PASS" if ok else "FAIL")

class Scoreboard(Logger, Checker):      # two parents
    pass

sb = Scoreboard()
sb.log("comparing")      # from Logger
sb.check(True)           # from Checker
```

`Scoreboard` gets the methods of **both** parents.

### Compared with SystemVerilog

A SystemVerilog class can `extends` only **one** class (single inheritance). The workaround in SV is interface classes (added in SystemVerilog-2012), where a class can `implements` several of them, but those only declare method prototypes and carry no implementation. In Python, a class can inherit real attributes and real method code from several parents.

| | SystemVerilog | Python |
|---|---|---|
| Parents of a class | One (`extends`) | One or many |
| Reusing code from several classes | Not directly (interface classes give only prototypes) | Yes, list them all |

### Method resolution order (MRO)

When a method exists in more than one parent, Python needs a rule to pick one. The rule is the **method resolution order**, which you can print:

```python
print(Scoreboard.__mro__)
# (<class 'Scoreboard'>, <class 'Logger'>, <class 'Checker'>, <class 'object'>)
```

Python searches this list from left to right and uses the first match. So the order of the parents in `class Scoreboard(Logger, Checker)` matters: if both defined `log`, `Logger`'s version would win.

### `super()` follows the MRO

`super()` does not simply mean "the one parent". It means "the next class in the MRO". This makes cooperative designs work even in a diamond:

```python
class A:
    def hello(self):
        print("A")

class B(A):
    def hello(self):
        print("B")
        super().hello()

class C(A):
    def hello(self):
        print("C")
        super().hello()

class D(B, C):
    def hello(self):
        print("D")
        super().hello()

D().hello()
# D
# B
# C
# A          <- A runs once, even though both B and C inherit from it
print(D.__mro__)    # D, B, C, A, object
```

### Keep it simple

Multiple inheritance is powerful but easy to overuse. Most testbench code uses single inheritance, and the common safe use of multiple inheritance is a small **mixin**: a helper class that adds one focused ability (like logging or checking) and has no state of its own.

---

## 10. Common pitfalls

### Pitfall 1: overriding `__init__` without calling the parent's

```python
class WriteTransaction(Transaction):
    def __init__(self, addr, data):
        self.data = data              # forgot super().__init__(addr)

w = WriteTransaction(0x10, 0xFF)
print(w.addr)    # AttributeError: 'WriteTransaction' object has no attribute 'addr'
```

The child's constructor replaced the parent's, so `addr` was never created.

### Pitfall 2: wrong arguments to the parent's constructor

```python
super().__init__()    # TypeError: __init__() missing 1 required positional argument: 'addr'
```

Pass what the parent's `__init__` expects.

### Pitfall 3: passing `self` to `super()`

```python
super().__init__(self, addr)    # wrong: self is passed twice
super().__init__(addr)          # correct
```

### Pitfall 4: silently replacing instead of extending

An override that does not call `super()` removes the parent's behavior entirely. That is sometimes what you want, but when the parent does important setup (as in PyUVM phases), forgetting `super()` can break the framework in ways that are hard to trace.

### Pitfall 5: misspelling the method name when overriding

```python
class WriteTransaction(Transaction):
    def shwo(self):           # typo: this is a NEW method, not an override
        ...
```

Python raises no error. The parent's `show` keeps being used.

### Pitfall 6: the order of parents matters

```python
class Scoreboard(Logger, Checker): ...   # Logger's methods win on a name clash
class Scoreboard(Checker, Logger): ...   # Checker's methods win
```

If two parents define the same method name, the one listed first in the parentheses is used. Print `__mro__` when in doubt.

### Pitfall 7: using inheritance when it is not an "is-a" relationship

A scoreboard *has a* queue, it is not *a* queue. If the relationship is "has-a", store the object as an attribute (composition) instead of inheriting.

---

## 11. Why this matters for PyUVM

Inheritance is the backbone of PyUVM. The library provides base classes, and you build your testbench by inheriting from them and filling in the parts that are specific to your design:

```python
class MyDriver(uvm_driver):
    def __init__(self, name, parent):
        super().__init__(name, parent)       # let the base class set itself up
        self.count = 0

    def build_phase(self):
        super().build_phase()                # keep the base class behavior
        ...

    async def run_phase(self):               # override the phase with your own code
        ...
```

- `class MyDriver(uvm_driver)` means `MyDriver` is a `uvm_driver`, so it inherits everything a driver has.
- The constructor takes `name` and `parent` and passes them up with `super().__init__(name, parent)`. Forgetting this is one of the most common beginner errors.
- Phase methods such as `build_phase`, `connect_phase`, and `run_phase` are **overridden**: the framework calls them by name, and your version runs. Calling `super().build_phase()` first keeps the base-class work intact.
- Your transactions inherit from a base sequence-item class in the same way, which is how they pick up the features PyUVM expects.
- Unlike SystemVerilog, Python would let a component also inherit from small helper classes (mixins). Most PyUVM code sticks to single inheritance, but it is good to know the option exists.
- Because `isinstance` follows the chain, a monitor, driver, or scoreboard can be treated generically as a component by the framework.

---

**Next up:** the cocotb section, starting with coroutines and `@cocotb.test()`.

