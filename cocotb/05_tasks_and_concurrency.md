# Road to PyUVM, cocotb Part 5: Tasks, Concurrency, `gather`, and `select`

In the previous parts, we wrote coroutines that ran sequentially using `await`. But hardware verification is inherently parallel: clocks tick continuously, reset sequences run, monitors sniff bus traffic, drivers stream transactions, and scoreboards check results—all at the exact same simulation time.

In cocotb, parallel activities are modeled using **Tasks**. This part explains how tasks are spawned, how to manage and cancel them like SystemVerilog processes, and how to synchronize multiple tasks using `gather` and `select`.

**Topics covered**

1. [What is a task?](#1-what-is-a-task)
2. [Spawning a task with `cocotb.start_soon()`](#2-spawning-a-task-with-cocotbstart_soon)
3. [Managing tasks using task handles](#3-managing-tasks-using-task-handles)
4. [Tasks vs. SystemVerilog processes](#4-tasks-vs-systemverilog-processes)
5. [Synchronizing multiple tasks: `gather`](#5-synchronizing-multiple-tasks-gather)
6. [Racing tasks with `select`](#6-racing-tasks-with-select)
7. [Full example](#7-full-example)
8. [Why this matters for PyUVM](#8-why-this-matters-for-pyuvm)

---

## 1. What is a task?

A **Task** is an independent execution process created from a coroutine. 

When you `await my_coroutine()`, execution halts in the caller until `my_coroutine()` completes. It is strictly sequential.

A task, by contrast, runs **in parallel with the parent process**:
- The parent process launches the task and continues immediately (non-blocking launch).
- The task runs concurrently in the background. Whenever the parent or other tasks yield control (by awaiting a trigger or time delay), the simulator scheduler allows the task to run its step.
- Python remains single-threaded, but cocotb cooperatively interleaves execution between tasks as they hit simulation events.

```
 Sequential:  parent ────────► await helper ────────► parent continues
 
 Concurrent:  parent ────────┬──────────────────────► parent continues
 (Tasks)                     └─► task runs alongside │
                                 whenever parent yields
```

---

## 2. Spawning a task with `cocotb.start_soon()`

To turn a coroutine into an independent background task, pass it to `cocotb.start_soon()`:

```python
import cocotb
from cocotb.triggers import Timer

async def background_blinker():
    while True:
        await Timer(10, "ns")

@cocotb.test()
async def test_demo(dut):
    # Start the background task
    task = cocotb.start_soon(background_blinker())
    
    # Parent continues immediately!
    await Timer(100, "ns")
```

`cocotb.start_soon()` schedules the coroutine to start on the next step and returns a **Task handle**.

---

## 3. Managing tasks using task handles

The handle returned by `cocotb.start_soon()` gives you full control over the lifecycle of the task:

### Awaiting task completion: `await task`
You can await a task handle just like a trigger. The caller pauses until the task finishes and returns its result:

```python
async def compute():
    await Timer(50, "ns")
    return 0xCAFE

task = cocotb.start_soon(compute())
# ... do other work in parallel ...
result = await task   # waits for task if not yet done; result = 0xCAFE
```

### Checking status: `.done()`
Query whether a task has completed without pausing:

```python
if task.done():
    val = task.result()
```

### Cancelling a task: `.cancel()`
Background workers (like clock generators or continuous bus monitors) often loop forever (`while True`). When your test finishes, you can cancel them cleanly using `.cancel()`:

```python
task = cocotb.start_soon(background_monitor(dut))

# Run main traffic
await run_traffic(dut)

# Terminate background task
task.cancel()
```

When `.cancel()` is called on a task, it is immediately stopped and removed from the cocotb scheduler.

---

## 4. Tasks vs. SystemVerilog processes

If you come from SystemVerilog, cocotb tasks map directly to dynamic processes (`fork / join` and `process` class):

| Operation | SystemVerilog | cocotb |
|---|---|---|
| Spawn background process | `fork begin ... end join_none` | `task = cocotb.start_soon(...)` |
| Wait for process to finish | `wait fork;` or `process.await()` | `await task` |
| Terminate process | `disable fork;` or `process.kill()` | `task.cancel()` |
| Join all parallel branches | `fork ... join` | `await gather(*tasks)` |
| Join first completing branch | `fork ... join_any` | `await select(*tasks)` |

---

## 5. Synchronizing multiple tasks: `gather`

When you want to launch multiple concurrent actions and wait for **all of them to finish** (equivalent to SystemVerilog `fork ... join`), use `cocotb.triggers.gather`.

### What it does:
Takes multiple awaitables/tasks, runs them concurrently, and pauses until **all** of them complete.

### Inputs & Outputs:
- **Inputs**: Multiple awaitable objects or tasks passed as arguments: `gather(task1, task2, ...)` or unpacked from a list `gather(*task_list)`.
- **Output**: Returns a list containing the results of each task in the same order they were passed.

```python
from cocotb.triggers import gather, Timer

async def read_reg(addr):
    await Timer(20, "ns")
    return addr * 2

async def write_reg(addr, data):
    await Timer(30, "ns")
    return "OK"

@cocotb.test()
async def test_gather(dut):
    # Run both operations concurrently and wait for BOTH to finish
    results = await gather(read_reg(0x10), write_reg(0x20, 0xAA))
    
    # Total elapsed time = 30 ns (the longest operation)
    # results = [0x20, "OK"]
```

If any task raises an exception, `gather` re-raises that exception.

---

## 6. Racing tasks with `select`

When you want to run multiple operations concurrently and proceed as soon as the **first one completes**, use `cocotb.triggers.select`.

### What it does:
Races multiple awaitables/tasks against each other and wakes up as soon as the first one completes. 

> **Important:** Once the winning trigger/task finishes, `select` automatically **cancels all remaining unfinished tasks/triggers** participating in that race, ensuring no orphaned background processes keep running.

### Inputs & Outputs:
- **Inputs**: Multiple awaitable objects or tasks: `select(task1, task2, ...)`.
- **Output**: Returns a tuple `(result, index, task)`:
  - `result`: the value returned by the winning awaitable.
  - `index`: the 0-based position of the winner among the passed arguments.
  - `task`: the task object that finished first.

### Common pattern: operation with timeout
A classic verification requirement is guarding against a DUT lockup using a watchdog timer. Notice that because `select` cancels unfinished tasks automatically when one finishes, you don't need to manually cancel the remaining task:

```python
from cocotb.triggers import select, Timer, RisingEdge

async def wait_for_done(dut):
    while dut.done.value == 0:
        await RisingEdge(dut.clk)
    return "DONE"

@cocotb.test()
async def test_timeout(dut):
    done_task = cocotb.start_soon(wait_for_done(dut))
    timeout = Timer(100, "ns")

    # Race DUT response against watchdog timer
    # The loser is automatically cancelled!
    res, idx, win_task = await select(done_task, timeout)

    assert idx == 0, "Test timed out waiting for dut.done!"
    dut._log.info("Operation completed successfully!")
```

---

## 7. Full example

Here is a complete test demonstrating tasks, `start_soon`, cancellation, and `select`:

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import Timer, RisingEdge, ClockCycles, select, gather

async def generate_reset(dut, cycles):
    dut.rst.value = 1
    await ClockCycles(dut.clk, cycles)
    dut.rst.value = 0

async def packet_listener(dut, received_list):
    """Continuous background listener that samples output count"""
    while True:
        await RisingEdge(dut.clk)
        if dut.rst.value == 0:
            received_list.append(dut.count.value.to_unsigned())

@cocotb.test()
async def test_tasks_demo(dut):
    # 1. Start clock as an independent task
    cocotb.start_soon(Clock(dut.clk, 10, "ns").start())

    # 2. Start background monitor task
    samples = []
    mon_task = cocotb.start_soon(packet_listener(dut, samples))

    # 3. Use gather to run reset and an initial wait concurrently
    await gather(
        generate_reset(dut, 3),
        Timer(30, "ns")
    )

    # 4. Wait for 5 cycles using select with a timeout
    watchdog = Timer(100, "ns")
    counter_run = ClockCycles(dut.clk, 5)

    _, idx, _ = await select(counter_run, watchdog)
    assert idx == 0, "Counter stalled!"

    # 5. Clean up background monitor
    mon_task.cancel()
    dut._log.info(f"Monitor collected {len(samples)} samples: {samples}")
```

---

## 8. Why this matters for PyUVM

Tasks and concurrency are the operational backbone of UVM:
- **`run_phase` orchestration**: In PyUVM, all UVM components (`uvm_driver`, `uvm_monitor`, `uvm_scoreboard`) execute their `run_phase` methods concurrently as cocotb tasks.
- **Background threads in components**: A driver might have one loop fetching sequence items from a sequencer and another task driving the pin-level interface.
- **Heartbeats & watchdogs**: Scoreboards and environments use `select` or timer tasks to detect deadlocks and dropped packets.
- **Clean phase stopping**: When the test moves from `run_phase` to `extract_phase`, PyUVM stops and cleans up running tasks.

---

**Next up:** cocotb Part 6, assert
