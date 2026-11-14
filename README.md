# Road to PyUVM 🚀

Code examples, notes, and resources for the **Road to PyUVM** series.

## About

This repository contains comprehensive notes and hands-on code examples accompanying the **Road to PyUVM** LinkedIn series, where we explore the journey from Python fundamentals to verification using [PyUVM](https://github.com/pyuvm/pyuvm) and [cocotb](https://www.cocotb.org/).

## Series Contents

### 🐍 Python OOP

- **Part 1: Python Classes** — [`Python OOP/01_python_classes.md`](Python%20OOP/01_python_classes.md)
  - Class declaration and blueprints
  - Creating objects and memory layout
  - Object references vs. copying
  - Why it matters for PyUVM

- **Part 2: Attributes and `self`** — [`Python OOP/02_attributes_and_self.md`](Python%20OOP/02_attributes_and_self.md)
  - Class attributes vs. instance attributes
  - Understanding `self` and referencing the current instance
  - Common pitfalls (mutable class attributes, namespace shadowing)
  - Relevance to PyUVM components and transactions

- **Part 3: Constructors and `__str__`** — [`Python OOP/03_constructors_and_str.md`](Python%20OOP/03_constructors_and_str.md)
  - The constructor (`__init__`) and parameter defaults
  - Dunder methods overview
  - String representation (`__str__` vs. `__repr__`)
  - Common pitfalls and PyUVM verification relevance

- **Part 4: User-Defined Methods** — [`Python OOP/04_user_defined_methods.md`](Python%20OOP/04_user_defined_methods.md)
  - Method definition and the `self` rule
  - Parameters, return values, and mutating state
  - Calling methods from other methods (`self.method()`)
  - Methods vs. functions and PyUVM verification relevance

- **Part 5: Accessing Class Members** — [`Python OOP/05_accessing_class_members.md`](Python%20OOP/05_accessing_class_members.md)
  - The dot operator for attributes and methods
  - Dynamic access (`getattr`, `setattr`, `hasattr`, `delattr`)
  - Inspecting objects (`dir()`, `__dict__`)
  - Python encapsulation: conventions, `_single_underscore`, and `__name_mangling`
  - PyUVM relevance (component hierarchies, logging, transaction fields)

- **Part 6: Inheritance** — [`Python OOP/06_inheritance.md`](Python%20OOP/06_inheritance.md)
  - Parent and child classes ("is-a" relationship)
  - Extending constructors with `super().__init__()`
  - Method overriding and extending parent methods
  - Relationship checks (`isinstance`, `issubclass`) and MRO
  - Multiple inheritance and PyUVM base class architecture

### ⚡ cocotb

- **Part 1: Meet cocotb** — [`cocotb/01_cocotb_intro.md`](cocotb/01_cocotb_intro.md)
  - What is cocotb and cosimulation
  - Simulator support & VPI interface
  - Cocotb testbench anatomy
  - Simulator configuration via Makefile
  - PyUVM connection & significance

- **Part 2: Coroutines and `@cocotb.test()`** — [`cocotb/02_coroutines_and_cocotb_test.md`](cocotb/02_coroutines_and_cocotb_test.md)
  - What are coroutines (`async def`, `await`) and concurrency
  - Why hardware verification needs coroutines
  - Decorators overview and `@cocotb.test()`
  - Full "Hello, world" test example & test options
  - Common pitfalls and PyUVM relevance

- **Part 3: Triggers and `await`** — [`cocotb/03_triggers.md`](cocotb/03_triggers.md)
  - Coroutines vs. Triggers (awaitable types)
  - What happens when you await a trigger
  - Common triggers (`Timer`, `RisingEdge`, `FallingEdge`, `Edge`, etc.)
  - Triggers vs. SystemVerilog timing events
  - Timing nuances, common pitfalls, and PyUVM significance

- **Part 4: Accessing DUT Signals with `.value`** — [`cocotb/04_accessing_dut_signals.md`](cocotb/04_accessing_dut_signals.md)
  - The `dut` object and signal handles vs. values
  - Reading signals and four-state types (`LogicArray`, conversions)
  - Writing signals and scheduled update timing
  - Full driving and sampling example
  - Common pitfalls and PyUVM driver/monitor relevance

- **Part 5: Tasks, Concurrency, `gather`, and `select`** — [`cocotb/05_tasks_and_concurrency.md`](cocotb/05_tasks_and_concurrency.md)
  - Tasks as independent parallel processes
  - Spawning background tasks with `cocotb.start_soon()`
  - Task handles, awaiting, status checks, and `.cancel()`
  - SystemVerilog `fork / join` mapping
  - Synchronizing with `gather` and racing with `select`
  - PyUVM component `run_phase` architecture

- **Part 6: Checks with `assert` and Inter-Task Communication with `Queue`** — [`cocotb/06_assert_and_queue.md`](cocotb/06_assert_and_queue.md)
  - How cocotb tests pass or fail
  - The `assert` statement and descriptive failure messages
  - Safe inter-task communication vs. polling plain lists
  - Asynchronous FIFO queue (`cocotb.queue.Queue`)
  - Producer-consumer pattern with scoreboard verification
  - Foundation for PyUVM TLM FIFOs and analysis ports

### 🏗️ PyUVM

- **Part 1: Brief Introduction to PyUVM** — [`pyuvm/01_pyuvm_intro.md`](pyuvm/01_pyuvm_intro.md)
  - What is PyUVM and how it builds on cocotb
  - Core UVM concepts retained (Component tree, Factory, ConfigDB, Phasing)
  - Key differences from SystemVerilog UVM (no macros, simplified TLM, analysis implementations)
  - Series scope: Focusing on PyUVM vs. SV differences
  - Minimal PyUVM "Hello World" anatomy

- **Part 2: UVM Phases and Phasing Execution** — [`pyuvm/02_uvm_phases.md`](pyuvm/02_uvm_phases.md)
  - Phases as methods on `uvm_component`
  - Non-time-consuming (`def`) vs. time-consuming (`async def`) phases
  - Initiating phasing via `await uvm_root().run_test("test_name")`
  - Removal of the `uvm_phase phase` argument
  - Real-world APB environment examples (`connect_phase` and `run_phase`)

*More topics coming soon...*

## Follow the Series

📖 Follow along on LinkedIn: [Mahmoud Hussein](https://www.linkedin.com/in/mahmoud-hussein-04b712191/)

## Getting Started

Clone the repository:

```bash
git clone https://github.com/mahmoudhussein1227/Road-to-PyUVM.git
```

## License

This project is open source and available for educational purposes.
