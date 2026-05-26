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

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   ARM SoC with CoreSight Debug Architecture                      │
│                        (e.g., RDN2 / Neoverse Platform)                         │
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │   CPU Core 0    │  │   CPU Core 1    │  │   CPU Core 2    │  │ CPU Core 3 │ │
│  │  (Neoverse V1)  │  │  (Neoverse V1)  │  │  (Neoverse V1)  │  │(Neoverse V1│ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤  ├────────────┤ │
│  │ Debug Interface │  │ Debug Interface │  │ Debug Interface │  │Debug IF    │ │
│  │ • Breakpoints   │  │ • Breakpoints   │  │ • Breakpoints   │  │• BP/WP     │ │
│  │ • Watchpoints   │  │ • Watchpoints   │  │ • Watchpoints   │  │• Halt      │ │
│  │ • Halt control  │  │ • Halt control  │  │ • Halt control  │  │            │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └──────┬─────┘ │
│           │                    │                    │                   │       │
│  ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐  ┌──────▼─────┐ │
│  │ ETM (0x840000)  │  │ ETM (0x850000)  │  │ ETM (0x860000)  │  │ETM(0x87000)│ │
│  │ Trace Macrocell │  │ Trace Macrocell │  │ Trace Macrocell │  │Trace Macro │ │
│  │ Captures:       │  │ Captures:       │  │ Captures:       │  │Captures:   │ │
│  │ • PC samples    │  │ • PC samples    │  │ • PC samples    │  │• PC sample │ │
│  │ • Branch targets│  │ • Branch targets│  │ • Branch targets│  │• Branches  │ │
│  │ • Exceptions    │  │ • Exceptions    │  │ • Exceptions    │  │• Exception │ │
│  │ • Address range │  │ • Address range │  │ • Address range │  │• Filters   │ │
│  │   filters       │  │   filters       │  │   filters       │  │            │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └──────┬─────┘ │
│           │                    │                    │                   │       │
│  ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐  ┌──────▼─────┐ │
│  │ CTI (0x842000)  │  │ CTI (0x852000)  │  │ CTI (0x862000)  │  │CTI(0x87200)│ │
│  │ Cross Trigger   │  │ Cross Trigger   │  │ Cross Trigger   │  │Cross Trig  │ │
│  │ • Halt synchro  │  │ • Halt synchro  │  │ • Halt synchro  │  │• Halt sync │ │
│  │ • Trace start   │  │ • Trace start   │  │ • Trace start   │  │• Trace trig│ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └──────┬─────┘ │
│           └──────────────────┬──┴──────────────┬──────┴──────────────────┘       │
│                              │                 │                                 │
│                    ┌─────────▼─────────────────▼─────────┐                       │
│                    │    Trace Funnel (0x8C0000)          │                       │
│                    │  Merges 4 ETM streams into single   │                       │
│                    │  • Time-multiplexed packets         │                       │
│                    │  • Adds CPU core ID tags            │                       │
│                    │  • Priority-based arbitration       │                       │
│                    └─────────────────┬───────────────────┘                       │
│                                      │                                           │
│            ┌─────────────────────────┼─────────────────────────┐                 │
│            │                         │                         │                 │
│  ┌─────────▼─────────┐   ┌───────────▼────────────┐  ┌────────▼─────────────┐   │
│  │  ETF (0x8D0000)   │   │  ETR (0x8E0000)        │  │  TPIU (0x8F0000)     │   │
│  │  On-chip FIFO     │   │  Trace to DRAM         │  │  Trace Port Out      │   │
│  │  • 8KB-32KB       │   │  • 1MB-256MB buffer    │  │  • Streaming to      │   │
│  │  • Circular       │   │  • AXI write to RAM    │  │    external probe    │   │
│  │  • Last N instr   │   │  • Long captures       │  │  • Real-time decode  │   │
│  └───────────────────┘   └────────────────────────┘  └──────────────────────┘   │
│                                      │                                           │
│         ┌────────────────────────────┴────────────────────────────┐              │
│         │    CoreSight ROM Table (0x80000000)                     │              │
│         │  Component discovery — "Where are debug components?"    │              │
│         │  • Lists ETM, CTI, funnel, ETF, ETR base addresses      │              │
│         │  • Component type identification (via PIDR/CIDR)        │              │
│         └────────────────────────┬────────────────────────────────┘              │
│                                  │                                               │
│         ┌────────────────────────▼────────────────────────────┐                  │
│         │    Debug Access Port (DAP) — MEM-AP                 │                  │
│         │  • Register access (read/write CPU registers)       │                  │
│         │  • Memory access (read/write system RAM, MMIO)      │                  │
│         │  • Authentication (DBGEN, NIDEN, SPIDEN, SPNIDEN)   │                  │
│         │  • Multi-AP support (AP0=CPU, AP1=SCP, etc.)        │                  │
│         └────────────────────────┬────────────────────────────┘                  │
└──────────────────────────────────┼───────────────────────────────────────────────┘
                                   │
                     ┌─────────────▼─────────────┐
                     │     JTAG or SWD Pins      │
                     │  ┌──────────────────────┐ │
                     │  │ JTAG: TDI, TDO, TCK  │ │
                     │  │       TMS, TRST      │ │
                     │  │  Or                  │ │
                     │  │ SWD:  SWDIO, SWCLK   │ │
                     │  └──────────────────────┘ │
                     └─────────────┬─────────────┘
                                   │
                     ┌─────────────▼─────────────┐
                     │      Debug Probe          │
                     │  ARM DSTREAM (traces)     │
                     │  Segger J-Link (debug)    │
                     │  OpenOCD adapter (budget) │
                     │  ┌──────────────────────┐ │
                     │  │  USB 3.0 to host PC  │ │
                     │  └──────────────────────┘ │
                     └─────────────┬─────────────┘
                                   │
                     ┌─────────────▼─────────────┐
                     │  Host PC Debug Software   │
                     │  • ARM Development Studio │
                     │  • OpenOCD + GDB          │
                     │  • pyOCD (Python API)     │
                     │  • Segger Ozone           │
                     └───────────────────────────┘
```

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

On the FVP, we ran a PowerShell script and got port 7100. On real hardware, we need **physical wires** and **correct pinout**.

**Standard 20-pin ARM JTAG/SWD connector (Cortex Debug Connector):**

```
┌─────────────────────────────────────────────────────────────────────┐
│            20-Pin ARM Cortex Debug Header (Top View)                │
│                                                                     │
│   Pin 1 (Red stripe) ◄─────────────────────────────┐               │
│                                                     │               │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐       │               │
│   │ 1 │ 3 │ 5 │ 7 │ 9 │11 │13 │15 │17 │19 │       │               │
│   ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤  ◄────┘ Keyed notch   │
│   │ 2 │ 4 │ 6 │ 8 │10 │12 │14 │16 │18 │20 │       (orientation)   │
│   └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                       │
│                                                                     │
│  Pin │ Signal    │ Direction │ Purpose                             │
│  ────┼───────────┼───────────┼─────────────────────────────────── │
│   1  │ VTref     │  Input ◄  │ **Target voltage reference**        │
│   2  │ NC        │     -     │  (Not connected or optional)        │
│   3  │ nTRST     │  Output ► │  JTAG Test Reset (optional)         │
│   4  │ GND       │  Ground   │  Ground                             │
│   5  │ TDI       │  Output ► │  JTAG Data In (or NC for SWD)       │
│   6  │ GND       │  Ground   │  Ground                             │
│   7  │ SWDIO/TMS │  I/O ◄►   │  **SWD Data (or JTAG Mode Select)** │
│   8  │ GND       │  Ground   │  Ground                             │
│   9  │ SWCLK/TCK │  Output ► │  **SWD Clock (or JTAG Clock)**      │
│  10  │ GND       │  Ground   │  Ground                             │
│  11  │ RTCK      │  Input ◄  │  Return clock (for adaptive timing) │
│  12  │ GND       │  Ground   │  Ground                             │
│  13  │ SWO/TDO   │  Input ◄  │  Trace out / JTAG Data Out          │
│  14  │ GND       │  Ground   │  Ground                             │
│  15  │ nRESET    │  I/O ◄►   │  Target reset (bidirectional)       │
│  16  │ GND       │  Ground   │  Ground                             │
│  17  │ DBGRQ     │  Output ► │  Debug request (optional)           │
│  18  │ GND       │  Ground   │  Ground                             │
│  19  │ DBGACK    │  Input ◄  │  Debug acknowledge (optional)       │
│  20  │ GND       │  Ground   │  Ground                             │
└─────────────────────────────────────────────────────────────────────┘
```

**Minimal SWD connection (2-wire debug, most common):**

```
┌──────────────────┐                   ┌──────────────────────┐
│   J-Link Probe   │                   │   RDN2 Dev Board     │
│   (or DSTREAM)   │                   │   Debug Header J10   │
│                  │                   │                      │
│  Pin 1  VTref    ├──────────────────►│ Pin 1  VTref (3.3V)  │ ◄─ Sense voltage
│  Pin 7  SWDIO    ├◄─────────────────►│ Pin 7  SWDIO         │ ◄─ Bidirectional data
│  Pin 9  SWCLK    ├──────────────────►│ Pin 9  SWCLK         │ ◄─ Clock output
│  Pin 4/6/8 GND   ├──────────────────►│ Pin 4  GND           │ ◄─ Common ground
│  Pin 15 nRESET   ├◄─────────────────►│ Pin 15 nRESET        │ ◄─ Reset (optional)
│                  │                   │                      │
│      USB 3.0     │                   │    Power supply      │
│        │         │                   │         │            │
│        ▼         │                   │         ▼            │
│   Host PC        │                   │    12V/5V adapter    │
└──────────────────┘                   └──────────────────────┘
```

**Critical setup rules:**
1. **VTref MUST be connected** — probe needs to know target voltage (1.8V, 3.3V)
2. **GND must be solid** — bad ground = unreliable connection, random failures
3. **Short wires** (<15cm ideal) — longer wires introduce signal integrity issues at high speed
4. **Power the board BEFORE connecting probe** (unless debugging boot ROM)

**Common connection failures:**

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Error: SWDIO line stuck low` | No VTref, board unpowered | Connect VTref, power board |
| `Error: Cannot read IDCODE` | Wrong pins, bad ground, wrong transport | Check wiring, verify SWD selected |
| `Error: JTAG scan failed` | Board in reset, SCP not started | Release nRESET, wait for SCP boot |
| `Error: Target voltage is 0.00V` | VTref not connected | Connect pin 1 to target voltage rail |
| Intermittent failures | Long wires, poor contact | Use shorter ribbon cable, check connector |

---

### Power-On Sequence and Boot Timing

**FVP:** Starts instantly at `main()`, CPU at known EL3 state.

**Real hardware:** Multi-stage boot taking **10-60 seconds**, and debug access depends on which stage you're in.

**Complete boot sequence diagram:**

```
Time
(sec)
  0 ────► Power on (12V adapter connected)
  │       ┌──────────────────────────────────────────────┐
  │       │  Power Regulators Stabilize                  │
  │       │  • 12V → 5V, 5V → 3.3V, 3.3V → 1.8V         │
  │       │  • VTref becomes valid (~100ms)              │
  │       └──────────────────────────────────────────────┘
  1 ────► SCP (System Control Processor) ROM starts
  │       ┌──────────────────────────────────────────────┐
  │       │  SCP ROM Bootloader (Cortex-M7 @ 0x0)       │
  │       │  • Reads fuses, checks secure boot          │
  │       │  • Loads SCP firmware from flash            │
  │       │  • UART0 not initialized yet → Silent       │
  │       └──────────────────────────────────────────────┘
  3 ────► SCP firmware starts
  │       ┌──────────────────────────────────────────────┐
  │       │  SCP RAM Firmware (Cortex-M7 @ 0xBD06000)   │
  │       │  • Initializes UART0 (MCP console)          │
  │       │  • Configures clocks (CMN-700, PCIe, DDR)   │
  │       │  • Initializes DDR4 memory controller       │
  │       │  • Powers on AP (application processor)     │
  │       │  • ✅ JTAG/SWD becomes accessible HERE      │
  │       └──────────────────────────────────────────────┘
  │         **First output:** "SCP Firmware v2.11"
  8 ────► TF-A BL1 (Boot Loader stage 1)
  │       ┌──────────────────────────────────────────────┐
  │       │  TF-A Trusted Boot Firmware @ 0x0            │
  │       │  • Secure boot verification                 │
  │       │  • Loads BL2 from flash                     │
  │       │  • UART1 initialized (AP console)           │
  │       └──────────────────────────────────────────────┘
  │         **Output:** "NOTICE: BL1: v2.11(debug)"
 10 ────► TF-A BL2 (Firmware loader)
  │       ┌──────────────────────────────────────────────┐
  │       │  • Loads BL31, BL32, BL33 from flash        │
  │       │  • Verifies signatures (if secure boot on)  │
  │       └──────────────────────────────────────────────┘
 12 ────► TF-A BL31 (Secure monitor) + BL32 (StandaloneMM)
  │       ┌──────────────────────────────────────────────┐
  │       │  • BL31 runtime firmware @ 0xFF000000       │
  │       │  • BL32 (StandaloneMM) @ 0xFF200000         │
  │       │  • SMC handler installed                    │
  │       └──────────────────────────────────────────────┘
  │         **Output:** "NOTICE: BL31: StandaloneMM loaded"
 15 ────► BL33 (UEFI firmware)
  │       ┌──────────────────────────────────────────────┐
  │       │  • EDK2 UEFI @ 0xE0000000                   │
  │       │  • PEI phase, DXE phase drivers load        │
  │       │  • UEFI variables via StandaloneMM          │
  │       └──────────────────────────────────────────────┘
  │         **Output:** "UEFI firmware (version 1.0)"
 25 ────► Grub boot loader
  │       ┌──────────────────────────────────────────────┐
  │       │  • Loads Linux kernel from disk             │
  │       └──────────────────────────────────────────────┘
 30 ────► Linux kernel
  │       ┌──────────────────────────────────────────────┐
  │       │  • Kernel decompression @ 0x80080000        │
  │       │  • MMU setup, CPU feature detection         │
  │       │  • ⚠️ Our SVE trap happened here            │
  │       └──────────────────────────────────────────────┘
 45 ────► Linux userspace
          ┌──────────────────────────────────────────────┐
          │  • systemd starts services                  │
          │  • Login prompt appears                     │
          └──────────────────────────────────────────────┘
```

**Debug access windows:**

| Boot Stage | JTAG/SWD Access | What You Can Debug |
|------------|-----------------|-------------------|
| 0-3 sec (Power-on, SCP ROM) | ❌ Not ready | N/A — clocks not initialized |
| 3-8 sec (SCP firmware) | ✅ **Yes**, but only SCP core | SCP firmware, CMN-700 init, DDR setup |
| 8+ sec (TF-A BL1 onwards) | ✅ **Yes**, all AP cores | TF-A, UEFI, StandaloneMM, Linux |

**To debug SCP boot (before 3 seconds):**
```bash
# Hold board in reset, connect probe
openocd -f board/rdn2.cfg -c "init; reset halt"

# System stays at SCP ROM entry (0x0)
> reg pc
pc: 0x00000000

# Now step through SCP ROM bootloader
> step; reg pc
```

**To debug normal boot (TF-A onwards):**
```bash
# Let board boot normally, connect anytime after 8 seconds
openocd -f board/rdn2.cfg

# Halt and inspect
> halt
> reg pc
pc: 0xFF018234  ← We're in BL31 runtime firmware
```

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
  CoreSight Debug Auth: DBGEN=1 NIDEN=1 SPIDEN=1 SPNIDEN=1  ← All enabled!
```

If SPIDEN=0, you'll see zeros when trying to read secure memory or get access denied errors.

**How to enable on development boards:**
- **DIP switches** (e.g., "Secure Debug Enable" switch on RD-N2 dev board)
- **Firmware configuration** (some SCP firmware reads a GPIO to enable SPIDEN)
- **JTAG command** (some platforms allow setting via DAP registers, but this is rare)

---

## Part IV: Symbol Loading and Source Debugging on Real Hardware

### Loading ELF Symbols: Same Files, Different Tool

On FVP, we used `aarch64-linux-gnu-addr2line` in Python scripts. On real hardware with GDB, we load symbols directly.

**GDB symbol loading (same .elf files from our builds):**

```gdb
# Connect to OpenOCD's GDB server
$ gdb-multiarch
(gdb) target extended-remote localhost:3333

# Load BL31 symbols at its load address
(gdb) symbol-file ~/rdn2/tf-a/build/rdn2/debug/bl31/bl31.elf
Reading symbols from bl31.elf...

# The PC we read was 0xFF01843C
(gdb) info symbol 0xFF01843C
plat_panic_handler in section .text of bl31.elf
```

**For position-independent code or relocated modules:**

```gdb
# StandaloneMM is loaded by TF-A at runtime (we found it at 0xFF200000)
(gdb) add-symbol-file ~/rdn2/edk2/Build/SgiMmStandalone/DEBUG_GCC5/AARCH64/StandaloneMmCore.dll 0xFF200000
add symbol table from file "StandaloneMmCore.dll" at
        .text_addr = 0xFF200000
(y or n) y
Reading symbols from StandaloneMmCore.dll...

# Now we can resolve the faulting address
(gdb) info symbol 0xFF207DA0
_ModuleEntryPoint + 160 in section .text
```

**This is exactly what we did with addr2line on FVP:**
```python
addr_to_source('StandaloneMmCore.dll', 0xFF207DA0)
# → _ModuleEntryPoint at ModuleEntryPoint.S:45
```

### Source-Level Debugging

**On FVP:** We used `objdump` to disassemble around crash addresses.

**On real hardware with GDB:**

```gdb
# Disassemble around the fault
(gdb) x/10i 0xFF207DA0
   0xff207d90:  ldr    x8, [x8]
   0xff207d94:  cbz    w8, 0xff207da8
   0xff207d98:  mov    x0, #0x2
   0xff207d9c:  blr    x8
=> 0xff207da0:  str    x0, [sp, #-16]!    ← We're here (PC)
   0xff207da4:  mov    x0, #0x0
   0xff207da8:  bl     0xff207eb0

# List source (if debug info is present)
(gdb) list *0xFF207DA0
45      str    x0, [sp, #-16]!    // Push x0 to stack
```

**Single-stepping:**
```gdb
(gdb) stepi     # Step one instruction (like model.step(1) on FVP)
(gdb) info registers pc
pc             0xff207da4

(gdb) stepi
(gdb) info registers pc
pc             0xff207da8
```

---

## Part V: Breakpoints and Watchpoints — Real vs FVP

### Hardware Breakpoints: A Precious Resource

**On FVP:** Unlimited breakpoints. We could set 100 if we wanted.

**On real hardware:** ARM cores have **4-6 hardware breakpoint registers** (implementation-defined). That's it.

**Checking available breakpoints:**
```
(gdb) info breakpoints
No breakpoints or watchpoints.

# ARM Cortex-A/Neoverse typically has 6 hardware breakpoints
```

**Setting a breakpoint:**
```gdb
(gdb) hbreak *0xFF207DA0
Hardware assisted breakpoint 1 at 0xFF207DA0

(gdb) continue
Continuing.

Breakpoint 1, 0x00000000FF207DA0 in _ModuleEntryPoint ()
```

**When you run out of hardware breakpoints, GDB automatically tries to use software breakpoints** (writes a `BRK` instruction to memory). This works for:
- ✅ RAM code
- ❌ ROM/Flash code (can't write)
- ❌ Instruction caches disabled (breakpoint might not be seen)

### Watchpoints: Data Access Breakpoints

**Scenario:** You want to know when the SP register gets corrupted to `0xFFFFFFFFFFFFFD90`.

```gdb
# Watch writes to a memory address
(gdb) watch *(uint64_t*)0x4034FE40
Hardware watchpoint 2: *(uint64_t*)0x4034FE40

(gdb) continue
Continuing.

Hardware watchpoint 2: *(uint64_t*)0x4034FE40
Old value = 0x0
New value = 0xDEADBEEF
```

**Limitation:** Typical ARM cores have **4 watchpoint registers**. Use them wisely.

### Cross-Trigger Interface (CTI): Halt Others When One Breaks

**Scenario:** CPU0 crashes in StandaloneMM. You want to see what CPU1, CPU2, CPU3 were doing at that exact moment.

**CTI configuration (OpenOCD):**
```tcl
# When CPU0 halts, trigger halt on CPU1-3
cti create cpu0.cti -dap rdn2.dap -ap-num 0 -ctibase 0x80420000
cti create cpu1.cti -dap rdn2.dap -ap-num 0 -ctibase 0x80520000

cpu0.cti enable
cpu1.cti enable

# Configure cross-trigger: CPU0 halt → CPU1 halt
cpu0.cti write CTIINEN0 0x1
cpu1.cti write CTIOUTEN0 0x1
```

Now when CPU0 hits a breakpoint or crashes, CPU1 automatically halts too. You can read both states in perfect sync.

---

## Part VI: Exception and Crash Analysis — Real Debugging Session

### Scenario: Reproducing Our StandaloneMM Crash on Real Hardware

**On FVP, we had:** System crashed with bad stack pointer (`0xFFFFFFFFFFFFFD90`), data abort, we connected Iris and dumped registers.

**On real hardware:** Same crash, but we need to catch it with JTAG/SWD. Here's the complete step-by-step workflow.

---

### Step 1: Initial Connection and Discovery

**Start OpenOCD** (connect to RDN2 board via J-Link probe):

```bash
$ openocd -f interface/jlink.cfg -f board/arm_rdn2.cfg

Open On-Chip Debugger 0.12.0
Licensed under GNU GPL v2
For bug reports, read http://openocd.org/doc/doxygen/bugs.html

Info : J-Link V11 compiled Dec  6 2024 14:23:38
Info : Hardware version: 11.00
Info : VTarget = 3.318 V
Info : clock speed 10000 kHz
Info : SWD DPIDR 0x6ba02477
Info : [rdn2.cpu0] Cortex-A72 r0p3 (target halted)
Info : [rdn2.cpu1] Cortex-A72 r0p3 (target halted)
Info : [rdn2.cpu2] Cortex-A72 r0p3 (target halted)
Info : [rdn2.cpu3] Cortex-A72 r0p3 (target halted)
Info : Listening on port 3333 for gdb connections
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
```

✅ **Connected!** VTarget = 3.3V (board powered), 4 CPUs discovered.

---

### Step 2: Let System Boot to Crash Point

**Monitor UART console** (on /dev/ttyUSB0):

```
SCP Firmware v2.11.0 (Mar 10 2025)
[MCP] CMN-700 initialization... done
[MCP] DDR4 training... done

NOTICE:  BL1: v2.11(debug):v2.11.0-dirty
NOTICE:  BL1: Built : 14:32:21, Mar 10 2025
NOTICE:  BL31: v2.11(debug):v2.11.0-dirty
NOTICE:  BL31: Built : 14:34:15, Mar 10 2025
INFO:    BL31: Initializing runtime services
INFO:    BL31: Initializing BL32 (StandaloneMM)
UEFI firmware (version 1.0 Built at 14:40:00 on Mar 10 2025)

Platform Init...PlatformPeim: SetBootMode: BootMode = 0
SecureBootPolicyInit: Authentication Status: 0

**SYSTEM HANGS HERE** ← No further output, watchdog will trigger in 60s
```

**The crash happened during UEFI variable access** (EFI_VARIABLE_WRITE_ARCH_PROTOCOL triggering an SMC to StandaloneMM).

---

### Step 3: Halt and Capture State

**Telnet to OpenOCD** (within 60 seconds before watchdog resets):

```bash
$ telnet localhost 4444
```

**Halt all CPUs:**

```
> halt
[rdn2.cpu0] target halted in ARM64 state due to debug-request, current mode: EL3
[rdn2.cpu1] target halted in ARM64 state due to debug-request, current mode: EL1N
[rdn2.cpu2] target halted in ARM64 state due to debug-request, current mode: EL1N
[rdn2.cpu3] target halted in ARM64 state due to debug-request, current mode: EL1N
```

✅ **All CPUs halted.** CPU0 is in **EL3** (secure monitor) — the crash happened during SMC handling!

---

### Step 4: Read Exception State (CPU0)

**Check program counter:**

```
> rdn2.cpu0 reg pc
pc (/64): 0x00000000FF207DA0
```

**Read exception syndrome register (EL3):**

```
> rdn2.cpu0 reg ESR_EL3
esr_el3 (/64): 0x000000005E000000

# Decode: EC (bits [31:26]) = 0x17 (SMC instruction trapped to EL3)
#         ISS = 0 (immediate value from SMC #0)
```

**Read EL3 return address:**

```
> rdn2.cpu0 reg ELR_EL3
elr_el3 (/64): 0x00000000FF207DA4

# This is where we'll return to after SMC completes
# (the instruction AFTER the SMC in StandaloneMM)
```

**Read EL1 exception state** (where the actual fault happened):

```
> rdn2.cpu0 reg ESR_EL1
esr_el1 (/64): 0x0000000092000044

# Decode: EC = 0x24 (Data Abort from lower EL)
#         ISV = 0 (syndrome not valid)
#         DFSC = 0x04 (Translation fault, level 0)
```

**Read faulting address:**

```
> rdn2.cpu0 reg FAR_EL1
far_el1 (/64): 0xFFFFFFFFFFFFFD90

# ← BAD STACK POINTER! This is not a valid address.
```

**Read stack pointer:**

```
> rdn2.cpu0 reg SP
sp (/64): 0xFFFFFFFFFFFFFD90

# Confirmed: SP is corrupted. Translation fault because:
# • 0xFFFFFFFF_FFFFFFD90 is a canonical address (upper half)
# • But level 0 translation table has no entry for this region
# • Likely: stack pointer was decremented beyond valid stack
```

---

### Step 5: Inspect Call Stack

**Read link register (return address):**

```
> rdn2.cpu0 reg X30
x30 (/64): 0x00000000FF018400

# X30 (LR) = return address = 0xFF018400
# This is in BL31 runtime_exceptions handler
```

**Read frame pointer:**

```
> rdn2.cpu0 reg X29
x29 (/64): 0x00000000FF207D80

# X29 (FP) = 0xFF207D80 (StandaloneMM stack, looks valid)
# But SP = 0xFFFFFFFFFFFFFD90 is NOT near FP!
```

**Manual stack backtrace:**

```
> mdw 0xFF207D80 8
0xFF207D80: 0xFF207DC0  0xFF018400  0x00000000  0x5E000000
            ▲           ▲
            │           └─ Previous LR (runtime_exceptions @ BL31)
            └─ Previous FP

# Stack frame shows:
# [FP+0]  = Previous FP = 0xFF207DC0
# [FP+8]  = Previous LR = 0xFF018400 (BL31)
# [FP+16] = Local var = 0x0
# [FP+24] = Local var = 0x5E000000 (looks like ESR_EL3!)
```

**Reconstructed call stack:**

```
Frame 0: PC=0xFF207DA0  (_ModuleEntryPoint+160 in StandaloneMM)
         SP=0xFFFFFFFFFFFFFD90 ← CORRUPTED
         FP=0xFF207D80 (valid)
         LR=0xFF018400 (BL31 runtime_exceptions)

Frame 1: PC=0xFF018400  (runtime_exceptions in BL31)
         FP=0xFF207DC0
         LR=0xFF018234 (bl31_main)

Frame 2: PC=0xFF018234  (bl31_main in BL31)
```

---

### Step 6: GDB Symbol Resolution

**Connect GDB to OpenOCD:**

```bash
$ gdb-multiarch
(gdb) target extended-remote localhost:3333
Remote debugging using localhost:3333
0x00000000FF207DA0 in ?? ()
```

**Load BL31 symbols:**

```gdb
(gdb) symbol-file ~/rdn2/tf-a/build/rdn2/debug/bl31/bl31.elf
Reading symbols from bl31.elf...
```

**Resolve PC=0xFF207DA0:**

```gdb
(gdb) info symbol 0xFF207DA0
No symbol matches 0xFF207DA0.

# Hmm, PC is NOT in BL31. Let's check where it is:

(gdb) info symbol 0xFF018400
runtime_exceptions + 64 in section .text of bl31.elf

# LR points to BL31, but PC doesn't. PC must be in StandaloneMM.
```

**Load StandaloneMM symbols at runtime address:**

```gdb
(gdb) add-symbol-file ~/rdn2/edk2/Build/SgiMmStandalone/DEBUG_GCC5/AARCH64/StandaloneMmCore.dll 0xFF200000
add symbol table from file "StandaloneMmCore.dll" at
        .text_addr = 0xFF200000
(y or n) y
Reading symbols from StandaloneMmCore.dll...

# Now resolve again:
(gdb) info symbol 0xFF207DA0
_ModuleEntryPoint + 160 in section .text

(gdb) list *0xFF207DA0
42    ModuleEntryPoint (IN VOID *HobStart)
43    {
44       // ...
45       MmFoundationEntryPoint (&HobStart);
46  >>>  return EFI_SUCCESS;    ← Faulted here!
47    }
```

**Disassemble around fault:**

```gdb
(gdb) disassemble 0xFF207D90,+32
Dump of assembler code from 0xff207d90 to 0xff207db0:
   0x00000000ff207d90 <_ModuleEntryPoint+144>: ldr  x8, [x19, #8]
   0x00000000ff207d94 <_ModuleEntryPoint+148>: cbz  w8, 0xff207da8
   0x00000000ff207d98 <_ModuleEntryPoint+152>: mov  w0, #0x2
   0x00000000ff207d9c <_ModuleEntryPoint+156>: blr  x8
=> 0x00000000ff207da0 <_ModuleEntryPoint+160>: str  x0, [sp, #-16]!   ← FAULT
   0x00000000ff207da4 <_ModuleEntryPoint+164>: mov  w0, #0x0
   0x00000000ff207da8 <_ModuleEntryPoint+168>: bl   0xff207eb0
   0x00000000ff207dac <_ModuleEntryPoint+172>: ldr  x0, [sp], #16
End of assembler dump.
```

**Instruction causing fault:**

```asm
str  x0, [sp, #-16]!

# This means: Write X0 to [SP-16], then update SP = SP-16
# Translation:  SP was 0xFFFFFFFFFFFFFDA0
#               Instruction tries SP = SP - 16 = 0xFFFFFFFFFFFFFD90
#               Then tries to write to [0xFFFFFFFFFFFFFD90]
#               MMU says: "No translation for this address!" → Data Abort
```

---

### Step 7: Root Cause Analysis

**Check all general-purpose registers:**

```gdb
(gdb) info registers
x0             0x0                  0
x1             0xff200000           4279238656
x2             0xff300000           4280287232
x3             0xff3ffe00           4281335296
x4             0x0                  0
x5             0x1                  1
...
x19            0xff207e00           4279263744
x20            0xff300000           4280287232
x29            0xff207d80           4279263616   ← FP looks OK
x30            0xff018400           4278739968   ← LR = runtime_exceptions
sp             0xffffffffffff\fd90  0xfffffffffffffffd90  ← **CORRUPTED SP**
pc             0xff207da0           0xff207da0
```

**SP should be near FP (0xFF207D80), but it's at 0xFFFFFFFF_FFFFFFD90!**

**Why?**

1. **Hypothesis 1:** Stack overflow — SP was decremented too many times, wrapped around
   - Unlikely: Address is way too far from valid stack region

2. **Hypothesis 2:** SP was explicitly set to a bad value
   - Check: Who sets SP in _ModuleEntryPoint?

**Disassemble _ModuleEntryPoint entry:**

```gdb
(gdb) disassemble _ModuleEntryPoint
Dump of assembler code for function _ModuleEntryPoint:
   0x00000000ff207d10 <+0>:  stp   x29, x30, [sp, #-16]!
   0x00000000ff207d14 <+4>:  mov   x29, sp
   0x00000000ff207d18 <+8>:  sub   sp, sp, #0x20
   ...
   0x00000000ff207d30 <+32>: bl    0xff208000 <MmFoundationEntryPoint>
   ...
   0x00000000ff207da0 <+160>: str   x0, [sp, #-16]!   ← Fault here
```

**Wait! After `bl MmFoundationEntryPoint`, the function tries to push x0 to stack. But did `MmFoundationEntryPoint` corrupt SP?**

**Check MmFoundationEntryPoint code:**

```gdb
(gdb) disassemble MmFoundationEntryPoint
   ...
   0x00000000ff2080f0:  mov   sp, x8      ← **HERE! SP is loaded from X8**
   0x00000000ff2080f4:  bl    0xff208200
   0x00000000ff2080f8:  ret
```

**Root cause found:**

1. `MmFoundationEntryPoint` **overwrites SP** with a value from X8
2. X8 contains **0xFFFFFFFF_FFFFFFD90** (invalid stack pointer)
3. When `_ModuleEntryPoint` resumes, SP is corrupted
4. Next instruction (`str x0, [sp, #-16]!`) tries to write to bad address → **Data Abort**

**Fix:** `MmFoundationEntryPoint` should set up a VALID stack pointer, not 0xFFFFFFFF_FFFFFFD90.

---

### The Registers Are Identical (FVP vs Real HW)

Whether on FVP or real hardware, the exception registers are the same:

| Register | FVP (Iris API) | Real HW (OpenOCD/GDB) | Value |
|----------|---------------|---------------------|-------|
| **ESR_EL3** | `cpu0.read_register('ESR_EL3')` | `reg ESR_EL3` | `0x5E000000` (SMC) |
| **ELR_EL3** | `cpu0.read_register('ELR_EL3')` | `reg ELR_EL3` | `0xFF207DA4` |
| **ESR_EL1** | `cpu0.read_register('ESR_EL1')` | `reg ESR_EL1` | `0x92000044` (Data Abort, L0 fault) |
| **FAR_EL1** | `cpu0.read_register('FAR_EL1')` | `reg FAR_EL1` | `0xFFFFFFFFFFFFFD90` (bad stack) |
| **PC** | `cpu0.read_register('PC')` | `reg pc` | `0xFF207DA0` (_ModuleEntryPoint+160) |
| **SP** | `cpu0.read_register('SP')` | `reg sp` | `0xFFFFFFFFFFFFFD90` |

**The debugging approach is identical — only the tool syntax differs.**

---
> halt
target halted in ARM64 state, current mode: EL3

> reg ESR_EL3
esr_el3 (/64): 0x000000005E000000

> reg ELR_EL3  
elr_el3 (/64): 0x00000000FF207DA4

> reg ESR_EL1
esr_el1 (/64): 0x0000000092000044

> reg FAR_EL1
far_el1 (/64): 0xFFFFFFFFFFFFFD90
```

**Decoding is identical:**
```python
ec = (0x92000044 >> 26) & 0x3F  # 0x24 = Data Abort
dfsc = 0x92000044 & 0x3F         # 0x04 = Translation fault, level 0
```

### Post-Mortem Analysis: Preserving Crash State

**FVP:** System hangs, we connect Iris, read everything.

**Real hardware:** If the system already reset (watchdog, panic handler, etc.), you've lost the state UNLESS:

1. **Halt-on-reset configured:**
   ```tcl
   # OpenOCD config
   rdn2.cpu0 configure -event reset-assert-post {halt}
   ```

2. **Warm reset preserves debug state** (board-dependent):
   - Some platforms preserve registers across nRESET

3. **CoreSight trace was running** (see Part VII — this is the killer feature!):
   - Trace buffer contains the entire execution history leading to the crash

**Example: Catching a panic before watchdog reset:**
```gdb
# Set breakpoint on panic handler
(gdb) hbreak plat_panic_handler
Hardware assisted breakpoint 1 at 0xFF01843C

(gdb) continue

# System crashes, hits panic
Breakpoint 1, 0xFF01843C in plat_panic_handler ()

# Now you have full state before watchdog fires
(gdb) bt
#0  plat_panic_handler () at plat/common/aarch64/plat_common.c:128
#1  0xFF018400 in runtime_exceptions ()
#2  0xFF207DA0 in _ModuleEntryPoint ()
```

---

## Part VII: CoreSight Trace — The Real Hardware Superpower

### What FVP Cannot Give You

On FVP, we can **single-step** through code or set **breakpoints**. But we can't:
- Record the execution history before a crash
- Trace every instruction executed across milliseconds
- Capture branch targets, function calls, returns automatically
- See the path the CPU took to reach a bad state

**CoreSight ETM (Embedded Trace Macrocell) on real hardware gives us all of this.**

### CoreSight Trace Architecture — Detailed Flow

**ETM trace flow from CPU to decode:**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        ETM Trace Data Flow                                   │
│                                                                              │
│  Step 1: CPU Execution → ETM Capture                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  CPU Core (Neoverse V1)                                              │   │
│  │  ┌────────────────────────────────────────────────────────────────┐  │   │
│  │  │  Instruction Pipeline                                          │  │   │
│  │  │  • Fetch: 0xFF018400 (mrs x0, esr_el3)                         │  │   │
│  │  │  • Execute: Read ESR_EL3 = 0x5E000000                          │  │   │
│  │  │  • Next:  0xFF018404                                           │  │   │
│  │  └────────────────────────────┬───────────────────────────────────┘  │   │
│  │                                │ Instruction commit signal            │   │
│  │  ┌─────────────────────────────▼──────────────────────────────────┐  │   │
│  │  │  ETM (Embedded Trace Macrocell) @ 0x840000                     │  │   │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │  │   │
│  │  │  │  Trace Generation Logic                                  │  │  │   │
│  │  │  │  • Address: 0xFF018400                                   │  │  │   │
│  │  │  │  • Opcode:  mrs (read system register)                   │  │  │   │
│  │  │  │  • Branch:  No                                           │  │  │   │
│  │  │  │  • Exception: No                                         │  │  │   │
│  │  │  │  → Generate: Atom packet (N = not taken branch)          │  │  │   │
│  │  │  │             + Address packet (when branch taken)         │  │  │   │
│  │  │  └──────────────────────────────────────────────────────────┘  │  │   │
│  │  │                                                                 │  │   │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │  │   │
│  │  │  │  Trace Compression (ETM v4.0+)                           │  │  │   │
│  │  │  │  • Most instructions: Atom packets (1 bit each!)         │  │  │   │
│  │  │  │  • Branch target:     Address packet (4-8 bytes)         │  │  │   │
│  │  │  │  • Exception:         Exception packet (8 bytes)         │  │  │   │
│  │  │  │  Compression ratio: ~100:1 typical                       │  │  │   │
│  │  │  └──────────────────────────────────────────────────────────┘  │  │   │
│  │  └─────────────────────────────┬───────────────────────────────────┘  │   │
│  └────────────────────────────────┼──────────────────────────────────────┘   │
│                                   │ Compressed trace packets                 │
│  Step 2: Funnel → Merge Multiple Cores                                      │
│                   ┌───────────────▼───────────────┐                          │
│                   │    Trace Funnel @ 0x8C0000    │                          │
│                   │  Combines 4 ETM streams:      │                          │
│                   │  CPU0: [N][N][E][A:0xFF01...] │                          │
│                   │  CPU1: [N][N][N][N][N]        │                          │
│                   │  CPU2: [N][E][A:0x800800...]  │                          │
│                   │  CPU3: [idle]                 │                          │
│                   │  ↓ Time-multiplexed output    │                          │
│                   │  [ID:0][N][N][ID:1][N][ID:0]  │                          │
│                   │  [E][A:0xFF01...][ID:2]...    │                          │
│                   └───────────────┬───────────────┘                          │
│                                   │ Merged packets with source IDs           │
│  Step 3: ETF or ETR → Buffer                                                │
│         ┌─────────────────────────┴─────────────────────┐                   │
│         │   ETF (On-chip FIFO) @ 0x8D0000               │                   │
│         │   ┌──────────────────────────────────────┐    │                   │
│         │   │  Circular Buffer (32KB)              │    │                   │
│         │   │  [Packet][Packet][Packet]...[Packet] │    │                   │
│         │   │   ▲                          │       │    │                   │
│         │   │   └──────Wraps around────────┘       │    │                   │
│         │   │  Captures: Last ~10,000 instructions │    │                   │
│         │   └──────────────────────────────────────┘    │                   │
│         └────────────────────┬──────────────────────────┘                   │
│                              │ OR (if configured)                            │
│         ┌────────────────────▼──────────────────────────┐                   │
│         │   ETR (Trace Router) @ 0x8E0000               │                   │
│         │   Writes to System DRAM via AXI:              │                   │
│         │   ┌──────────────────────────────────────┐    │                   │
│         │   │  DRAM Buffer @ 0x80_0000_0000        │    │                   │
│         │   │  Size: 16MB (configurable 1-256MB)   │    │                   │
│         │   │  [Packet stream ─────────────────►]  │    │                   │
│         │   │  Captures: Millions of instructions  │    │                   │
│         │   └──────────────────────────────────────┘    │                   │
│         └───────────────────────────────────────────────┘                   │
│                                                                              │
│  Step 4: Extract and Decode                                                 │
│         ┌──────────────────────────────────────────────┐                    │
│         │  Host PC (via OpenOCD / Arm DS)             │                    │
│         │  1. Halt system, read ETR write pointer     │                    │
│         │  2. Extract: dump_image trace.bin ...       │                    │
│         │  3. Decode with:                             │                    │
│         │     • ARM CoreSight Access Library           │                    │
│         │     • perf (Linux kernel tool)               │                    │
│         │     • Arm DS GUI trace viewer                │                    │
│         │  4. Output: Full instruction flow with PCs   │                    │
│         └──────────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

**ETM Packet Format (compressed trace):**

```
┌────────────────────────────────────────────────────────────────────────┐
│  ETM Trace Packet Types (ETMv4.0/ETMv4.6)                             │
│                                                                        │
│  1. Atom Packets (most common, 1 bit per non-branch instruction)      │
│     Binary: N N N N N N E N N N...                                    │
│              │ │ │ │ │ │ │ │ │ └─ Not taken (sequential)             │
│              │ │ │ │ │ │ │ │ └─── Not taken                          │
│              │ │ │ │ │ │ │ └───── Exception occurred!                │
│              └─└─└─└─└─└────────── More sequential instructions       │
│     Meaning: CPU executed 6 straight-line instructions, then          │
│              exception, then 3 more instructions.                     │
│                                                                        │
│  2. Address Packets (when branch target cannot be inferred)           │
│     Hex: 9D 40 18 00 FF 00 00 00                                      │
│          ▲  └──────┬──────────┘                                       │
│          │         └─ Address: 0xFF018400 (compressed, 4-8 bytes)     │
│          └─ Packet header (0x9D = long address)                       │
│                                                                        │
│  3. Exception Packets (when exception happens)                        │
│     Hex: 06 17 00 00 00                                               │
│          ▲  └───┬────┘                                                │
│          │      └─ Exception number: 0x17 (SMC to EL3)                │
│          └─ Packet header (0x06 = exception packet)                   │
│                                                                        │
│  4. Context ID Packets (track process/thread switches)                │
│     Hex: 6E 34 12 00 00                                               │
│          ▲  └───┬────┘                                                │
│          │      └─ Context ID: 0x1234 (process ID)                    │
│          └─ Packet header (0x6E = context ID)                         │
│                                                                        │
│  5. Timestamp Packets (correlate with external events)                │
│     Hex: 02 A0 F3 45 01                                               │
│          ▲  └────┬─────┘                                              │
│          │       └─ Timestamp: 21,459,872 cycles                      │
│          └─ Packet header (0x02 = timestamp)                          │
└────────────────────────────────────────────────────────────────────────┘
```

**Compression efficiency example:**

```
Raw instruction trace (without ETM):
    100,000 instructions × 4 bytes/instruction = 400 KB

ETM compressed trace:
    • 95,000 sequential instructions → Atom packets: 95,000 bits = 11.8 KB
    • 4,500 branches → Address packets: 4,500 × 5 bytes = 22.5 KB
    • 500 exceptions → Exception packets: 500 × 5 bytes = 2.5 KB
    Total: ~37 KB  (10:1 compression ratio!)
```

---

### Three Trace Capture Modes

1. **ETR to DRAM:** Trace writes to a large system memory buffer
2. **ETF on-chip:** Circular buffer, last 8KB-32KB of execution
3. **Streaming to probe:** Real-time trace to DSTREAM-PT or other probe (expensive!)

### Enabling ETM Trace (OpenOCD Example)

**Basic ETM configuration for CPU0:**

```tcl
# Enable ETM on CPU0
cti create cpu0.etm -dap rdn2.dap -ap-num 0 -baseaddr 0x80440000

# Configure trace to ETR (DRAM buffer at 0x8000000000, 16MB)
etm config cpu0.etm -trigger-percent 10 -buffer-size 0x1000000 -base 0x8000000000

# Start tracing
etm start cpu0.etm
```

**ETR configuration (writes to system RAM):**

```
> mdw 0x80460000 0x10    ← Read ETR Control register
0x80460000: 0x00000001    ← Enabled

> mdw 0x80460004 0x10    ← Read ETR buffer base
0x80460004: 0x80000000    ← Trace buffer at 0x8000000000

> mdw 0x80460008 0x10    ← Read ETR write pointer
0x80460008: 0x800F3A80    ← ~1MB of trace captured
```

### Decoding Trace: From Raw Packets to Execution Flow

**Stop trace and extract:**

```bash
# Stop ETM
> etm stop cpu0.etm

# Dump trace buffer to file
> dump_image trace.bin 0x8000000000 0x1000000

# Decode with Arm CoreSight Access Library (or perf on Linux)
$ perf script --itrace=i1i -F +addr,+flags --show-task-events < trace.bin > trace_decoded.txt
```

**What you get:**
```
[CPU0] 0xFF018000: bl     0xFF018400    ← Called into runtime_exceptions
[CPU0] 0xFF018404: mrs    x0, esr_el3   ← Reading exception syndrome
[CPU0] 0xFF018408: tbz    w0, #17       ← Testing EC field
[CPU0] 0xFF01840C: b      0xFF018420    ← Branch not taken
[CPU0] 0xFF018420: bl     0xFF207D00    ← Call into StandaloneMM
[CPU0] 0xFF207D00: stp    x29, x30, [sp, #-16]!  ← MM entry
...
[CPU0] 0xFF207DA0: str    x0, [sp, #-16]!  ← **FAULT HERE**
[Exception] Data Abort, FAR=0xFFFFFFFFFFFFFD90
```

**You now see the ENTIRE path the CPU took to reach the fault**, not just the final PC.

### Use Case: Silent Boot Hang

Remember our silent kernel boot (no UART output)? On FVP, we bisected by adding debug prints.

**With ETM on real hardware:**

1. Configure trace to capture boot
2. System hangs
3. Halt CPU, dump trace buffer
4. Decode trace to see the last 10,000 instructions before hang

**Example output:**
```
[CPU0] 0x0000000080010000: bl    start_kernel      ← Kernel entry
[CPU0] 0x0000000080012340: bl    setup_arch
[CPU0] 0x0000000080012400: bl    setup_cpufeatures
[CPU0] 0x00000000800124A0: mrs   x0, id_aa64mmfr0_el1
[CPU0] 0x00000000800124A4: tst   x0, #0xF0000      ← Testing for SVE
[CPU0] 0x00000000800124A8: b.eq  0x80012500        ← Branch to hang point
[CPU0] 0x0000000080012500: wfi                     ← **HUNG HERE**
```

Now you know it's an SVE detection issue without adding a single print statement!

### Trace Filtering: Focus on Secure World Only

**Problem:** Tracing everything creates huge data. We only care about TF-A and StandaloneMM.

**Solution:** Address range filtering in ETM:

```tcl
# Trace only addresses 0xFE000000-0xFFFFFFFF (secure firmware)
etm config cpu0.etm -address-range 0xFE000000 0xFFFFFFFF

# Or trace only EL3 (using context ID filtering)
etm config cpu0.etm -context-id 3   ← EL3 only
```

Now the trace buffer only captures secure world execution, lasting much longer before wrapping.

---

## Part VIII: UART and Console — Same Logs, Physical Wires

### Serial Console on FVP vs Real Hardware

**On FVP:** UART output goes to a telnet port or file:
```bash
$ telnet localhost 5000
SCP Firmware v2.11
AP Firmware: BL31 Starting...
```

**On real hardware:** UART is a physical connector (usually USB-to-serial adapter).

**Physical connection:**
```
RD-N2 Dev Board             USB-to-Serial Adapter
┌────────────┐              ┌──────────┐
│  J10       │              │          │
│  Pin 1: GND├─────────────►│ GND      │
│  Pin 2: TX ├─────────────►│ RX       │
│  Pin 3: RX ├─────────────►│ TX       │
└────────────┘              │ (FTDI)   ├──► USB to PC
                            └──────────┘
```

**Connecting:**
```bash
$ ls /dev/ttyUSB*
/dev/ttyUSB0

$ screen /dev/ttyUSB0 115200
# Or
$ minicom -D /dev/ttyUSB0 -b 115200
```

### UART Debugging: Same Techniques

The UART output is identical:
- SCP firmware prints to UART0 (MCP console)
- TF-A BL31 prints to UART1 (AP console)
- Linux kernel prints to same UART

**Silent UART troubleshooting checklist:**

| Check | FVP | Real HW |
|-------|-----|---------|
| UART enabled in firmware? | ✅ | ✅ Same DT/config |
| Baud rate mismatch? | N/A | ⚠️ Check 115200 vs 9600 |
| Wrong UART pinmux? | N/A | ⚠️ Verify board schematic |
| UART clock disabled? | N/A | ⚠️ Check SCP config |
| RX/TX swapped? | N/A | ⚠️ Try swapping wires |

**Debugging via MMIO registers (same on both):**

```gdb
# Check if UART TX FIFO is working
(gdb) x/1x 0x2A400000          ← PL011 UART base address
0x2A400000: 0x00000011          ← UARTFR: TX FIFO not full

# Write directly to UART data register
(gdb) set *(uint32_t*)0x2A400000 = 0x41  ← Write 'A'
```

If you see 'A' on the console, the UART hardware is working — the problem is in software.

---

## Part IX: Power Domain and Boot Debugging — Before main()

### The Boot Sequence Real Hardware Must Navigate

**On FVP:** Boot is instant. You connect to a running system.

**On real hardware:** The system must:
1. **Power on** (PSU, voltage regulators stabilize)
2. **ROM bootloader** runs (burned into silicon)
3. **SCP firmware** starts (loads from flash)
4. **TF-A BL1** (secure boot loader)
5. **TF-A BL2** (firmware loader)
6. **TF-A BL31** (runtime secure monitor)
7. **UEFI** (if using Tianocore)
8. **OS loader**

### Debugging ROM Code

**Challenge:** ROM code is in read-only memory at 0x00000000 (or wherever the reset vector is). You can't modify it.

**Approach:**

1. **Connect immediately after power-on:**
   ```tcl
   # OpenOCD: halt on reset
   reset_config srst_only srst_nogate
   rdn2.cpu0 configure -event reset-start {halt}
   ```

2. **Read the reset vector:**
   ```
   > mdw 0x00000000 0x10
   0x00000000: 0x58000040    ← LDR X0, #PC+8 (loads reset handler address)
   0x00000004: 0xD61F0000    ← BR X0 (jumps to it)
   0x00000008: 0x0A000000    ← Reset handler at 0x0A000000 (ROM space)
   ```

3. **Set breakpoint in ROM:**
   ```gdb
   (gdb) hbreak *0x0A000100
   (gdb) continue
   ```

4. **Step through ROM boot:**
   ```gdb
   (gdb) stepi
   (gdb) info registers
   ```

This lets you see what the ROM does before loading SCP firmware (e.g., authentication checks, fuse reads, UART init).

### Debugging SCP Firmware Startup

**Scenario:** SCP firmware doesn't start, no UART output.

**Debug path:**

1. **Check if SCP is running:**
   ```
   > halt   ← Halt the system
   
   # Read SCP CPU PC (SCP usually runs on Cortex-M7 at different AP)
   > dap apreg 1 0x84        ← Read SCP debug registers (AP 1)
   > mdw 0x44000000 0x10     ← SCP RAM (check if code loaded)
   ```

2. **Breakpoint on SCP entry point:**
   ```
   # SCP firmware entry at 0x0BD06000 (from linker map)
   > bp 0x0BD06000 4 hw
   > resume
   
   # Did we hit it?
   > halt
   > reg pc
   pc: 0x0BD06010   ← Yes! SCP started but maybe hung later
   ```

3. **SCP firmware loaded from flash** — check flash controller:
   ```
   > mdw 0x08000000 0x10    ← NOR flash base
   0x08000000: 0x464C457F   ← ELF magic (SCP image is there)
   
   > mdw 0x70000000 0x10    ← Flash controller registers
   0x70000000: 0x00000001   ← Controller enabled
   ```

### Power Domain Debugging

**Scenario:** System boots, but UEFI can't initialize PCIe because the power domain is off.

**Check power state (via SCP or SCMI):**

```gdb
# Read CMN-700 Node Info registers (interconnect power state)
(gdb) x/1x 0x50000000
0x50000000: 0x00008001    ← PERIPHBASE[0] powered

# Check PCIe root complex power
(gdb) x/1x 0x60000000
0x60000000: 0x00000000    ← Not powered! ← Root cause
```

**Fix:** Enable power domain in SCP firmware or device tree.

---

## Part X: FVP vs Real Hardware — Side-by-Side Comparison

| Feature | FVP (Model) | Real Hardware (JTAG/SWD) |
|---------|-------------|--------------------------|
| **Connection** | Python Iris API, CADI | OpenOCD, Arm DS, JTAG/SWD probe |
| **Boot time** | Instant | 10-60 seconds (power-on, ROM, firmware) |
| **Breakpoints** | Unlimited software | 4-6 hardware breakpoints |
| **Watchpoints** | Unlimited | 4 hardware watchpoints |
| **Execution control** | `model.run()`, `model.step(N)` | `continue`, `stepi` (GDB) |
| **Register access** | `cpu0.read_register('ESR_EL3')` | `reg ESR_EL3` (OpenOCD), `info reg` (GDB) |
| **Memory access** | `cpu0.read(0xFFFF, 64)` | `mdw 0xFFFF` (OpenOCD), `x/16x` (GDB) |
| **Symbol loading** | `addr2line`, manual scripts | `symbol-file`, `add-symbol-file` (GDB) |
| **Source debugging** | Manual objdump disassembly | `list`, `disassemble`, `bt` (GDB) |
| **Trace execution** | ❌ No historical trace | ✅ **ETM captures everything** |
| **Multi-core halt** | Manual per-core halt | ✅ CTI cross-trigger (halt all on break) |
| **UART** | Telnet port, stdout | Physical serial (USB-to-UART adapter) |
| **Power sequencing** | N/A (instant boot) | Must handle reset, ROM, power domains |
| **Secure debug** | Always enabled | Requires SPIDEN/NIDEN fuses or DIP switch |
| **Cost** | Free (FVP license) | $500 (J-Link) to $10,000 (Arm DSTREAM) |

---

## Part XI: When Is Real Hardware Harder? When Is It Easier?

### Real Hardware Is **Harder** When:

1. **Limited breakpoints:** You have 6 hardware breakpoints. Choose wisely.
2. **Boot sequencing:** You must understand power-on, ROM, SCP, TF-A flow to catch early issues.
3. **Physical access needed:** Can't debug remotely without a network-enabled probe.
4. **Secure debug locked:** If SPIDEN=0 and you don't have the key, you're stuck.
5. **Timing-sensitive bugs:** Real hardware has real timing. Race conditions appear that FVP never showed.
6. **UART wiring:** Wrong pinout = no console output. FVP doesn't have this problem.

### Real Hardware Is **Easier** When:

1. **Historical trace (ETM):** You can see the ENTIRE execution path leading to a crash, not just the final state.
   - FVP: "The system crashed. Where did it come from?" → Manual bisection with breakpoints.
   - Real HW: "Dump trace. Here's the last 10,000 instructions." → Instant root cause.

2. **Real performance:** Timing-dependent bugs (race conditions, cache coherency issues) only reproduce on real hardware.
   - FVP might not show cache issues that real silicon exposes.

3. **Real peripherals:** FVP models are simplified. Real hardware shows actual hardware quirks:
   - Example: FVP's GIC model might not show a real GICv3 interrupt routing bug.

4. **Production validation:** You MUST test on real hardware eventually. Debugging there first saves later surprises.

---

## Conclusion: The Complementary Debugging Flow

**Use FVP (with Iris/GDB) when:**
- Developing initial firmware ports
- Testing algorithm logic
- Rapid iteration (no reflashing needed)
- Teaching and learning ARM architecture
- Unlimited breakpoints are useful

**Use real hardware (with JTAG/OpenOCD) when:**
- Validating final integration
- Debugging timing-sensitive issues
- Investigating hardware-specific quirks
- Using ETM trace for complex crash analysis
- Testing actual boot flow (ROM, SCP, power)

**The best debugging approach combines both:**

1. **Develop on FVP:** Fast iteration, unlimited breakpoints, easy symbol loading.
2. **Validate on real HW:** Use ETM trace to catch issues FVP didn't show.
3. **Reproduce on FVP:** If a bug appears on real HW, try to reproduce on FVP for faster debugging.

---

## Appendix A: Quick Reference Commands

### OpenOCD Commands (Real Hardware)

```tcl
# Connection
openocd -f board/arm_rdn2.cfg

# Halt and inspect
halt
reg pc
reg x0
mdw 0xFF207DA0 0x10          # Read memory (32-bit words)
mdb 0xFF207DA0 0x40          # Read memory (bytes)

# Breakpoints
bp 0xFF207DA0 4 hw           # Hardware breakpoint (4-byte instruction)
rbp 0xFF207DA0               # Remove breakpoint
rbp all                       # Remove all breakpoints

# Stepping
step                          # Step one instruction
step 10                       # Step 10 instructions
resume                        # Continue execution

# GDB connection
target extended-remote :3333
```

### GDB Commands (with OpenOCD Backend)

```gdb
# Connection
target extended-remote localhost:3333

# Symbol loading
symbol-file bl31.elf
add-symbol-file StandaloneMmCore.dll 0xFF200000

# Breakpoints
hbreak *0xFF207DA0           # Hardware breakpoint at address
hbreak plat_panic_handler    # Hardware breakpoint at symbol

# Inspection
info registers
info registers ESR_EL3
x/16x 0xFF207DA0             # Examine 16 hex words
x/16i 0xFF207DA0             # Disassemble 16 instructions
disassemble 0xFF207DA0,+64
bt                            # Backtrace

# Stepping
stepi                         # Step one instruction
stepi 10                      # Step 10 instructions
continue                      # Resume execution

# Source debugging
list *0xFF207DA0              # Show source at address
info symbol 0xFF207DA0        # Lookup symbol at address
```

### Arm Development Studio Commands

```
# Connect to target
connect RDN2_Board

# Halt and inspect
stop
print $PC
print $X0
memory read 0xFF207DA0 0x40

# Breakpoints
break *0xFF207DA0
break plat_panic_handler
delete                        # Remove all breakpoints

# Trace
trace config --etm cpu0 --buffer-size 16MB
trace start
trace stop
trace decode --output trace.txt
```

---

## Appendix B: Related Blog Posts

- **[ARM64 Debugging Handbook: Iris + GDB + FVP](/posts/arm64-debugging-handbook-iris-gdb-fvp/)** — The complete guide to debugging on ARM FVP using Iris Python API
- **[Debugging Kernel Hang: ARM FVP SVE Trap](/posts/debugging-kernel-hang-arm-fvp-sve-trap/)** — How we traced a silent kernel boot hang using FVP and GDB
- **[Enabling Persistent UEFI Variables: ARM FVP + StandaloneMM](/posts/enabling-persistent-uefi-variables-arm-fvp-standalonemm/)** — The StandaloneMM crash we debugged and fixed
- **[ARM RDN2 FVP Firmware Bring-Up Story](/posts/arm-rdn2-fvp-firmware-bringup-story/)** — SCP PCID incompatibility and other boot adventures

---

**Published:** May 26, 2025  
**Tags:** ARM64, Debugging, JTAG, CoreSight, OpenOCD, Arm DS, GDB, TF-A, StandaloneMM, Real Hardware
