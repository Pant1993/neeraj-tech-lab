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

## Part VI: Exception and Crash Analysis — Same Approach, Different Path

### The Registers Are Identical

Whether on FVP or real hardware, when an exception happens:
- **ESR_ELx** captures the exception syndrome
- **ELR_ELx** captures the return address
- **FAR_ELx** captures the faulting address (for aborts)
- **SPSR_ELx** captures the saved processor state

**Our StandaloneMM crash on FVP:**
```python
esr3 = cpu0.read_register('ESR_EL3')  # 0x5E000000 (EC=0x17, SMC)
elr3 = cpu0.read_register('ELR_EL3')  # 0xFF207DA4 (return from SMC)
esr1 = cpu0.read_register('ESR_EL1')  # 0x92000044 (Data Abort, L0 translation fault)
far1 = cpu0.read_register('FAR_EL1')  # 0xFFFFFFFFFFFFFD90 (bad stack address)
```

**Same crash on real hardware (OpenOCD):**
```
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

### CoreSight Trace Architecture

```
  CPU Core
    │
    ├── ETM (Embedded Trace Macrocell)
    │     │ Captures: PC samples, branches, exceptions, context switches
    │     │ Outputs: Compressed trace packets
    │     ↓
    ├── Funnel (combines traces from multiple cores)
    │     ↓
    ├── ETF (Embedded Trace FIFO) [Optional on-chip buffer: 8KB-32KB]
    │     ↓
    ├── ETR (Embedded Trace Router) [Writes to DRAM buffer: 1MB-256MB]
    │     ↓
    └── TPIU (Trace Port Interface Unit) [Outputs to external trace probe]
```

**Three trace capture modes:**

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
