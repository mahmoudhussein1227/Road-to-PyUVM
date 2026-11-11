# Road to PyUVM, cocotb Part 6: Checks with `assert` and Inter-Task Communication with `Queue`

In Part 5, we learned how to spawn independent background tasks with `cocotb.start_soon()`. But real testbenches need two crucial things:
1. **Verification**: How do we verify DUT outputs and fail the test when behavior is incorrect?
2. **Communication**: How do concurrent tasks (like a driver, monitor, and scoreboard) pass data between each other safely without race conditions?

This final part of the cocotb section covers Python's **`assert` statement** for test checking and cocotb's **`Queue`** for safe, async inter-task communication.

**Topics covered**

1. [How cocotb tests pass or fail](#1-how-cocotb-tests-pass-or-fail)
2. [The `assert` statement](#2-the-assert-statement)
3. [Writing descriptive failure messages](#3-writing-descriptive-failure-messages)
4. [Communicating between tasks: Why plain lists are risky](#4-communicating-between-tasks-why-plain-lists-are-risky)
5. [The cocotb `Queue` object](#5-the-cocotb-queue-object)
6. [Putting and getting data: `put()`, `get()`, and non-blocking variants](#6-putting-and-getting-data-put-get-and-non-blocking-variants)
7. [Full example: Producer-Consumer with Checker](#7-full-example-producer-consumer-with-checker)
8. [Why this matters for PyUVM](#8-why-this-matters-for-pyuvm)

---

## 1. How cocotb tests pass or fail

In cocotb, a test's outcome is determined by whether an unhandled exception is raised:

- **Pass**: The `@cocotb.test()` coroutine reaches the end of its execution without unhandled exceptions.
- **Fail**: An exception is raised (most commonly `AssertionError`). The test immediately stops, and the failure along with its traceback is logged in the simulation report.

A test with no checks or assertions will always "pass" as long as it doesn't crash! To make it a true verification test, you must actively verify output data.

---

## 2. The `assert` statement

Python's built-in `assert` is the standard checking mechanism in cocotb:

```python
assert condition, "Optional error message"
```

- If `condition` evaluates to `True`, execution continues normally.
- If `condition` evaluates to `False`, Python raises an `AssertionError` with the provided message, immediately failing the test.

```python
actual = dut.count.value.to_unsigned()
expected = 5

assert actual == expected, f"Counter mismatch: expected {expected}, got {actual}"
```

---

## 3. Writing descriptive failure messages

When an assertion fails in a large simulation regression, the error message should immediately tell you:
1. **What was expected**
2. **What was actually received**
3. **Relevant context** (e.g., simulation time, transaction index, address)

### Bad:
```python
assert dut.data_out.value == expected_data
# Output on failure: AssertionError (no context!)
```

### Good:
```python
assert dut.data_out.value == expected_data, (
    f"[Packet {pkt_id}] Data mismatch at addr 0x{addr:04X}! "
    f"Expected: 0x{expected_data:02X}, Actual: 0x{dut.data_out.value.to_unsigned():02X}"
)
```

---

## 4. Communicating between tasks: Why plain lists are risky

In Part 5, we had multiple tasks running concurrently (e.g., monitor and generator). If a monitor collects packets and a scoreboard checks them:

```python
# Risky pattern: plain Python list
packets = []

async def monitor():
    while True:
        ...
        packets.append(item)

async def scoreboard():
    while True:
        if len(packets) > 0:
            item = packets.pop(0)
            check(item)
        await Timer(1, "ns")  # Inefficient polling!
```

Polling a plain list with `while len(packets) == 0: await Timer(...)` has major drawbacks:
- It wastes simulator cycles polling repeatedly.
- If you poll on a fixed timer, you risk missing events or introducing timing skew.
- There is no built-in synchronization or backpressure (capacity limits).

---

## 5. The cocotb `Queue` object

To solve this, cocotb provides `cocotb.queue.Queue`: an **asynchronous FIFO queue** specifically designed for coroutines.

```python
from cocotb.queue import Queue

q = Queue(maxsize=0)  # maxsize=0 means infinite capacity
```

Key benefits of `Queue`:
- **Producer-Consumer synchronization**: If a consumer calls `await q.get()`, it pauses cleanly until an item is pushed. No polling needed!
- **Backpressure**: If `maxsize > 0` and the queue is full, a producer calling `await q.put(item)` pauses until space becomes available.
- **Thread/Coroutine safe**: Handles coordination between concurrent cocotb tasks without race conditions.

---

## 6. Putting and getting data: `put()`, `get()`, and non-blocking variants

`cocotb.queue.Queue` provides both coroutine-based (blocking on empty/full) and synchronous (immediate) methods:

| Method | Behavior | When to use |
|---|---|---|
| `await q.put(item)` | Puts an item into the queue. Pauses if the queue is full (`maxsize > 0`). | Standard producer in coroutines |
| `q.put_nowait(item)` | Puts an item immediately without awaiting. Raises `QueueFull` if full. | When producer cannot or should not yield |
| `await q.get()` | Retrieves and removes an item. **Pauses until an item is available**. | Standard consumer in coroutines |
| `q.get_nowait()` | Retrieves an item immediately. Raises `QueueEmpty` if empty. | Non-blocking check |
| `q.empty()` | Returns `True` if the queue is currently empty. | Sanity checks or drain checks |
| `q.qsize()` | Returns the number of items currently in the queue. | Checking backlog |

---

## 7. Full example: Producer-Consumer with Checker

Here is a full pattern where a driver produces packets, a monitor observes the DUT and pushes observed items into a `Queue`, and a checker verifies them with `assert`:

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, ClockCycles
from cocotb.queue import Queue


async def driver(dut, tx_queue):
    """Pulls stimulus and drives DUT inputs"""
    while not tx_queue.empty():
        val = await tx_queue.get()
        dut.data_in.value = val
        await RisingEdge(dut.clk)


async def monitor(dut, mon_queue, count):
    """Monitors DUT output and pushes captured items into a queue"""
    for _ in range(count):
        await RisingEdge(dut.clk)
        captured = dut.data_out.value.to_unsigned()
        await mon_queue.put(captured)


async def scoreboard(mon_queue, expected_values):
    """Pulls items from queue and validates them against expected values"""
    for exp in expected_values:
        # Pauses until the monitor puts an item!
        actual = await mon_queue.get()
        assert actual == exp, f"Mismatch: expected 0x{exp:02X}, got 0x{actual:02X}"


@cocotb.test()
async def test_queue_and_assert(dut):
    # 1. Start clock
    cocotb.start_soon(Clock(dut.clk, 10, "ns").start())

    # 2. Reset DUT
    dut.rst.value = 1
    await ClockCycles(dut.clk, 2)
    dut.rst.value = 0

    # 3. Create test data & queues
    test_data = [0x10, 0x20, 0x30, 0x40]
    tx_queue = Queue()
    mon_queue = Queue()

    for item in test_data:
        tx_queue.put_nowait(item)

    # 4. Spawn concurrent driver, monitor, and scoreboard tasks
    cocotb.start_soon(driver(dut, tx_queue))
    cocotb.start_soon(monitor(dut, mon_queue, count=len(test_data)))

    # 5. Wait for scoreboard to finish checking all items
    await scoreboard(mon_queue, expected_values=test_data)
    dut._log.info("All transactions verified successfully!")
```

---

## 8. Why this matters for PyUVM

This combination of **assertions** and **queues** directly introduces how PyUVM structures verification environments:

- **TLM FIFOs & Analysis FIFOs**: PyUVM's `uvm_tlm_fifo` and `uvm_analysis_port` are built on top of this exact asynchronous queue concept. Monitors write into FIFOs, and scoreboards pull from them using `get()`.
- **Scoreboards**: In PyUVM, a scoreboard holds reference models and comparison logic implemented using Python `assert` statements.
- **Decoupled Architecture**: Queues allow producers (drivers/monitors) and consumers (checkers/scoreboards) to run completely decoupled in time.

---

🎉 **This concludes the cocotb section!**

**Next up:** Entering the **PyUVM** section — starting with *PyUVM Architecture and the UVM Component Tree*.
