# Road to PyUVM, PyUVM Part 1: Brief Introduction to PyUVM

We have completed the **Python OOP** fundamentals and the **cocotb** cosimulation foundation. Now, we arrive at the core destination of this series: **PyUVM**.

If you are already familiar with SystemVerilog UVM (IEEE 1800.2), you will feel right at home with PyUVM. However, Python is not SystemVerilog: some things are vastly simpler, some are adapted to Python's dynamic nature, and some SV-UVM features are implemented differently or omitted altogether.

This part introduces what PyUVM is, how it maps to SystemVerilog UVM, and outlines the focus of upcoming tutorials.

**Topics covered**

1. [What is PyUVM?](#1-what-is-pyuvm)
2. [What exists in PyUVM: Core UVM concepts retained](#2-what-exists-in-pyuvm-core-uvm-concepts-retained)
3. [Key differences & what is not fully implemented](#3-key-differences--what-is-not-fully-implemented)
4. [Scope of this series: Focusing on differences, not re-teaching UVM](#4-scope-of-this-series-focusing-on-differences-not-re-teaching-uvm)
5. [A minimal PyUVM "Hello World" anatomy](#5-a-minimal-pyuvm-hello-world-anatomy)
6. [Summary](#6-summary)

---

## 1. What is PyUVM?

**PyUVM** is an open-source Python implementation of the Universal Verification Methodology (UVM), developed by Ray Salemi. It is built directly on top of **cocotb**.

- **cocotb** provides the simulator interaction, clock generation, signal driving/sampling (`.value`), triggers, and asynchronous tasks.
- **PyUVM** provides the verification methodology structure: standard component hierarchies, phases, the factory pattern, the configuration database (ConfigDB), and sequence mechanisms.

Instead of writing verbose SystemVerilog boilerplate code with macros, packages, and complex interface bindings, PyUVM provides a clean, pythonic way to build full-scale UVM testbenches.

---

## 2. What exists in PyUVM: Core UVM concepts retained

PyUVM implements the primary pillars of standard UVM:

- **Component Hierarchy & Tree**: Base classes like `uvm_component`, `uvm_driver`, `uvm_monitor`, `uvm_sequencer`, `uvm_scoreboard`, `uvm_agent`, `uvm_env`, and `uvm_test`.
- **The UVM Factory**: Dynamic object and component creation with type overrides and instance overrides (`uvm_factory`).
- **Configuration Database (ConfigDB)**: Safe passing of parameters, configuration objects, and virtual interface handles (`ConfigDB().set()` and `ConfigDB().get()`).
- **Phasing Mechanism**: Standard build and runtime execution phases (`build_phase`, `connect_phase`, `run_phase`, `extract_phase`, `check_phase`, `report_phase`).
- **Sequences and Sequence Items**: Transaction generation using `uvm_sequence_item`, `uvm_sequence`, and `uvm_sequencer`.

If you know how a UVM testbench is architected in SystemVerilog, your mental model of the hierarchy remains identical in PyUVM.

---

## 3. Key differences & what is not fully implemented

While PyUVM stays faithful to UVM concepts, Python's dynamic typing and language design change how things work in practice:

### 1. No Macro Boilerplate
In SV-UVM, every component and sequence item requires macros like `` `uvm_component_utils `` or `` `uvm_field_* ``. In PyUVM, **there are no macros**. Python's introspection and dynamic nature handle registration and field inspection naturally.

### 2. Analysis Implementations (`uvm_analysis_imp`)
In SystemVerilog, connecting multiple monitors to a single scoreboard requires distinct `uvm_analysis_imp_decl` macros to implement multiple `write()` methods.
In PyUVM, `uvm_analysis_imp` does not exist in the same way. Instead, PyUVM primarily uses **`uvm_tlm_fifo`** and **`uvm_analysis_export`**, or leverages Python's flexible callbacks and queues to handle multiple subscriber streams cleanly.

### 3. Phasing is Simplified
SV-UVM has elaborate sub-phases (`pre_reset_phase`, `reset_phase`, `post_reset_phase`, `main_phase`, etc.). PyUVM focuses on the standard, essential top-level phases (`build_phase`, `connect_phase`, `run_phase`, `extract_phase`, `check_phase`, `report_phase`).

### 4. Objections & Async Phase Execution
In PyUVM, the `run_phase` is an `async` coroutine running as a cocotb task. Raising and dropping objections (`phase.raise_objection()`, `phase.drop_objection()`) controls the lifetime of the `run_phase` cleanly without complex phase-domain synchronization.

---

## 4. Scope of this series: Focusing on differences, not re-teaching UVM

> **Important Note on Series Scope:**
> This series assumes you already understand general hardware verification concepts and the fundamental purpose of UVM (what an agent is, why we use a factory, what transactions do).
> 
> Rather than re-explaining general UVM theory from scratch, the upcoming tutorials will focus specifically on:
> - **How to implement each UVM concept in Python/PyUVM.**
> - **Where PyUVM diverges from SystemVerilog UVM.**
> - **Pythonic idioms and patterns that replace complex SV workarounds.**

---

## 5. A minimal PyUVM "Hello World" anatomy

Here is what a minimal PyUVM test looks like:

```python
import cocotb
from pyuvm import *

# 1. Define a UVM Test component
class HelloWorldTest(uvm_test):
    def build_phase(self):
        super().build_phase()
        self.logger.info("Hello from build_phase!")

    async def run_phase(self):
        self.raise_objection()
        self.logger.info("Hello from run_phase!")
        await cocotb.triggers.Timer(10, "ns")
        self.drop_objection()

# 2. Launch the UVM testbench from a cocotb test
@cocotb.test()
async def test_top(dut):
    """Launch PyUVM environment"""
    await uvm_root().run_test("HelloWorldTest")
```

Key takeaways from this minimal anatomy:
- `HelloWorldTest` inherits from `uvm_test`.
- `build_phase()` is synchronous, while `run_phase()` is asynchronous (`async def`).
- `uvm_root().run_test("TestName")` starts the entire UVM lifecycle from within a `@cocotb.test()` top-level wrapper.

---

## 6. Summary

| Feature | SystemVerilog UVM | PyUVM |
|---|---|---|
| **Underlying Engine** | Simulator Kernel | **cocotb** (Python + VPI/VHPI) |
| **Boilerplate / Macros** | Heavy (`` `uvm_component_utils ``) | **None** (Plain Python OOP) |
| **Hierarchy & Tree** | Component tree with parents | **Identical concept** (`uvm_component`) |
| **Factory & Overrides** | `uvm_factory` | **`uvm_factory` (Pythonic)** |
| **ConfigDB** | `uvm_config_db#(T)::set/get` | **`ConfigDB().set()` / `ConfigDB().get()`** |
| **Analysis Ports** | Ports, Exports, Imps (`uvm_analysis_imp`) | **Analysis Ports + `uvm_tlm_fifo` / Callbacks** |
| **Transactions & Sequences** | `uvm_sequence_item`, `uvm_sequence` | **`uvm_sequence_item`, `uvm_sequence`** |

---

**Next up:** PyUVM Part 2, The PyUVM Component Tree and Phasing Mechanism.
