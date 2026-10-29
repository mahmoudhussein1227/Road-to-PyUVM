# Road to PyUVM, Part 2: Attributes and `self`

In Part 1 we saw that a class is a blueprint and that every object is its own instance. This part covers what those objects actually *hold*: **attributes**. There are two kinds, class attributes and object (instance) attributes, and `self` is how a method refers to the object it is working on.

**Topics covered**

1. [Class attributes](#1-class-attributes)
2. [Object (instance) attributes](#2-object-instance-attributes)
3. [`self`: referencing the current object](#3-self-referencing-the-current-object)
4. [Common pitfalls](#5-common-pitfalls)
5. [Why this matters for PyUVM](#6-why-this-matters-for-pyuvm)


---

## 1. Class attributes

A class attribute is a variable defined **in the class body**, outside any method. It belongs to the class itself and is **shared by all objects**.

```python
class Packet:
    protocol = "AXI"        # class attribute
    count = 0               # class attribute
```

You can access it through the class or through any object:

```python
print(Packet.protocol)      # AXI
p1 = Packet()
p2 = Packet()
print(p1.protocol)          # AXI
print(p2.protocol)          # AXI
```

Changing it **through the class** changes it for everyone:

```python
Packet.protocol = "APB"
print(p1.protocol)          # APB
print(p2.protocol)          # APB
```

Use class attributes for values that are the same for every object: constants, defaults, counters shared by all instances.

---

## 2. Object (instance) attributes

An object attribute belongs to **one specific object**. Each object has its own copy, stored in that object only.

```python
p1.addr = 0x10
p2.addr = 0x20

print(p1.addr)    # 16
print(p2.addr)    # 32
```

Python does not require you to declare attributes up front. Assigning to `object.name` creates the attribute on that object if it does not exist yet.

```python
p1.data = 0xFF
print(hasattr(p1, "data"))    # True
print(hasattr(p2, "data"))    # False -> only p1 has it
```

Creating attributes from outside the class works, but it gets messy fast, because different objects can end up with different attributes. The clean way is to create them inside the class using `self`, which is the next section. The constructor (`__init__`) is the standard place to do it, and it gets its own post.

### Class attributes vs object attributes at a glance

| | Class attribute | Object attribute |
|---|---|---|
| Defined | In the class body | By assigning on an object, usually through `self` |
| Belongs to | The class | One specific object |
| Copies in memory | One, shared | One per object |
| Accessed via | `ClassName.x` or `obj.x` | `obj.x` |
| Typical use | Constants, defaults, shared counters | The state of each individual object |

You can see where each one lives with `__dict__`:

```python
print(Packet.__dict__.keys())   # includes 'protocol', 'count'
print(p1.__dict__)              # {'addr': 16, 'data': 255}
print(p2.__dict__)              # {'addr': 32}
```

---

## 3. `self`: referencing the current object

Methods usually need to work on the data of a particular object. `self` is how a method refers to **the object it was called on**.

```python
class Packet:
    protocol = "AXI"                 # class attribute

    def set_addr(self, addr):
        self.addr = addr             # object attribute, belongs to the current object

    def show(self):
        print(f"{self.protocol} packet, addr = {self.addr}")
```

```python
p1 = Packet()
p2 = Packet()

p1.set_addr(0x10)
p2.set_addr(0x20)

p1.show()    # AXI packet, addr = 16
p2.show()    # AXI packet, addr = 32
```

The same method code runs for both objects. `self` is what makes the results different: in `p1.set_addr(0x10)`, `self` is `p1`, and in `p2.set_addr(0x20)`, `self` is `p2`.

### What Python does behind the scenes

When you write `p1.show()`, Python translates it into:

```python
Packet.show(p1)
```

The object before the dot is passed automatically as the first argument. That is why every instance method must declare `self` as its first parameter, and why you do not pass it yourself when calling.

```python
p1.show()          # correct
Packet.show(p1)    # equivalent, rarely written this way
p1.show(p1)        # TypeError: show() takes 1 positional argument but 2 were given
```

### `self` is a convention, not a reserved word

`self` is not a keyword in Python. It is just the universal name for the first parameter, and every Python developer and every PyUVM code base expects it. Technically you could name it something else, but you should not. The behavior is what matters: the first parameter receives the current object.

### Reading and writing through `self`

| Expression | Meaning |
|---|---|
| `self.addr = 5` | Create or update the **object** attribute `addr` on the current object |
| `self.addr` | Read the object attribute (falls back to the class attribute if none exists) |
| `self.protocol` | Read the class attribute through the object |
| `self.__class__.count += 1` | Update the **class** attribute for all objects |

Example of a shared counter that tracks how many packets were created:

```python
class Packet:
    count = 0

    def register(self):
        self.__class__.count += 1    # class attribute, shared
        self.id = Packet.count       # object attribute, unique per object

a = Packet(); a.register()
b = Packet(); b.register()
print(a.id, b.id)       # 1 2
print(Packet.count)     # 2
```
---

## 4. Common pitfalls

### Pitfall 1: assigning through an object *shadows* the class attribute

```python
p1 = Packet()
p2 = Packet()

p1.protocol = "APB"        # does NOT change the class attribute

print(p1.protocol)         # APB  -> new object attribute on p1 only
print(p2.protocol)         # AXI  -> still the class attribute
print(Packet.protocol)     # AXI
```

Reading `obj.x` can fall back to the class, but assigning `obj.x = value` always creates or updates an **object** attribute. To change the shared value, assign through the class: `Packet.protocol = "APB"`.

### Pitfall 2: mutable class attributes are shared

```python
class Monitor:
    seen = []                # one list, shared by every Monitor

    def record(self, item):
        self.seen.append(item)

m1 = Monitor()
m2 = Monitor()
m1.record("pkt1")
print(m2.seen)               # ['pkt1']  -> surprise! same list
```

`append` mutates the shared list instead of assigning a new attribute. If each object needs its own list, create it per object through `self` (usually in the constructor):

```python
self.seen = []
```

### Pitfall 3: forgetting `self`

```python
class Packet:
    def set_addr(self, addr):
        addr = addr           # only changes a local variable, nothing is stored

    def show(self):
        print(addr)           # NameError: name 'addr' is not defined
```

Without `self.`, you are creating a plain local variable that disappears when the method ends. Use `self.addr` to store data on the object.

### Pitfall 4: reading an attribute that was never set

```python
p = Packet()
print(p.addr)    # AttributeError: 'Packet' object has no attribute 'addr'
```

This is one more reason to initialize every attribute up front in the constructor, which is the next topic.

---

## 5. Why this matters for PyUVM

- A driver or monitor stores its own state in **object attributes** (`self.dut`, `self.queue`). Each component instance has its own.
- Settings that are the same for every instance can be **class attributes**.
- A mutable class attribute, such as a list shared by every instance of a scoreboard, is a classic source of bugs where two components unexpectedly share data.
- Every PyUVM method you write, `build_phase`, `run_phase`, `connect_phase`, takes `self` as its first parameter, and you use `self.x = ...` to build up the component's state.

---


**Next up:** Part 3, Constructors and dunder methods (`__init__`, `__str__`).
