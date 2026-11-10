# Road to PyUVM, cocotb Part 2: Coroutines and `@cocotb.test()`

In cocotb Part 1 we saw that cocotb runs Python inside the simulator. This part covers the two ideas every cocotb test is built on: **coroutines**, which let a Python function pause and resume, and the **`@cocotb.test()` decorator**, which tells cocotb which functions are tests. We finish with a complete "Hello, world" test.

**Topics covered**

1. [What is a coroutine?](#1-what-is-a-coroutine)
2. [Defining, calling, and awaiting a coroutine](#2-defining-calling-and-awaiting-a-coroutine)
3. [Coroutines and concurrency](#3-coroutines-and-concurrency)
4. [Why hardware verification needs coroutines](#4-why-hardware-verification-needs-coroutines)
5. [A quick look at decorators](#5-a-quick-look-at-decorators)
6. [`@cocotb.test()`](#6-cocotbtest)
7. [Full "Hello, world" example](#7-full-hello-world-example)
8. [Test options and multiple tests](#8-test-options-and-multiple-tests)
9. [Common pitfalls](#9-common-pitfalls)
10. [Why this matters for PyUVM](#10-why-this-matters-for-pyuvm)

---

## 1. What is a coroutine?

A **coroutine** is a special kind of function that can be **paused and resumed**.

A normal function runs from its first line to its last without stopping: once called, nothing else can happen until it returns. A coroutine can stop in the middle, hand control back, and later continue from exactly the point where it paused, with all its local variables intact.

```
 normal function:   start ─────────────────────────► end

 coroutine:         start ───► pause ......... resume ───► end
                                  │ other work can run here │
```

Coroutines are one of the pillars of **concurrency**: they let a program juggle several activities at once by letting each one pause while it waits, so the others can make progress.

---

## 2. Defining, calling, and awaiting a coroutine

Python uses two keywords:

- **`async def`** defines a coroutine function.
- **`await`** pauses the current coroutine until the thing being waited on is finished. It can only be used inside an `async def`.

```python
import asyncio

async def worker(name, delay):
    print(f"{name}: start")
    await asyncio.sleep(delay)      # pause here, let others run
    print(f"{name}: done")

async def main():
    await asyncio.gather(worker("A", 2), worker("B", 1))

asyncio.run(main())
```

Output:

```
A: start
B: start
B: done
A: done
```

Both workers start immediately. `A` pauses for 2 seconds and `B` for 1, so `B` finishes first. Neither blocked the other: while one was paused, the other ran.

This example uses Python's built-in `asyncio` library as the scheduler, just to show the idea. **cocotb has its own scheduler** and its own things to wait for (clock edges, simulation time), but `async def` and `await` work the same way.

### Calling a coroutine does not run it

```python
w = worker("A", 1)
print(w)      # <coroutine object worker at 0x...>
```

Calling `worker("A", 1)` only creates a **coroutine object**. Its body has not started. It runs only when something **awaits** it or the scheduler starts it. If you forget this you get a warning like `coroutine 'worker' was never awaited`.

---

## 3. Coroutines and concurrency

Coroutines give **cooperative concurrency**:

- Many coroutines can be in progress at the same time.
- Only **one** runs at any instant, on a single thread.
- A coroutine keeps running until it reaches an `await`. That is when it "cooperates" by giving control back.
- When the thing it awaits is ready, the scheduler resumes it.

This is different from threads, where the operating system can interrupt a thread at any moment. With coroutines, switching happens only at `await` points, so you always know where your code can be interrupted, and there are no unexpected interleavings between two lines of code.

### In cocotb: waiting on the simulator

In a cocotb test, the thing you usually await is a **simulator event**: a clock edge, a time delay, a signal change. These are called **triggers** (the next topic in the series).

```python
await Timer(10, "ns")       # pause until 10 ns of simulation time have passed
```

While your coroutine is paused on this trigger, the simulator keeps running. When 10 ns have passed, the simulator calls back into Python, and your coroutine resumes on the next line.

An important detail: **simulation time does not advance while Python code runs.** Your code between two `await` statements executes in zero simulation time. Time moves forward only when every coroutine is waiting for something.

---

## 4. Why hardware verification needs coroutines

Hardware is naturally concurrent: many things happen at the same time. In a testbench you typically need, simultaneously:

- a clock generator toggling forever,
- a driver applying stimulus,
- a monitor watching outputs,
- a checker comparing results.

Each of these is a loop that spends most of its time **waiting**: for a clock edge, a handshake, a response. Coroutines are the natural fit, because each activity is written as a simple top-to-bottom sequence with `await` at each wait, and the scheduler interleaves them. Without coroutines you would have to write callbacks and state machines by hand.

If you know SystemVerilog, a coroutine plays a similar role to a `task` containing `@(posedge clk)` or `#10`, and running several at once is like `fork ... join_none`. (We cover the cocotb version, `cocotb.start_soon()`, later in the series.)

---

## 5. A quick look at decorators

A **decorator** is a function that takes another function and returns a modified or registered version of it. You apply it by writing `@name` on the line above a `def`.

```python
def shout(func):
    def wrapper():
        print("before")
        func()
        print("after")
    return wrapper

@shout
def hi():
    print("hi")

hi()
# before
# hi
# after
```

The `@shout` line is shorthand for:

```python
hi = shout(hi)
```

So a decorator can wrap a function, change its behavior, or simply **register** it somewhere. That last use is what cocotb needs.

---

## 6. `@cocotb.test()`

cocotb provides a decorator, `@cocotb.test()`, that **marks a coroutine as a test**.

```python
import cocotb

@cocotb.test()
async def my_test(dut):
    ...
```

### Why it is needed

The simulator knows nothing about your Python function, and cocotb can't guess which of your functions are tests and which are helpers. The decorator is the label. It registers the function with cocotb so it can be found and run.

### What happens when the simulation starts

1. The Makefile starts the simulator and loads cocotb (cocotb Part 1).
2. cocotb imports the Python modules listed in `COCOTB_TEST_MODULES`.
3. Importing the module executes each `@cocotb.test()` decorator, which **registers** the decorated functions as tests.
4. cocotb runs the registered tests one after another (by default, in the order they are defined in the file), executing each test's body.
5. When a test finishes, cocotb records pass or fail. After the last test, it prints a summary and ends the simulation.

So you never call your tests yourself. You decorate a function, and cocotb finds and runs it.

### The rules

- The function must be **`async def`**, because a test must be able to pause on triggers.
- It takes one argument, conventionally named **`dut`**: a handle to the toplevel design. Signals are reached with the dot operator, like `dut.clk` (Python OOP Part 5).
- Use the decorator **with parentheses**: `@cocotb.test()`.
- A test **passes** if it finishes without raising an exception, and **fails** if an exception is raised, such as a failed `assert`. (Assertions are covered later in the series.)

---

## 7. Full "Hello, world" example

A complete cocotb project is three small files.

```
hello_world/
├── Makefile
├── my_design.v
└── test_hello.py
```

### `my_design.v` (a tiny design to test)

```verilog
module my_design (
    input            clk,
    input            rst,
    output reg [3:0] count
);
    always @(posedge clk) begin
        if (rst) count <= 4'd0;
        else     count <= count + 4'd1;
    end
endmodule
```

### `test_hello.py` (the cocotb test)

```python
import cocotb
from cocotb.triggers import Timer


@cocotb.test()
async def hello_world(dut):
    """My first cocotb test"""
    dut._log.info("Hello, world! The test is starting")

    await Timer(10, "ns")           # pause: simulation advances 10 ns

    dut._log.info("Back after 10 ns of simulation time")
```

What each part does:

| Line | Meaning |
|---|---|
| `import cocotb` | Gives us the `@cocotb.test()` decorator |
| `from cocotb.triggers import Timer` | Imports a trigger that waits for simulation time |
| `@cocotb.test()` | Registers the function below as a cocotb test |
| `async def hello_world(dut)` | A coroutine; `dut` is the handle to `my_design` |
| `dut._log.info(...)` | Prints a log message, stamped with simulation time |
| `await Timer(10, "ns")` | Pauses the coroutine for 10 ns of simulated time, then resumes |

### `Makefile`

```make
SIM ?= questa
TOPLEVEL_LANG ?= verilog

VERILOG_SOURCES += $(PWD)/my_design.v

COCOTB_TOPLEVEL = my_design
COCOTB_TEST_MODULES = test_hello

include $(shell cocotb-config --makefiles)/Makefile.sim
```

### Run it

```
make
```

You should see output similar to this (the exact format depends on your cocotb version):

```
     0.00ns INFO     cocotb.my_design      Hello, world! The test is starting
    10.00ns INFO     cocotb.my_design      Back after 10 ns of simulation time
    10.00ns INFO     cocotb.regression     hello_world passed

** TEST                    STATUS  SIM TIME (ns)  REAL TIME (s) ... **
** test_hello.hello_world   PASS        10.00           0.00     ... **
```

Read the timestamps: the first message appears at 0 ns, the coroutine pauses, and the second message appears at 10 ns. That jump is the pause-and-resume from section 1, happening in simulation time.

---

## 8. Test options and multiple tests

A file can contain several tests. Each decorated coroutine is its own test and appears as its own row in the summary:

```python
@cocotb.test()
async def test_reset(dut):
    ...

@cocotb.test()
async def test_counting(dut):
    ...
```

The decorator accepts optional arguments to control a test. Two common ones:

```python
@cocotb.test(skip=True)           # register it but do not run it
async def not_ready_yet(dut): ...

@cocotb.test(expect_fail=True)    # this test is expected to fail
async def known_bug(dut): ...
```

There are also options for timeouts. See the cocotb documentation for the full list for your version.

---

## 9. Common pitfalls

### Pitfall 1: forgetting `async`

```python
@cocotb.test()
def hello_world(dut):        # missing async: not a coroutine, cannot await
    ...
```

### Pitfall 2: forgetting `await` on a trigger

```python
Timer(10, "ns")              # creates the trigger and does nothing
await Timer(10, "ns")        # actually pauses
```

This is the coroutine rule from section 2 again: without `await`, nothing waits.

### Pitfall 3: blocking with ordinary Python waits

```python
import time
time.sleep(1)                # freezes the whole simulator for 1 real second
```

Python runs inside the simulator process on one thread, so a blocking call blocks the simulation too. To wait, use a cocotb trigger such as `await Timer(...)`.

### Pitfall 4: forgetting the `dut` parameter

```python
@cocotb.test()
async def hello_world():     # no dut: cocotb passes it, so this fails
    ...
```

### Pitfall 5: test not found

If your test never runs, check that the file name is in `COCOTB_TEST_MODULES` (without `.py`), that the function has the `@cocotb.test()` decorator, and that you ran `make` in the folder containing the file.

### Pitfall 6: a test with no checks always passes

A test passes simply because it finished without an exception. Hello world "passes" without verifying anything. Real tests add `assert` statements, which are covered later in this series.

### Pitfall 7: calling a coroutine without awaiting

```python
my_helper(dut)               # RuntimeWarning: coroutine was never awaited
await my_helper(dut)         # runs it
```

---

## 10. Why this matters for PyUVM

- A PyUVM testbench is launched from a cocotb test. The starting point is an `@cocotb.test()` coroutine that creates the UVM environment and runs it, for example `await uvm_root().run_test("MyTest")`.
- The `run_phase` of your drivers, monitors, and sequences is written as **`async def`**: each is a coroutine that awaits clock edges and handshakes, all running concurrently under cocotb's scheduler.
- Everything in the next parts of the series (triggers, `start_soon`, `gather`, `select`) builds on `async`, `await`, and the idea that a coroutine can pause and resume.

---

**Next up:** cocotb Part 3, Triggers.

