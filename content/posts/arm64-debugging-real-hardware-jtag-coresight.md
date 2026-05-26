---
title: "ARM64 Debugging on Real Hardware: JTAG, CoreSight, and Everything the Simulator Can't Tell You"
date: 2026-05-26T21:14:00+05:30
draft: false
tags: ["arm64", "debugging", "jtag", "coresight", "arm-ds", "openocd", "real-hardware", "firmware", "embedded"]
categories: ["Deep Dives"]
summary: "Everything we learned debugging on the FVP — now translated to real ARM hardware. JTAG connections, CoreSight trace, debug authentication, and the techniques that work the same (and the ones that don't)."
---

> "On the FVP, we had Iris — a Python script away from every register. On real hardware, we have JTAG probes, voltage levels, debug authentication gates, and the superpower the simulator never had: **CoreSight trace**."

This is the companion to our [ARM64 Debugging Handbook](/posts/arm64-debugging-handbook-iris-gdb-fvp/) — everything we did on the FVP, now translated to real silicon. If you've been following our firmware bring-up journey on the FVP, this is where those techniques meet physical hardware.

---

## Part I: The Real Hardware Debugging Arsenal

### JTAG vs SWD: Two Paths to the Same Goal


**JTAG (Joint Test Action Group)** is the granddaddy of hardware debugging — a 4-wire interface (TDI, TDO, TCK, TMS + GND) designed in 1985 for testing PCB connections. It evolved into the standard for embedded debug.

**SWD (Serial Wire Debug)** is ARM's 2-wire alternative (SWDIO, SWCLK + GND) — faster, simpler, and designed specifically for ARM CoreSight debug architecture. On most ARM systems, SWD is preferred.

| Feature | JTAG | SWD |
|---------|------|-----|
| Wires | 4 (TDI/TDO/TCK/TMS) | 2 (SWDIO/SWCLK) |
| Speed | Up to 10 MHz typical | Up to 50 MHz with DAP v2 |
| Multi-device | Daisy chain (complex) | Single-drop (simple) |
| ARM CoreSight | Supported | Native |
| When to use | Legacy systems, FPGA | Modern ARM (Cortex-A/R/M) |

On the RD-N2 or similar ARM platforms, you'll typically use **SWD for Cortex-M cores (like SCP)** and **JTAG or SWD for Cortex-A/Neoverse cores** (board-dependent).

### The CoreSight Debug Architecture

ARM's CoreSight is the debug infrastructure inside every ARM SoC. Think of it as the hardware equivalent of Iris — but distributed across the chip.

**Key components:**

`
┌─────────────────────────────────────────────────────────┐
│                    ARM SoC                              │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  CPU 0   │  │  CPU 1   │  │  SCP     │            │
│  │  (A/N)   │  │  (A/N)   │  │(Cortex-M)│            │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘            │
│       │ETM          │ETM          │MTB                 │
│       └─────────┬───┴─────────┬───┘                   │
│                 │             │                        │
│              ┌──▼─────────────▼──┐                    │
│              │   Debug APB Bus   │                    │
│              │   (CoreSight)     │                    │
│              └──┬─────────┬──────┘                    │
│                 │ETF      │CTI                         │
│                 │(Trace)  │(Cross Trigger)            │
│                 └────┬────┘                           │
│                      │                                 │
│              ┌───────▼──────┐                         │
│              │  Debug Access │                         │
│              │  Port (DAP)   │                         │
│              └───────┬───────┘                         │
└──────────────────────┼─────────────────────────────────┘
                       │
                  JTAG/SWD pins ──► To debug probe

`

- **ETM (Embedded Trace Macrocell)**: Records every instruction executed by a CPU (real-time, non-invasive)
- **ETF (Embedded Trace FIFO)**: On-chip trace buffer
- **ETR (Embedded Trace Router)**: Routes trace to DRAM
- **CTI (Cross-Trigger Interface)**: "Halt CPU 1 when CPU 0 hits breakpoint"
- **DAP (Debug Access Port)**: The gatekeeper — JTAG/SWD talks to DAP, DAP talks to everything

**FVP equivalent:** On FVP, Iris gave us direct access to all components. On real hardware, we go through DAP → CoreSight bus → component.

### Debug Probes: Your Hardware-to-PC Bridge


| Probe | Speed | Features | Price | Best For |
|-------|-------|----------|-------|----------|
| **ARM DSTREAM** | Ultra-fast | CoreSight trace, 4GB capture, streaming | \$\$\$\$\$ | Production firmware teams |
| **ULINKpro** | Fast | CoreSight trace, integrated with Keil/DS | \$\$\$ | ARM Development Studio users |
| **Segger J-Link** | Fast | Excellent GDB support, unlimited breakpoints | \$\$-\$\$\$ | Cross-platform, Linux-friendly |
| **OpenOCD adapters** | Moderate | Open source, wide board support | \$-\$\$ | Makers, budget projects |
| **pyOCD + CMSIS-DAP** | Moderate | Python-scriptable (Iris-like!) | \$-\$\$ | Automation, CI/CD |

**For the work we did on FVP:** A DSTREAM or ULINKpro would be ideal (trace capture), but a J-Link or OpenOCD adapter with GDB works for 90% of debugging tasks.

### ARM Development Studio vs OpenOCD

**ARM Development Studio (Arm DS)**:
- Commercial ($\$\$), powerful, official ARM tool
- Native CoreSight trace decode
- Best symbol handling for ARM firmware
- Scripting via Python (like our Iris scripts!)

**OpenOCD (Open On-Chip Debugger)**:
- Free, open source
- Excellent GDB integration
- Limited trace support (mostly halt-mode debug)
- Config files for hundreds of boards

**For our FVP debugging workflow → real hardware:**

| FVP (Iris + Python) | Real Hardware Equivalent |
|---------------------|--------------------------|
| model.get_target('cpu0') | OpenOCD: 	argets / Arm DS: connection browser |
| cpu0.read_register('PC') | OpenOCD: eg pc / Arm DS: register view |
| cpu0.read_memory(addr, len) | OpenOCD: mdw addr count / GDB: x/Nx addr |
| model.stop() | OpenOCD: halt / Arm DS: click "Halt" |
| model.step(1) | OpenOCD: step / GDB: stepi |
| Load symbols | GDB: symbol-file bl31.elf + dd-symbol-file |

---

## Part II: Connecting to a Real Board

### Physical Connection

On the FVP, we ran a PowerShell script and got port 7100. On real hardware, we need wires.

**Typical JTAG/SWD header (20-pin ARM standard):**

`
 1  VTref ─────► 3  nTRST      ┐
 2  NC          4  GND         │ JTAG signals
 5  TDI ────────► 6  GND       │ (or SWD: SWDIO/SWCLK
 7  TMS/SWDIO ─► 8  GND       │  on pins 7/9)
 9  TCK/SWCLK ─► 10 GND        │
11  RTCK ◄────  12 GND        │
13  TDO/SWO ◄─  14 GND        │
15  nRESET     16 GND         │
17  DBGRQ      18 GND         │
19  DBGACK     20 GND         ┘
`

**Pin 1 (VTref)** is critical — it tells the probe what voltage level the target uses (1.8V, 3.3V). **Always connect this first**.

**For SWD-only (2-wire):**
- Connect: VTref (pin 1), GND (any), SWDIO (pin 7), SWCLK (pin 9), optional nRESET (pin 15)

**Level shifters:** If your probe is 3.3V and the board is 1.8V, you need a level shifter or a probe with adaptive voltage (like J-Link).

### Power-On and Reset Behavior

**FVP:** Starts instantly, CPU at known state.

**Real hardware:** Power-on sequence matters.


1. **Power on the board first** (SCP starts, initializes clocks/power/DDR)
2. **Then connect the debug probe** (JTAG pins may not be active until SCP configures them)
3. **Or:** Hold the board in reset, connect probe, then release reset (for debugging boot ROM)

**Warm reset vs cold reset:**
- **Cold reset:** Full power cycle — all state lost
- **Warm reset:** nRESET signal — debug logic may survive, registers preserved (useful for post-mortem analysis!)

**Our FVP crash scenario:** When StandaloneMM crashed, we just connected Iris and read registers. On real hardware, if the system already crashed and watchdog reset it, you need to catch it BEFORE the reset (using a breakpoint or halt-on-reset).

### JTAG Scan Chain: Discovering Cores

On multi-core ARM systems, all debug APs are on a single JTAG scan chain. When you connect, you need to discover what's there.

**OpenOCD automatic scan:**
`bash
openocd -f interface/jlink.cfg -c "transport select swd" -c "adapter speed 10000" -c "init" -c "scan_chain"
`

**Output:**
`
TapName             Enabled  IdCode     Expected   IrLen IrCap IrMask
-- ------------------ -------- ---------- ---------- ----- ----- ------
 0 rdn2.dap          Y        0x6ba00477 0x6ba00477     4 0x01  0x0f
`

This tells you there's one DAP (Debug Access Port) with ID  x6ba00477 (ARM CoreSight DP).

**Inside that DAP, there are multiple Access Ports (APs):**
- **AP 0:** Usually the main CoreSight AP (for accessing debug components)
- **AP 1, 2, 3...:** Often mapped to different CPU clusters or memory regions
- **AMBA AHB-AP vs APB-AP:** Different bus types, but you mostly don't care — the tools handle it

**Selecting a target:**
`
openocd> targets
    TargetName         Type       Endian TapName            State
--  ------------------ ---------- ------ ------------------ -------
 0* rdn2.cpu0         aarch64    little rdn2.dap           running
 1  rdn2.cpu1         aarch64    little rdn2.dap           running
 2  rdn2.scp          cortex_m   little rdn2.dap           running
`

`
openocd> targets rdn2.cpu0   # Switch to CPU0
openocd> halt                 # Stop it
openocd> reg pc               # Read PC
pc (/64): 0x00000000FF01843C
`

**FVP equivalent:** cpu0 = model.get_target('component.RD_N2.css.cluster0.cpu0')

### Multi-Core Debugging Example (OpenOCD)

**Scenario:** StandaloneMM crashes on CPU0. You want to check what all CPUs are doing.

`tcl
# OpenOCD config for RD-N2 (example)
adapter driver jlink
transport select swd
adapter speed 10000

# Define the DAP
swd newdap rdn2 dap -irlen 4 -expected-id 0x6ba00477

# Define CPU targets
dap create rdn2.dap -chain-position rdn2.dap
target create rdn2.cpu0 aarch64 -dap rdn2.dap -ap-num 0 -coreid 0
target create rdn2.cpu1 aarch64 -dap rdn2.dap -ap-num 0 -coreid 1
target create rdn2.scp cortex_m -dap rdn2.dap -ap-num 1

# Halt all on connect
rdn2.cpu0 configure -event reset-assert-post {halt}
`

**Debug session:**
`
$ openocd -f rdn2.cfg
Open On-Chip Debugger 0.12.0
Info : Listening on port 3333 for gdb connections

# In another terminal:
$ telnet localhost 4444
> targets
 0* rdn2.cpu0  aarch64  running
 1  rdn2.cpu1  aarch64  running
 2  rdn2.scp   cortex_m running

> halt        # Halt CPU0 (current target)
target halted in ARM64 state due to debug-request, current mode: EL3
cpsr: 0x600003CD pc: 0xFF01843C

> reg pc
pc (/64): 0x00000000FF01843C

> targets rdn2.cpu1
> halt
> reg pc
pc (/64): 0x0000000080220000  # CPU1 is in Linux kernel

> targets rdn2.scp
> halt
> reg pc
pc (/32): 0x0BD00124  # SCP is idle
`

---

## Part III: Register Access on Real Hardware

### Reading System Registers: Same Registers, Different Access Rules

The registers we read on FVP (ESR_EL3, ELR_EL3, FAR_EL1, SCR_EL3) exist identically on real hardware. The difference: **access control**.

**On FVP/Iris:** We could always read any register at any EL, even from Python while the system was running.

**On real hardware:**
1. **Core must be halted** to read most system registers via JTAG
2. **Debug authentication gates** control what you can access
3. **Security state** affects visibility (secure registers hidden if invasive secure debug is disabled)

**OpenOCD register read (same as FVP):**
`
> halt
> reg ESR_EL3
esr_el3 (/64): 0x5E000000

> reg ELR_EL3
elr_el3 (/64): 0x00000000FF207DA4

> reg FAR_EL1
far_el1 (/64): 0xFFFFFFFFFFFFFD90
`

**GDB register read:**
`gdb
(gdb) info registers
pc             0xff01843c
sp             0x4034fe40
...
`

`gdb
(gdb) info registers system
CurrentEL      0x0000000c  (EL3)
ESR_EL3        0x5e000000
ELR_EL3        0xff207da4
SCR_EL3        0x00000731  (bit 0 = 1 → NS=1)
`

This is EXACTLY what we did with Iris: cpu0.read_register('ESR_EL3').

### Debug Authentication: The Gates You Must Unlock

ARM's CoreSight has authentication signals that gate debug access. If these aren't asserted, you can't access certain features.

**Key signals (usually set via board DIP switches or firmware):**

| Signal | What It Gates | If Disabled |
|--------|---------------|-------------|
| **DBGEN** | Invasive debug (halt, step, breakpoints) | Can't halt CPU, can't read registers |
| **NIDEN** | Non-invasive debug (trace, PMU) | Can't capture ETM trace |
| **SPIDEN** | Secure invasive debug | Can't read secure memory or secure-world registers |
| **SPNIDEN** | Secure non-invasive debug | Can't trace secure code |

**Our StandaloneMM crash was in Secure EL1.** To debug it on real hardware, we'd need **SPIDEN=1** (secure invasive debug enabled).

**Checking authentication status (OpenOCD):**
`
> dap info
AP # 0x00000000
  Type is MEM-AP AHB3
  MEM-AP BASE 0x80010000
  Debug Base Address: 0x80010000
  Access Port IDR: 0x24770011
