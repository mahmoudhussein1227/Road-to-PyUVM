# Road to PyUVM, cocotb Part 4: Accessing DUT Signals with `.value`

In Part 9 we learned how a test waits for the simulation. Now we need the other half of a testbench: **driving the design's inputs and reading its outputs**. In cocotb, every signal is reached through the `dut` object, and its value is read and written through a single property: **`.value`**.

**Topics covered**

1. [The `dut` object and signal handles](#1-the-dut-object-and-signal-handles)
2. [Handle vs value](#2-handle-vs-value)
3. [Reading a signal](#3-reading-a-signal)
4. [Writing a signal](#4-writing-a-signal)
5. [When does a write take effect?](#5-when-does-a-write-take-effect)
6. [Full example](#6-full-example)
7. [Common pitfalls](#7-common-pitfalls)
8. [Why this matters for PyUVM](#8-why-this-matters-for-pyuvm)

---

## 1. The `dut` object and signal handles

Every cocotb test receives one argument, `dut` (Part 8). It is a **handle to the toplevel module** of your design. Each port, signal, and submodule inside it is reachable through the **dot operator**, exactly like the attribute access from Part 5:

```python
dut.clk          # handle to the clk signal
dut.rst          # handle to the rst signal
dut.count        # handle to the count output
```

A **handle** is a Python object that *refers to* an object inside the simulator. It does not contain the signal's value. It is more like a remote control: it knows which signal it points to, and through it you can read or write that signal.

The names must match the **HDL names exactly**, including capitalization. If the design has `output reg [3:0] count`, then `dut.count` is the handle.

Handles are what you pass to triggers, as we did in Part 9:

```python
await RisingEdge(dut.clk)      # the trigger takes a handle
```

---

## 2. Handle vs value

This is the single most important distinction in this part:

| Expression | What it is |
|---|---|
| `dut.count` | The **handle**: a reference to the signal |
| `dut.count.value` | The **value** currently on the signal |

```python
h = dut.count            # alias to the handle, no value is read
v = dut.count.value      # reads the current value
```

Forgetting `.value` is one of the most common beginner mistakes: you end up comparing or printing a handle instead of a value (see the pitfalls).

---

## 3. Reading a signal

Use `.value` on the right-hand side of an expression:

```python
value = dut.count.value
dut._log.info(f"count = {value}")
```

### What type do you get back?

In cocotb 2.x, the Python type depends on the signal:

| Signal | `.value` returns |
|---|---|
| A vector of logic bits (`[3:0] count`) | `LogicArray` |
| A single bit (`clk`) | `Logic` |
| An integer-typed signal (VHDL `integer`, SV `int`) | a Python `int` |

The reason is that hardware signals are **four-state**: each bit can be `0`, `1`, `X` (unknown), or `Z` (high impedance). A plain Python `int` cannot represent `X` or `Z`, so cocotb returns a `LogicArray`, which can.

### Converting to a Python number

```python
raw = dut.count.value              # LogicArray, for example 0101
n = raw.to_unsigned()              # 5, as a Python int
s = raw.to_signed()                # interpret the bits as two's complement
b = raw.to_bytes(byteorder="little")
text = str(raw)                    # '0101'
```

Converting a value that contains `X` or `Z` to a number is an error, because it has no numeric value. This is a useful alarm: reading `X` right after time 0, before reset, is normal, and the failed conversion tells you the design has not been initialized.

### Comparing

A `LogicArray` can be compared directly with a Python int:

```python
if dut.count.value == 5:
    ...
```

### Width and bits

```python
len(dut.count)                     # 4, the width of the signal in bits
dut.count.value[0]                 # the least significant bit
dut.count.value[3:1]               # a slice of bits
```

---

## 4. Writing a signal

Use `.value` on the **left-hand side** of an assignment:

```python
dut.rst.value = 1
dut.data_in.value = 0xFF
dut.data_in.value = 255            # same thing
```

Notes:

- Assign a normal Python `int`. cocotb makes no assumption about signed or unsigned: both are written as the bit pattern.
- A value that does not fit the signal width raises a **`ValueError`**. For a 4-bit signal, `= 16` is an error.
- To drive `X` or `Z` on purpose, use `Logic` or `LogicArray` objects:

```python
from cocotb.types import Logic, LogicArray

dut.data_in.value = Logic("X")
dut.bus.value = LogicArray("01XZ")
```

- Normal assignment is a **deposit**: it sets the value, and the design (or a later assignment) can change it afterward. cocotb also provides `Force`, `Freeze`, and `Release` for overriding signals, which you rarely need when you are starting out.

You can write only to signals that make sense to drive: typically the design's **inputs**. Writing to an output that the design itself drives will fight with the design's own logic.

---

## 5. When does a write take effect?

A write through `.value` is **not applied immediately**. Like assignments in HDL, it is scheduled and applied at the **next write cycle**, when your coroutine yields control by awaiting something.

```python
dut.rst.value = 1
dut._log.info(f"{dut.rst.value}")    # still shows the OLD value

await Timer(1, "ns")                 # the coroutine yields, the write is applied

dut._log.info(f"{dut.rst.value}")    # now shows 1
```

The practical rules:

- A read right after a write, with no `await` in between, gives the **old** value. This is normal, not a bug.
- Several writes in a row before an `await` are all applied together, at the same time step. That is handy for driving a whole interface at once.

```python
dut.valid.value = 1
dut.data.value = 0xA5
dut.addr.value = 0x10
await RisingEdge(dut.clk)            # all three take effect together
```

If you really need a write applied right away, cocotb offers `Immediate`:

```python
from cocotb.handle import Immediate

dut.rst.value = Immediate(1)
```

Use it sparingly. The default scheduled behavior is what makes testbench timing predictable.

---

## 6. Full example

We reuse the 4-bit counter from Part 8 (`my_design.v` with `clk`, `rst`, and `count`) and the same Makefile, with the test module renamed:

```make
COCOTB_TEST_MODULES = test_signals
```

### `test_signals.py`

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, FallingEdge, ClockCycles


@cocotb.test()
async def signals_demo(dut):
    """Write inputs, read outputs"""

    # Start a clock in the background (covered in the next part)
    cocotb.start_soon(Clock(dut.clk, 10, "ns").start())

    # ---- WRITE: drive the reset input ----
    dut.rst.value = 1
    await ClockCycles(dut.clk, 2)          # the write takes effect while we wait
    dut.rst.value = 0

    # ---- READ: sample the counter output ----
    await ClockCycles(dut.clk, 5)
    await FallingEdge(dut.clk)             # sample when the outputs are stable

    raw = dut.count.value                  # a LogicArray
    first = raw.to_unsigned()              # a Python int
    dut._log.info(f"count = {raw} (binary), {first} (int), width = {len(dut.count)} bits")

    # ---- READ again one cycle later ----
    await RisingEdge(dut.clk)
    await FallingEdge(dut.clk)
    second = dut.count.value.to_unsigned()
    dut._log.info(f"one cycle later: {second}")

    # A first taste of checking (assertions are covered later in the series)
    assert second == (first + 1) % 16, f"expected {(first + 1) % 16}, got {second}"
```

What to look at:

- `dut.rst.value = 1` is a **write**, `dut.count.value` is a **read**.
- The test samples on the **falling edge**, as in Part 9, so the counter's registered output has settled.
- `to_unsigned()` turns the four-state `LogicArray` into a plain Python `int` that we can do arithmetic on.
- The counter is 4 bits wide, so we compare against `(first + 1) % 16` to handle the wrap-around from 15 to 0.
- The `assert` simply fails the test if the counter does not advance by one. We cover assertions properly in a later part.

---

## 7. Common pitfalls

### Pitfall 1: forgetting `.value`

```python
if dut.count == 5:            # compares a handle, not the signal's value
    ...
if dut.count.value == 5:      # correct
    ...
```

`print(dut.count)` shows information about the handle, not the number.

### Pitfall 2: expecting a write to take effect immediately

```python
dut.rst.value = 1
print(dut.rst.value)          # old value: the write has not been applied yet
```

Wait on a trigger first, or use `Immediate` if you truly need it.

### Pitfall 3: reading right after a clock edge

As in Part 9, registered outputs may still show their pre-edge values right after `await RisingEdge`. Sample on the falling edge or after `ReadOnly()`.

### Pitfall 4: converting `X` or `Z` to a number

Before reset, a register is `X`. Calling `to_unsigned()` on it raises an error. Apply reset first, or check the value before converting.

### Pitfall 5: value too large for the signal

```python
dut.count.value = 16          # ValueError: does not fit in 4 bits
```

### Pitfall 6: a wrong or misspelled signal name

```python
dut.cnt.value                 # AttributeError: no signal named cnt
```

Names must match the HDL exactly, including capitalization.

### Pitfall 7: writing to a signal the design drives

Writing to an output, or to a wire driven inside the design, competes with the design's own logic and gives confusing results. Drive inputs.

### Pitfall 8: writing after `ReadOnly()`

As in Part 9, writes are not allowed after `await ReadOnly()` in the same time step. Move on to a later time step first.

### Pitfall 9: mixing old and new API names

`.integer` and `.binstr` are cocotb 1.x. In 2.x, use `to_unsigned()` and `str(...)`.

---

## 8. Why this matters for PyUVM

- A PyUVM **driver** writes to the design's inputs through `.value`, for example `self.dut.valid.value = 1` and `self.dut.data.value = item.data`.
- A PyUVM **monitor** reads the design through `.value`, usually after awaiting a clock edge, and builds a transaction object from the values it sees.
- Components get the `dut` handle from the **configuration database**. The usual pattern is to store `cocotb.top` in the ConfigDB in the test, and have each driver and monitor retrieve it in its `build_phase`, then save it as `self.dut`.
- Everything we covered here applies unchanged: handle vs value, scheduled writes, converting `LogicArray` to `int`, and sampling at a stable point.

---

**Next up:** Part 11, Running coroutines concurrently with `cocotb.start_soon()`, `Combine`, and `First`.
