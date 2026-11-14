# Road to PyUVM, PyUVM Part 2: UVM Phases and Phasing Execution

In Part 1, we introduced PyUVM as the Python implementation of UVM built on cocotb. In standard UVM, test execution is governed by **phases**—ordered steps that build, connect, run, and tear down the testbench.

In SystemVerilog, phases involve complex phase domains, schedules, and task/function distinctions. In PyUVM, phases are **plain Python methods** on `uvm_component`.

This part explains how phases work in PyUVM, how time-consuming vs. non-time-consuming phases are declared, how phasing is kicked off, and key differences from SystemVerilog.

**Topics covered**

1. [Phases are methods of `uvm_component`](#1-phases-are-methods-of-uvm_component)
2. [Functions vs. Coroutines: Non-time-consuming vs. time-consuming phases](#2-functions-vs-coroutines-non-time-consuming-vs-time-consuming-phases)
3. [Starting phasing: `uvm_root().run_test()`](#3-starting-phasing-uvm_rootrun_test)
4. [No `uvm_phase phase` argument in PyUVM](#4-no-uvm_phase-phase-argument-in-pyuvm)
5. [Real-world code examples: `connect_phase` and `run_phase`](#5-real-world-code-examples-connect_phase-and-run_phase)
6. [Summary comparison with SystemVerilog](#6-summary-comparison-with-systemverilog)

---

## 1. Phases are methods of `uvm_component`

In PyUVM, every UVM phase is simply a predefined method on the `uvm_component` base class:

- When you write your own driver, monitor, agent, or test, you inherit from `uvm_component` (or one of its subclasses like `uvm_driver`, `uvm_env`, `uvm_test`).
- To implement phase behavior, you **override** that phase's method.
- Always call `super().<phase_name>()` (e.g., `super().build_phase()` or `await super().run_phase()`) to allow the base component implementation to do its work.

The standard execution sequence in PyUVM consists of:
1. `build_phase` (top-down construction)
2. `connect_phase` (bottom-up TLM port and signal binding)
3. `end_of_elaboration_phase`
4. `start_of_simulation_phase`
5. `run_phase` (parallel runtime execution)
6. `extract_phase`
7. `check_phase`
8. `report_phase`
9. `final_phase`

---

## 2. Functions vs. Coroutines: Non-time-consuming vs. time-consuming phases

A fundamental distinction in verification is whether a phase advances simulation time:

### Non-time-consuming phases (`def`)
Phases that execute in **zero simulation time** are declared as normal Python functions using `def`:
- `build_phase(self)`
- `connect_phase(self)`
- `end_of_elaboration_phase(self)`
- `start_of_simulation_phase(self)`
- `extract_phase(self)`
- `check_phase(self)`
- `report_phase(self)`
- `final_phase(self)`

Because these are synchronous functions, you **cannot** use `await` inside them. They run consecutively to completion.

### Time-consuming phases (`async def`)
Phases that **consume simulation time** (waiting for clock edges, delays, signals) are declared as coroutines using `async def`:
- `run_phase(self)`

Every component's `run_phase()` runs as an independent concurrent cocotb task, scheduled and managed by the cocotb event loop.

---

## 3. Starting phasing: `uvm_root().run_test()`

Phasing does not start automatically when you import files. Instead, phasing begins when you call:

```python
await uvm_root().run_test("test_name_as_string")
```

This call is made inside an `@cocotb.test()` entry function:

```python
import cocotb
from pyuvm import uvm_root

@cocotb.test()
async def test_random(dut):
    """Cocotb entry point that initiates PyUVM phasing"""
    await uvm_root().run_test("apb_test_base")
```

When `run_test("apb_test_base")` is called:
1. PyUVM queries the factory to create the root test component named `"uvm_test_top"` of type `"apb_test_base"`.
2. PyUVM steps through the standard phase order (`build_phase`, `connect_phase`, etc.).
3. When reaching `run_phase`, it starts each component's `run_phase()` coroutine as a background cocotb task.
4. It awaits objections: once all objections are dropped, `run_phase` terminates and clean-up phases (`extract`, `check`, `report`) run.

---

## 4. No `uvm_phase phase` argument in PyUVM

In SystemVerilog UVM, every phase method receives a phase handle as an argument:

```systemverilog
// SystemVerilog UVM: phase handle is required in every signature
virtual function void build_phase(uvm_phase phase);
virtual function void connect_phase(uvm_phase phase);
virtual task run_phase(uvm_phase phase);
```

In PyUVM, **no `phase` argument is passed**:

```python
# PyUVM: clean signatures with only self
def build_phase(self):
    super().build_phase()

def connect_phase(self):
    super().connect_phase()

async def run_phase(self):
    await super().run_phase()
```

If you need to raise or drop objections in `run_phase`, you call methods directly on `self`:
```python
self.raise_objection()
# ... do work ...
self.drop_objection()
```
There is no `phase.raise_objection(this)` needed!

---

## 5. Real-world code examples: `connect_phase` and `run_phase`

Here are two practical examples from an APB slave verification environment ([PyUVM-APB_slave-environment](https://github.com/mahmoudhussein1227/PyUVM-APB_slave-environment)):

### Example 1: `connect_phase` in an Agent (Non-time-consuming, synchronous)
In `connect_phase`, TLM ports are bound together, and hardware handles (e.g., `dut`) are distributed to child components. It executes synchronously in zero simulation time:

```python
class apb_agent(uvm_agent, reset_handler):
    """Bundle the APB sequencer, driver, monitor, and reset coordination."""

    def __init__(self, name, parent=None):
        super().__init__(name, parent)
        self.dut = ConfigDB().get(self, "", "dut")

    def build_phase(self):
        super().build_phase()
        self.driver = apb_driver.create("driver", self)
        self.sqr = apb_sqr.create("sqr", self)
        self.monitor = apb_monitor.create("monitor", self)

    def connect_phase(self):
        super().connect_phase()
        # Connect driver's sequence item port to sequencer export
        self.driver.seq_item_port.connect(self.sqr.seq_item_export)
        
        # Pass DUT signal handle to bus-facing components
        self.driver.dut = self.dut
        self.monitor.dut = self.dut
```

### Example 2: `run_phase` in a Test (Time-consuming, asynchronous coroutine)
In `run_phase`, simulation time advances using `await`. Notice the objection handling and sequence execution:

```python
class apb_test_base(uvm_test):
    """Default randomized test: run sequences against the environment."""

    def __init__(self, name, parent=None):
        super().__init__(name, parent)
        self.num_sequences = 5000

    def build_phase(self):
        super().build_phase()
        self.env = apb_env.create("env", self)

    async def run_phase(self):
        await super().run_phase()
        
        # Raise objection to prevent test from ending early
        self.raise_objection()
        
        # Time-consuming wait for reset to settle
        await Timer(100, "ns")

        # Start randomized stimulus sequences
        for _ in range(self.num_sequences):
            seq = apb_sequence_random.create("seq")
            await seq.start(self.env.agent.sqr)

        await Timer(500, "ns")
        
        # Drop objection when traffic completes
        self.drop_objection()
```

---

## 6. Summary comparison with SystemVerilog

| Aspect | SystemVerilog UVM | PyUVM |
|---|---|---|
| **Phase Definition** | Methods on `uvm_component` | Methods on `uvm_component` |
| **Non-time-consuming phases** | `function void <phase>(uvm_phase phase);` | Normal Python functions: `def <phase>(self):` |
| **Time-consuming phase** | `task run_phase(uvm_phase phase);` | Python coroutine: `async def run_phase(self):` |
| **Phase argument** | Required: `(uvm_phase phase)` | **None**: Just `(self)` |
| **Objection calls** | `phase.raise_objection(this)` | Directly on component: `self.raise_objection()` |
| **Starting Phasing** | `run_test("test_name")` in initial block | `await uvm_root().run_test("test_name")` in `@cocotb.test()` |

---

**Next up:** PyUVM Part 3, The Factory and Object Creation.
