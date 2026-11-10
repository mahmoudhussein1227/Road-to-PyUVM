# Road to PyUVM, cocotb Part 3: Triggers and `await`

In cocotb Part 2 we saw that a coroutine can pause with `await`, and that awaiting another coroutine runs it to completion and gives back its return value. This part covers the **second kind of awaitable object**: the **trigger**. Triggers are how your Python test waits for things to happen in the simulation: time passing, a clock edge, a signal change.

**Topics covered**

1. [Two kinds of awaitable objects](#1-two-kinds-of-awaitable-objects)
2. [What is a trigger?](#2-what-is-a-trigger)
3. [What happens when you `await` a trigger](#3-what-happens-when-you-await-a-trigger)
4. [Common triggers](#4-common-triggers)
5. [Full example](#5-full-example)
6. [Triggers vs SystemVerilog](#6-triggers-vs-systemverilog)
7. [Why the timing of a trigger matters](#7-why-the-timing-of-a-trigger-matters)
8. [Common pitfalls](#8-common-pitfalls)
9. [Why this matters for PyUVM](#9-why-this-matters-for-pyuvm)

---

## 1. Two kinds of awaitable objects

Only **awaitable** objects can follow the `await` keyword. In the first part of cocotb, you will use two kinds:

| | Coroutine | Trigger |
|---|---|---|
| What it is | A function defined with `async def` | An object that represents a simulation event |
| What `await` does | Runs the coroutine until it finishes | Pauses until the event happens |
| Result of the `await` | The coroutine's return value (`var = await co()`) | Resumes the coroutine when the event occurs |
| Example | `await send_packet(pkt)` | `await RisingEdge(dut.clk)` |

Coroutines are the second pillar of cocotb's concurrency model, and triggers are what connect your Python code to the passing of simulation time. Almost every line of real testbench code is some mix of the two.

---

## 2. What is a trigger?

A **trigger** is an object that stands for a **simulator event or condition** that will happen at some point in the future, such as "10 ns from now" or "the next rising edge of `clk`".

When a coroutine awaits a trigger:

- It behaves like a **blocking statement**: the coroutine stops at that line and does not continue until the event happens.
- The coroutine **yields control**, either back to the **simulator** (so simulation time can advance) or to **another coroutine** that is ready to run.
- When the event occurs, the simulator tells cocotb, and cocotb **resumes** the coroutine on the next line.

"Blocking" here describes the coroutine only. The simulator is not blocked: it keeps running, and other coroutines keep making progress. This is the difference from `time.sleep()`, which would freeze the whole simulation (cocotb Part 2, pitfall 3).

---

## 3. What happens when you `await` a trigger

Take this line:

```python
await RisingEdge(dut.clk)
```

Step by step:

1. `RisingEdge(dut.clk)` creates a trigger object for "a 0-to-1 change on `dut.clk`".
2. `await` hands the trigger to cocotb, which **registers a callback** with the simulator through VPI (cocotb Part 1): "call me when `clk` rises".
3. The coroutine is **paused**. Control goes back to the simulator, or to any other coroutine that is ready to run.
4. The simulation continues. Eventually `clk` goes from 0 to 1.
5. The simulator calls the callback, cocotb **resumes** your coroutine, and the next line runs.

```
 your coroutine   : run ... await RisingEdge(dut.clk)           ... resume, run on
                                  |                                  ^
                                  | yields control                   | callback fires
                                  v                                  |
 simulator + any other coroutines :  time advances ... clk goes 0 -> 1
```

Remember from cocotb Part 2: the code between two `await` lines runs in **zero simulation time**. Time only moves forward while coroutines are paused on triggers.

---

## 4. Common triggers

All of these live in `cocotb.triggers`:

```python
from cocotb.triggers import Timer, RisingEdge, FallingEdge, ClockCycles, ReadOnly
```

### `Timer`: wait for simulation time

```python
await Timer(10, "ns")
```

Resumes after the given amount of **simulated** time. The second argument is the unit (`"fs"`, `"ps"`, `"ns"`, `"us"`, `"ms"`, `"sec"`).

### `RisingEdge` and `FallingEdge`: wait for a signal edge

```python
await RisingEdge(dut.clk)     # signal goes 0 -> 1
await FallingEdge(dut.clk)    # signal goes 1 -> 0
```

These are by far the most used triggers: synchronous designs do their work on clock edges, so testbenches do too. They apply to **single-bit** signals. The argument is the signal handle, not a string: `RisingEdge(dut.clk)`, not `RisingEdge("clk")`.

### `ValueChange` (called `Edge` in cocotb 1.x): any change

```python
await ValueChange(dut.data_out)    # resumes whenever data_out changes
```

Useful for multi-bit signals or when you care about any change, not a specific direction. In cocotb 1.x this trigger is named `Edge`.

### `ClockCycles`: wait for several clock edges

```python
await ClockCycles(dut.clk, 5)                 # 5 rising edges
await ClockCycles(dut.clk, 5, rising=False)   # 5 falling edges
```

A shortcut for awaiting `RisingEdge` in a loop.

### `ReadOnly`: wait until the time step has settled

```python
await ReadOnly()
```

Resumes at the end of the current time step, after all signal changes in that step have finished. Signal values are final here, which makes it good for sampling. In this phase you **cannot write signals** or await `ReadWrite` or `ReadOnly` again: cocotb raises a `RuntimeError`.

### `ReadWrite` and `NextTimeStep`

```python
await ReadWrite()        # wait until the current evaluation cycle has settled
await NextTimeStep()     # wait until the next time step starts
```

You will meet these less often. They give finer control over where in the simulator's time step your code runs.

### More triggers coming up

Triggers such as `Combine` and `First` let you wait on several things at once, similar to `fork ... join` and `join_any` in SystemVerilog. They come with the next part of the series, on running coroutines concurrently.

---

## 5. Full example

We reuse the design from cocotb Part 2: a 4-bit counter with a clock, a synchronous reset, and a `count` output.

```
triggers_demo/
├── Makefile
├── my_design.v
└── test_triggers.py
```

`my_design.v` and the `Makefile` are the same as in cocotb Part 2, except the test module is `test_triggers`:

```make
COCOTB_TEST_MODULES = test_triggers
```

### `test_triggers.py`

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import Timer, RisingEdge, FallingEdge, ClockCycles


@cocotb.test()
async def trigger_demo(dut):
    """Tour of the most common triggers"""

    # Start a 10 ns clock in the background (explained in a later part, copy it for now)
    cocotb.start_soon(Clock(dut.clk, 10, "ns").start())

    dut.rst.value = 1
    await ClockCycles(dut.clk, 2)          # hold reset for 2 clock cycles
    dut.rst.value = 0
    dut._log.info("Reset released")

    await RisingEdge(dut.clk)              # wait for the next rising edge
    dut._log.info("Saw a rising edge")

    await FallingEdge(dut.clk)             # half a clock period later
    dut._log.info("Saw a falling edge")

    await Timer(25, "ns")                  # wait for a fixed amount of time
    dut._log.info(f"25 ns later, count = {dut.count.value}")
```

Run it with `make`. Each log line carries a simulation timestamp, so you can read the effect of every `await`: the reset hold lasts two clock periods, the falling edge comes half a period after the rising edge, and the last message appears 25 ns after the falling edge.

The `Clock(...).start()` line creates a clock generator, which is itself a coroutine that loops forever on `Timer` triggers. `cocotb.start_soon()` runs it in the background. We cover both in the next part. Without a running clock, `RisingEdge(dut.clk)` would wait forever, because nothing would ever make `clk` rise.

---

## 6. Triggers vs SystemVerilog

If you come from SystemVerilog, most triggers have a direct counterpart:

| SystemVerilog | cocotb |
|---|---|
| `@(posedge clk);` | `await RisingEdge(dut.clk)` |
| `@(negedge clk);` | `await FallingEdge(dut.clk)` |
| `@(data_out);` | `await ValueChange(dut.data_out)` |
| `#10;` | `await Timer(10, "ns")` |
| `repeat (5) @(posedge clk);` | `await ClockCycles(dut.clk, 5)` |
| Sampling at the end of the time step (roughly, the observed or postponed regions) | `await ReadOnly()` |

The idea is the same: the code pauses until an event, then continues. In cocotb, the event is an object you `await`.

---

## 7. Why the timing of a trigger matters

When `await RisingEdge(dut.clk)` returns, the clock has just risen, but the design's registers may **not have updated yet** for that edge. Reading an output right after the edge can show the value from **before** the edge:

```python
await RisingEdge(dut.clk)
value = dut.count.value        # may still be the previous count
```

Two common ways to get settled values:

```python
await FallingEdge(dut.clk)     # sample half a cycle later, when everything is stable
value = dut.count.value
```

```python
await ReadOnly()               # sample at the end of the time step
value = dut.count.value        # no writes allowed after this
```

Sampling on the opposite clock edge is a simple habit that avoids most surprises while you learn. The cocotb documentation has a chapter on the simulator timing model if you want the details.

One simulator note: with Verilator, signal-change triggers (`RisingEdge`, `FallingEdge`, `ValueChange`) are handled after the evaluation cycle ends rather than in the middle of it, so exact timing can differ from event-driven simulators.

---

## 8. Common pitfalls

### Pitfall 1: forgetting `await`

```python
RisingEdge(dut.clk)           # creates a trigger and does nothing
await RisingEdge(dut.clk)     # actually waits
```

### Pitfall 2: passing a name instead of a handle

```python
await RisingEdge("clk")       # wrong
await RisingEdge(dut.clk)     # correct
```

### Pitfall 3: waiting on a clock that never starts

If nothing drives `clk`, `RisingEdge(dut.clk)` never fires, and the test hangs. Start the clock first.

### Pitfall 4: waiting for an event that never comes

Waiting for a signal that never changes also hangs the test. Later in the series, we add timeouts to protect against this.

### Pitfall 5: using `RisingEdge` on a multi-bit signal

`RisingEdge` and `FallingEdge` are for single-bit signals. For a vector, use `ValueChange` (or `Edge` in 1.x), or pick a single bit.

### Pitfall 6: reading right after the edge

As in section 7, the value you read may be from before the edge. Sample on the opposite edge or after `ReadOnly()`.

### Pitfall 7: writing after `ReadOnly()`

```python
await ReadOnly()
dut.rst.value = 1             # not allowed here: raises an error
```

After `ReadOnly`, move on to a later time step (for example, `await Timer(1, "ns")` or `await RisingEdge(dut.clk)`) before writing.

### Pitfall 8: mixing up `Timer` and wall-clock time

`Timer(10, "ns")` is **simulated** time. It has nothing to do with how many real seconds the simulation takes.

### Pitfall 9: using `Timer(0)`

Waiting zero time can behave inconsistently across simulators. Use `await NextTimeStep()` or `ReadWrite()` if you need a delta-level wait.

---

## 9. Why this matters for PyUVM

- Drivers and monitors are loops built on triggers. A driver's `run_phase` typically does something like `await RisingEdge(self.dut.clk)` before driving each transaction, and a monitor awaits an edge before sampling.
- Every PyUVM `run_phase` is an `async def`, so it can `await` triggers like any other coroutine, and everything you learned here applies unchanged.
- Choosing *when* to drive and sample (on which edge, before or after `ReadOnly`) is how you avoid race conditions in a PyUVM testbench, exactly as in a SystemVerilog UVM one.
- PyUVM adds its own waiting mechanisms for communication between components, built on the same coroutine machinery.

---

**Next up:** cocotb Part 4, Accessing DUT signals.
