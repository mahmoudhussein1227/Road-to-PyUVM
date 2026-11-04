# Road to PyUVM, cocotb part 1: Meet cocotb 

The Python OOP section is done, so we move to the second block of the roadmap: **cocotb**. PyUVM is built on top of cocotb, so every PyUVM testbench is really a cocotb test underneath. Before writing any test, this part answers three questions: what is cocotb, how does Python get inside a simulator, and what does the Makefile do to make it all work.

**Topics covered**

1. [What is cocotb?](#1-what-is-cocotb)
2. [A Python API supported by most simulators](#2-a-python-api-supported-by-most-simulators)
3. [How cocotb gets inside the simulator: VPI](#3-how-cocotb-gets-inside-the-simulator-vpi)
4. [What a cocotb testbench looks like](#4-what-a-cocotb-testbench-looks-like)
5. [The Makefile that configures the simulator](#5-the-makefile-that-configures-the-simulator)
6. [Why this matters for PyUVM](#6-why-this-matters-for-pyuvm)
7. [Installation recommendations](#7-installation-recommendations)

---

## 1. What is cocotb?

**cocotb** is an open-source Python framework for writing testbenches for hardware designs. The name comes from **CO**routine based **CO**simulation **T**est**B**ench.

- Your **design (DUT)** stays in Verilog, SystemVerilog, or VHDL.
- Your **testbench** is written in **Python**.
- The two run together inside a normal HDL simulator.

Instead of writing a SystemVerilog testbench, you write Python functions that drive the DUT's inputs, wait for clock edges, read its outputs, and check results with plain Python `assert` statements. You get Python's libraries, its readability, and its fast edit-and-run cycle.

A typical cocotb testbench needs **no extra HDL code**. The DUT is instantiated directly as the simulator's toplevel, without a wrapper testbench module.

---

## 2. A Python API supported by most simulators

cocotb is not tied to one simulator. It is a Python API that works with **any simulator that implements one of the industry-standard procedural interfaces**: VPI, VHPI, or FLI. In practice, that covers most of the simulators engineers use:

| Simulator | Notes |
|---|---|
| Icarus Verilog | Free and open source, a common choice for learning |
| Verilator | Free and open source, version requirements apply |
| GHDL / NVC | Free and open source VHDL simulators |
| Synopsys VCS | Commercial |
| Cadence Xcelium | Commercial |
| Siemens Questa / ModelSim | Commercial |
| Aldec Riviera-PRO / Active-HDL | Commercial |

Support levels and minimum versions differ and change between cocotb releases, so check the "Simulator Support" page of the cocotb documentation for your simulator and version.

The key benefit: **the same Python test runs on different simulators** with no code change. You switch simulators from the command line:

```
make SIM=icarus
make SIM=questa
```

Your testbench stays portable, which is useful when you prototype on a free simulator and later run on a commercial one.

---

## 3. How cocotb gets inside the simulator: VPI

### What are VPI, VHPI, and FLI?

Simulators expose a **C programming interface** that lets external code look at the design and react to simulation events:

| Interface | Full name | Used with |
|---|---|---|
| **VPI** | Verilog Procedural Interface | Verilog and SystemVerilog simulators (part of the IEEE 1364/1800 standards) |
| **VHPI** | VHDL Procedural Interface | VHDL simulators (part of the IEEE 1076 standard) |
| **FLI** | Foreign Language Interface | Proprietary interface of ModelSim and Questa for VHDL |

Through these interfaces, outside code can find signals by name, read and write their values, and register callbacks such as "call me on this signal's rising edge" or "call me after 10 ns".

### cocotb's layers

cocotb sits on top of these interfaces:

```
 Your Python test        async def my_test(dut): ...
        │
     cocotb              Python library: tests, triggers, scheduler
        │
      GPI                cocotb's C/C++ layer (Generic Procedural Interface)
        │                hides the differences between interfaces
 VPI / VHPI / FLI        the simulator's own C interface
        │
  HDL simulator + DUT
```

The **GPI** layer is cocotb's own abstraction. It gives Python one uniform view of the design, whichever interface the simulator provides underneath.

### What "injected into the simulator" means

cocotb is not a separate program that talks to the simulator over a network. It is **loaded into the simulator process**:

1. The Makefile starts the simulator and tells it to load a **shared library** that ships with cocotb (for example, the Icarus build loads `libcocotbvpi_icarus`).
2. The simulator loads that library through its VPI mechanism and calls its startup routine.
3. The library starts an **embedded Python interpreter** inside the simulator process.
4. Python imports cocotb and your test modules, and registers **callbacks** with the simulator (clock edges, time delays, value changes).
5. The simulation runs. Whenever a callback fires, the simulator hands control to Python, your test code runs until it waits on something again, and control returns to the simulator.
6. From Python, `dut.some_signal.value` reads and writes the design through VPI calls.

This is the "cosimulation" in cocotb's name: HDL and Python advance together, in lockstep, in one process. Your test only runs when the simulator yields to it, which is why cocotb tests wait on triggers like clock edges. We will see how in the coming parts.

---

## 4. What a cocotb testbench looks like

Here is a tiny taste. The details (coroutines, triggers, assertions) are the next parts of the series.

```python
# test_my_design.py
import cocotb

@cocotb.test()
async def my_first_test(dut):
    dut._log.info("Hello from Python, running inside the simulator")
```

- `@cocotb.test()` marks the function as a test that cocotb should find and run.
- `dut` is a handle to the design's toplevel. Signals appear as attributes, such as `dut.clk`.
- There is no `main`: the **simulator** starts the run, and cocotb calls your test.

Compare with SystemVerilog:

| | SystemVerilog testbench | cocotb testbench |
|---|---|---|
| Language | SystemVerilog | Python |
| Compiled with the design | Yes | No, Python runs inside the simulator |
| Toplevel | Testbench module wrapping the DUT | The DUT itself |
| Checks | `assert`, scoreboards | Python `assert`, Python libraries |
| Edit and rerun | Recompile | Edit the `.py` file and rerun |

---

## 5. The Makefile that configures the simulator

cocotb ships with a Makefile-based flow. You write a small Makefile that describes your project, and cocotb's own Makefile does the rest.

```make
# Makefile

# defaults
SIM ?= icarus
TOPLEVEL_LANG ?= verilog

VERILOG_SOURCES += $(PWD)/my_design.sv
# use VHDL_SOURCES for VHDL files

COCOTB_TOPLEVEL = my_design
COCOTB_TEST_MODULES = test_my_design

# include cocotb's make rules to take care of the simulator setup
include $(shell cocotb-config --makefiles)/Makefile.sim
```

### The main variables

| Variable | Meaning | Example |
|---|---|---|
| `SIM` | Which simulator to use | `icarus`, `verilator`, `questa`, `vcs`, `xcelium`, `ghdl` |
| `TOPLEVEL_LANG` | Language of the toplevel design | `verilog` or `vhdl` |
| `VERILOG_SOURCES` / `VHDL_SOURCES` | The design source files to compile | `$(PWD)/my_design.sv` |
| `COCOTB_TOPLEVEL` | Name of the toplevel module or entity | `my_design` |
| `COCOTB_TEST_MODULES` | Python module(s) containing the tests, **file name without `.py`** | `test_my_design` |
| `WAVES` (optional) | Set to `1` to dump waveforms | `WAVES = 1` |

Notes:

- `SIM ?= icarus` uses `?=`, which means "set this only if it was not already set". That lets you override it from the command line with `make SIM=questa`.
- `COCOTB_TOPLEVEL` and `COCOTB_TEST_MODULES` are the cocotb 2.x names. In cocotb 1.x they were called `TOPLEVEL` and `MODULE`. If you follow an older tutorial and see those, they are the same settings.
- To run several test files, list the modules separated by commas.

### The most important line

```make
include $(shell cocotb-config --makefiles)/Makefile.sim
```

This single line does the heavy lifting:

1. `cocotb-config --makefiles` is a command installed with cocotb. It prints the **path to the directory containing cocotb's Makefiles**, so you never hard-code a path.
2. `$(shell ...)` runs that command and substitutes its output.
3. `Makefile.sim` is cocotb's ready-made Makefile. Including it defines the targets to **compile your design for the chosen simulator, start the simulator, load cocotb through VPI (or VHPI/FLI), and run your tests**.

Think of your Makefile as the *configuration* and `Makefile.sim` as the *engine*. The engine reads your variables (`SIM`, sources, toplevel, test modules) and sets up the right simulator command line, library loading, and environment for you.

**The `include` line must come after all the variable definitions**, because `Makefile.sim` reads them when it is included.

### Optional: Python runners

cocotb also offers a pure-Python way to build and run simulations (the "Python runners"), as an alternative to the Makefile. This series uses the Makefile flow, since it is the most common starting point.

---

## 6. Why this matters for PyUVM

- **PyUVM is a Python library built on cocotb.** A PyUVM test is launched from a cocotb test, using the same simulator, the same VPI connection, and the same Makefile flow described here.
- Your PyUVM project's Makefile looks just like the one above. `COCOTB_TEST_MODULES` points at the Python file that starts the UVM test, and the rest is identical.
- Everything PyUVM does with the DUT (driving pins, waiting for clock edges, sampling outputs) goes through cocotb's handles and triggers, which are the next topics in the series.
- Because cocotb is simulator-independent, a PyUVM testbench can be moved between simulators by changing `SIM`.

---

## 7. Installation recommendations

There are several ways to install and set up cocotb depending on your operating system (Linux, macOS, Windows) and workflow preferences (native, virtual environments, Docker, or WSL). We recommend searching and picking the setup that best suits your OS and simulator setup.

### Recommended for Windows users: Miniconda / Conda

On Windows, managing compilers, C/C++ toolchains, Python versions, and Make can sometimes run into environment conflicts. A clean, reliable approach is to use **[Miniconda](https://docs.conda.io/en/latest/miniconda.html)**:

1. **Install Miniconda** on Windows.
2. **Create a dedicated environment** (e.g., Python 3.10 or 3.11):
   ```bash
   conda create -n cocotb_env python=3.11 -y
   conda activate cocotb_env
   ```
3. **Install cocotb** inside that environment:
   ```bash
   pip install cocotb
   ```
4. **Verify the installation**:
   ```bash
   cocotb-config --version
   ```

> **Note:** Many other installation methods exist (e.g., standard Python `venv`, WSL2 / Ubuntu on Windows, MSYS2, native Linux packages). Search the [official cocotb documentation](https://docs.cocotb.org/) and select the method that works best for your OS and installed simulator!

---

**Next up:** Part 8, Coroutines and `@cocotb.test()`.


