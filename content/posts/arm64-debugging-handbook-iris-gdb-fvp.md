---
title: "The ARM64 Debugger's Handbook — Iris, GDB, and Live Debugging on the RD-N2 FVP"
date: 2026-05-26T18:50:00+05:30
draft: false
tags: ["ARM", "FVP", "Iris", "GDB", "debugging", "ARM64", "AArch64", "exception-levels", "SMC", "HVC", "SVC", "FF-A", "TF-A", "UEFI", "StandaloneMM", "SCP", "registers", "call-stack", "symbols"]
categories: ["Debugging Guides"]
description: "A comprehensive guide to live debugging on the ARM RD-N2 FVP — from connecting Iris and GDB to loading symbols for every firmware layer, decoding exception syndromes, tracing through exception levels, and approaching real debugging scenarios with concrete examples from our bring-up journey."
cover:
  image: ""
  alt: "ARM64 debugging with Iris and GDB"
ShowToc: true
TocOpen: true
weight: 0
---

> "Stop guessing. Attach the debugger, read the registers, follow the evidence."

This is the debugging guide we wish we had when we started bringing up the ARM RD-N2 FVP. Every technique here was learned by needing it — in the middle of a hang, a crash, or a component that refused to speak.

---

## Part I: The Debugging Arsenal

### What is Iris?

**Iris** is ARM's debug API built into every Fast Model (FVP). It's a TCP-based protocol that gives you register-level and memory-level access to every component in the simulated system — CPUs, memories, peripherals — without needing JTAG or physical hardware.

Think of it as a software JTAG port that's always connected, always available, and works at full simulation speed.

**Key capabilities:**
- Read/write any CPU register (including system registers at all exception levels)
- Read/write physical memory (subject to TrustZone restrictions)
- Start/stop/step CPU cores
- Set breakpoints
- Query the component hierarchy

### When to Use Iris vs GDB

| Scenario | Tool | Why |
|----------|------|-----|
| Quick register dump after a hang | **Iris Python** | Fastest — 3 lines of code, no setup |
| Source-level debugging with step-through | **GDB** | Need symbol info and source correlation |
| Memory inspection (MMIO, page tables) | **Iris Python** | Direct physical address access |
| Breakpoints + conditional logic | **GDB** | Conditional breakpoints, watchpoints |
| Multi-component debugging (SCP + AP) | **Iris Python** | Can target any CPU in the model |
| Post-mortem crash analysis | **Iris Python** | Read registers of crashed core, no state change |

### The FVP Debug Ports

When the FVP launches with `-I` (Iris), it starts the Iris server:

```
Iris server started listening to port 7100
```

For GDB, each CPU exposes a GDB server on a sequential port starting from 7101:
```
CPU0: port 7101
CPU1: port 7102
...
```

The SCP (Cortex-M7) also has its own GDB port, typically at a higher offset.

---

## Part II: Iris — The Firmware Developer's Swiss Knife

### Connecting to the Model

```python
import sys
sys.path.insert(0, r'C:\Program Files\ARM\FVP_RD_N2\Iris\Python')
from iris.debug.Model import NetworkModel

# Connect to the FVP's Iris server
model = NetworkModel('localhost', 7100)
print(f"Connected: {model.isConnected()}")
```

### Discovering Targets

The FVP is a hierarchy of components. Each CPU, peripheral, and memory controller is a "target":

```python
# List all available targets
targets = model.get_targets()
for t in targets:
    info = model.get_target_info(t)
    print(f"  {info.instName}: {info.typeName}")
```

On the RD-N2, the key targets are:

| Target Path | What It Is |
|-------------|-----------|
| `component.RD_N2.css.cluster0.cpu0` | Application Processor core 0 (Neoverse N2) |
| `component.RD_N2.css.cluster1.cpu0` | AP core 1 |
| `component.RD_N2.css.scp.armcortexm7ct` | System Control Processor (Cortex-M7) |
| `component.RD_N2.css.mcp.armcortexm7ct` | Management Control Processor |

### Getting a CPU Handle

```python
# Get the primary AP core
cpu0 = model.get_target('component.RD_N2.css.cluster0.cpu0')

# Get the SCP
scp = model.get_target('component.RD_N2.css.scp.armcortexm7ct')
```

### Reading Registers

This is where Iris shines for post-mortem analysis. After a hang or crash, just read the state:

```python
# General purpose registers
pc  = cpu0.read_register('PC')
sp  = cpu0.read_register('SP_EL0')
lr  = cpu0.read_register('LR')       # X30
x0  = cpu0.read_register('X0')
x1  = cpu0.read_register('X1')

# Exception registers (the gold mine for crash analysis)
elr_el1 = cpu0.read_register('ELR_EL1')   # Return address from EL0/EL1 exception
elr_el2 = cpu0.read_register('ELR_EL2')   # Return address from EL1/EL2 exception  
elr_el3 = cpu0.read_register('ELR_EL3')   # Return address from EL2/EL3 exception
esr_el1 = cpu0.read_register('ESR_EL1')   # Exception syndrome at EL1
esr_el2 = cpu0.read_register('ESR_EL2')   # Exception syndrome at EL2
esr_el3 = cpu0.read_register('ESR_EL3')   # Exception syndrome at EL3
far_el1 = cpu0.read_register('FAR_EL1')   # Fault address at EL1
far_el2 = cpu0.read_register('FAR_EL2')   # Fault address at EL2

# System registers
sctlr_el1 = cpu0.read_register('SCTLR_EL1')  # System control (MMU, caches)
ttbr0_el1 = cpu0.read_register('TTBR0_EL1')  # User-space page tables
ttbr1_el1 = cpu0.read_register('TTBR1_EL1')  # Kernel page tables
currentel = cpu0.read_register('CurrentEL')   # What EL are we at?
spsr_el3  = cpu0.read_register('SPSR_EL3')   # Saved program state at EL3

# Security state
scr_el3   = cpu0.read_register('SCR_EL3')    # Secure Configuration Register

# SVE/FP control
cptr_el3  = cpu0.read_register('CPTR_EL3')   # Coprocessor trap register
cpacr_el1 = cpu0.read_register('CPACR_EL1')  # Coprocessor access control

print(f"PC:      0x{pc:016X}")
print(f"ELR_EL3: 0x{elr_el3:016X}")
print(f"ESR_EL3: 0x{esr_el3:08X}")
print(f"FAR_EL1: 0x{far_el1:016X}")
print(f"SP_EL0:  0x{sp:016X}")
```

### Reading and Writing Memory

```python
# Read 4 bytes at a physical address (returns list of bytes)
data = cpu0.read_memory(0x2A400000, 4)  # PL011 UART data register

# Read 8 bytes
val = cpu0.read_memory(0xFF200000, 8)    # BL31 region

# Write to memory
cpu0.write_memory(0x2A400000, [0x48, 0x65, 0x6C, 0x6C])  # Write "Hell" to UART

# Read a block of memory (useful for dumps)
block = cpu0.read_memory(0xEDC00000, 256)  # First 256 bytes of kernel
print(' '.join(f'{b:02X}' for b in block[:32]))
```

**IMPORTANT: TrustZone restrictions apply.** If you try to read secure memory (like the StandaloneMM's DRAM region) from a non-secure context, you'll get all zeros. The Iris read uses the CPU's current security context.

### Controlling Execution

```python
# Stop the simulation
model.stop()

# Single-step one instruction
model.step(1)

# Resume execution
model.run()

# Check if simulation is running
is_running = model.is_running()
```

### Practical Example: Post-Mortem Crash Dump

Here's the exact script we used to diagnose the StandaloneMM crash:

```python
import sys
sys.path.insert(0, r'C:\Program Files\ARM\FVP_RD_N2\Iris\Python')
from iris.debug.Model import NetworkModel

model = NetworkModel('localhost', 7100)
cpu0 = model.get_target('component.RD_N2.css.cluster0.cpu0')

# The system hung — let's find out why
print("=== CPU0 State ===")
print(f"PC:       0x{cpu0.read_register('PC'):016X}")
print(f"CurrentEL: {cpu0.read_register('CurrentEL')}")
print()

print("=== Exception State (EL3) ===")
print(f"ELR_EL3:  0x{cpu0.read_register('ELR_EL3'):016X}")
print(f"ESR_EL3:  0x{cpu0.read_register('ESR_EL3'):08X}")
print(f"SPSR_EL3: 0x{cpu0.read_register('SPSR_EL3'):08X}")
print()

print("=== Exception State (EL1 — SP's fault) ===")
print(f"ELR_EL1:  0x{cpu0.read_register('ELR_EL1'):016X}")
print(f"ESR_EL1:  0x{cpu0.read_register('ESR_EL1'):08X}")
print(f"FAR_EL1:  0x{cpu0.read_register('FAR_EL1'):016X}")
print(f"SP_EL0:   0x{cpu0.read_register('SP_EL0'):016X}")
print()

# Decode ESR
esr = cpu0.read_register('ESR_EL3')
ec = (esr >> 26) & 0x3F
iss = esr & 0x1FFFFFF
print(f"=== ESR_EL3 Decode ===")
print(f"EC  = 0x{ec:02X} ({decode_ec(ec)})")
print(f"ISS = 0x{iss:07X}")

esr1 = cpu0.read_register('ESR_EL1')
ec1 = (esr1 >> 26) & 0x3F
dfsc = esr1 & 0x3F
print(f"\n=== ESR_EL1 Decode ===")
print(f"EC   = 0x{ec1:02X} ({decode_ec(ec1)})")
print(f"DFSC = 0x{dfsc:02X} ({decode_dfsc(dfsc)})")

def decode_ec(ec):
    table = {
        0x00: "Unknown", 0x01: "WFI/WFE", 0x07: "SVE/SIMD/FP",
        0x15: "SVC from AArch64", 0x16: "HVC from AArch64",
        0x17: "SMC from AArch64", 0x18: "MSR/MRS trap",
        0x19: "SVE trap", 0x20: "Instruction Abort (lower EL)",
        0x21: "Instruction Abort (same EL)",
        0x24: "Data Abort (lower EL)", 0x25: "Data Abort (same EL)",
        0x26: "SP alignment fault", 0x2C: "FP trap",
    }
    return table.get(ec, f"Unknown EC 0x{ec:02X}")

def decode_dfsc(dfsc):
    table = {
        0x00: "Address size fault L0", 0x01: "Address size fault L1",
        0x04: "Translation fault L0", 0x05: "Translation fault L1",
        0x06: "Translation fault L2", 0x07: "Translation fault L3",
        0x09: "Access flag fault L1", 0x0A: "Access flag fault L2",
        0x0B: "Access flag fault L3", 0x0D: "Permission fault L1",
        0x0E: "Permission fault L2", 0x0F: "Permission fault L3",
        0x10: "Synchronous external abort", 0x21: "Alignment fault",
    }
    return table.get(dfsc, f"Unknown DFSC 0x{dfsc:02X}")
```

---

## Part III: GDB — Source-Level Debugging

### Why GDB?

Iris gives raw registers and memory. GDB gives you:
- Source-level stepping (line by line through C code)
- Symbol resolution (function names, variable names)
- Stack backtraces with function arguments
- Conditional breakpoints ("break when `x == 5`")
- Watchpoints ("stop when this memory address changes")

### Connecting GDB to the FVP

The FVP automatically starts GDB stub servers when launched with `-I`:

```bash
# From WSL2 or a terminal with aarch64-gdb available:
aarch64-linux-gnu-gdb

# Inside GDB:
(gdb) target remote localhost:7101    # CPU0
```

For multi-core debugging:
```
CPU0: localhost:7101
CPU1: localhost:7102
...
CPU15: localhost:7116
SCP (Cortex-M7): localhost:7117  # (port may vary)
```

---

## Part IV: Debugging Each Firmware Layer with Symbols

Every firmware component runs at a different exception level, loads at a different address, and has its own symbol file. Here's how to debug each one.

### 4.1: Debugging SCP Firmware (Cortex-M7)

**What it is:** The System Control Processor handles power domain management, clocking, SCMI messaging. It's an embedded Cortex-M7 running its own firmware.

**When you need to debug it:**
- FVP hangs during early boot (before AP cores power on)
- SCP UART shows assertion failures (like `PCID` errors)
- Power domain transitions hang

**Symbol files:**
```
ROM firmware: scp_romfw.elf
RAM firmware: scp_ramfw.elf
```

**Connecting GDB:**
```bash
arm-none-eabi-gdb scp_ramfw.elf
(gdb) target remote localhost:7117    # SCP's GDB port
```

**Iris approach (without GDB):**
```python
scp = model.get_target('component.RD_N2.css.scp.armcortexm7ct')

# Cortex-M7 registers
pc = scp.read_register('R15')    # PC (ARM naming)
lr = scp.read_register('R14')    # Link Register
sp = scp.read_register('R13')    # Stack Pointer
xpsr = scp.read_register('xPSR')  # Program Status Register

# Check if in fault handler
print(f"PC:   0x{pc:08X}")
print(f"LR:   0x{lr:08X}")
print(f"xPSR: 0x{xpsr:08X}")

# Cortex-M exception: if PC is in range 0x00000000-0x000001FF, 
# it's in the vector table (hard fault, etc.)
```

**Key SCP addresses on RD-N2:**
```
SCP ROM load:   0x00000000 (in SCP's address space)
SCP RAM load:   0x0BD80000 (loaded via --data flag)
SCP UART log:   uart_scp.log
```

**Real scenario we hit:**

The SCP produced this assertion:
```
[SCP_ROMFW] ASSERT: pcid != MOD_PD_INVALID_ID
```

This meant the Power Domain module couldn't find a valid power domain configuration ID. Using the SCP's address map and the assertion source, we traced it to an incompatibility between the SCP firmware version and the platform configuration. The fix was using matching firmware versions from the same ARM reference release.

---

### 4.2: Debugging TF-A (BL1 / BL2 / BL31)

**What it is:** Trusted Firmware-A implements the secure boot chain and runtime services at EL3.

**When you need to debug it:**
- System hangs after SCP boots AP cores
- BL1/BL2 authentication failures (TBBR)
- BL31 exception handling issues (SMC hangs)
- SPMC (Secure Partition Manager) problems

**Symbol files:**
```
BL1: build/rdn2/debug/bl1/bl1.elf
BL2: build/rdn2/debug/bl2/bl2.elf
BL31: build/rdn2/debug/bl31/bl31.elf
```

**Load addresses (RD-N2 DEBUG build):**
```
BL1:  0x00000000 (runs from ROM, copies itself to SRAM)
BL2:  0x04021000 (loaded by BL1 into secure SRAM)
BL31: 0xFF000000 (ARM_BL31_IN_DRAM=1 → secure DRAM at top)
```

> **Critical note on BL31 address:** With `ARM_BL31_IN_DRAM=1`, BL31 loads at `0xFF000000` instead of SRAM. This is the address you'll see in PC when EL3 code is running.

**Connecting GDB to BL31:**
```bash
aarch64-linux-gnu-gdb build/rdn2/debug/bl31/bl31.elf
(gdb) target remote localhost:7101
(gdb) info registers
```

If GDB doesn't auto-resolve symbols (because BL31's base address differs from the ELF's linked address):
```
(gdb) add-symbol-file build/rdn2/debug/bl31/bl31.elf 0xFF000000
```

**Finding BL31's exception loop (our SVE trap scenario):**

When the kernel SVE trap hit BL31, the CPU was stuck in:
```python
pc = cpu0.read_register('PC')  # 0xFF01843C
# This was bl31's unhandled exception WFI loop
```

Using the BL31 map file (`bl31.map`):
```
0xFF018400  plat_panic_handler
0xFF01843C  = plat_panic_handler + 0x3C  ← WFI instruction in loop
```

**TF-A debug print approach:**

TF-A has `LOG_LEVEL` (10=error, 20=notice, 30=warning, 40=info, 50=verbose):
```bash
make ... LOG_LEVEL=50   # Maximum verbosity
```

This goes to `uart_sec.log` (secure UART). But when things crash before logging works, the debugger is the only option.

---

### 4.3: Debugging StandaloneMM (Secure Partition)

**What it is:** The Management Mode secure partition that handles UEFI variable storage. Runs at S-EL0 under the SPMC.

**When you need to debug it:**
- SP hangs after "Secure Partition init start"
- Variable read/write fails from Linux
- NOR flash driver issues

**Symbol files:**
```
MM core + drivers: Build/SgiMmStandalone/DEBUG_GCC5/FV/BL32_AP_MM.fd
Debug symbols:     Build/SgiMmStandalone/DEBUG_GCC5/AARCH64/StandaloneMmPkg/Core/StandaloneMmCore/DEBUG/StandaloneMmCore.dll
Driver symbols:    Build/SgiMmStandalone/DEBUG_GCC5/AARCH64/Platform/ARM/Drivers/NorFlash/StandaloneMmNorFlash/DEBUG/NorFlashStandaloneMm.dll
```

Note: EDK2 builds produce `.dll` files on Linux — these are actually ELF executables with full debug symbols despite the Windows-style name.

**Load address:**

The SPMC loads MM at the address specified in the manifest DTS:
```dts
// rdn2_stmm_config.dts
load-address = <0x0 0xFF200000>;  // StandaloneMM load address
```

So to load symbols in GDB:
```
(gdb) add-symbol-file StandaloneMmCore.dll 0xFF200000
```

But MM has multiple modules loaded at different offsets. The entry point logs report each driver's load address:
```
Loading driver at 0xFF220000 NorFlashStandaloneMm.efi
Loading driver at 0xFF230000 FaultTolerantWriteStandaloneMm.efi
Loading driver at 0xFF240000 VariableStandaloneMm.efi
```

For each driver:
```
(gdb) add-symbol-file NorFlashStandaloneMm.dll 0xFF220000
```

**The challenge: Secure memory is not readable from Iris.**

When we tried to read MM's code region via Iris:
```python
data = cpu0.read_memory(0xFF200000, 32)
# Returns all zeros! TrustZone blocks non-secure access
```

This is because Iris reads memory using the CPU's current security state. If the CPU is executing non-secure code (Linux), secure DRAM reads return zeros. You CAN read secure memory while the CPU is in EL3 (BL31) — timing your reads carefully.

**Our real crash scenario:**

The SM hung. We read registers:
```
ELR_EL1 = 0xFF207DA0   ← The instruction that faulted
ESR_EL1 = 0x92000044   ← Data Abort, Translation Fault L0
FAR_EL1 = 0xFFFFFFFFFFFFFD90  ← The address that faulted
SP_EL0  = 0xFFFFFFFFFFFFFD90  ← Stack pointer!
```

FAR == SP → the SP tried to use its stack, but the stack pointer was garbage. Cross-referencing with `ModuleEntryPoint.S` source:
```asm
cbz    w8, FfaNotEnabled    // PcdFfaEnable == 0 → SKIP stack init!
// ...
adr    x0, StackEnd
mov    sp, x0               // ← Never reached when FFA disabled
```

The fix: `-D EDK2_ENABLE_FFA=TRUE`.

---

### 4.4: Debugging UEFI (DXE / BDS Phase)

**What it is:** The UEFI firmware that initializes hardware, provides boot services, and launches GRUB/Linux.

**When you need to debug it:**
- DXE phase hangs (driver initialization loop)
- Boot menu doesn't appear
- EFI variable access errors
- Boot device enumeration fails

**Symbol files:**

EDK2 produces individual `.dll` (ELF) files for every DXE driver:
```
Build/SgiPkg/DEBUG_GCC5/AARCH64/MdeModulePkg/Core/Dxe/DxeMain/DEBUG/DxeMain.dll
Build/SgiPkg/DEBUG_GCC5/AARCH64/MdeModulePkg/Universal/Variable/RuntimeDxe/DEBUG/VariableRuntimeDxe.dll
Build/SgiPkg/DEBUG_GCC5/AARCH64/ArmPkg/Drivers/ArmGic/ArmGicDxe/DEBUG/ArmGicDxe.dll
```

**Finding load addresses:**

UEFI logs its driver load addresses on the non-secure UART:
```
Loading driver at 0x000F02A0000 EntryPoint=0x000F1D538D8
Loading driver at 0x000F03B0000 EntryPoint=0x000F03B1234
```

Or from the DEBUG build output in `uart_nsec.log`:
```
InstallProtocolInterface: 5B1B31A1-9562-11D2-8E3F-00A0C969723B F02A0000
Loading driver 8F87890C-EE3F-4644-AAC8-B3C0E45E0B34
```

**Loading symbols in GDB:**
```
(gdb) add-symbol-file DxeMain.dll 0xF02A0000
(gdb) add-symbol-file VariableRuntimeDxe.dll 0xF03B0000
```

**Real scenario — DXE hang:**

During our bring-up, UEFI hung in the DXE phase. The non-secure UART just stopped printing. Using Iris:

```python
pc = cpu0.read_register('PC')  # 0xF02B5430
```

Cross-referencing with the UEFI build's map file:
```
0xF02B5400  InitializeUefiPlatformBds
0xF02B5430  = BDS init + 0x30 → inside a PciIo polling loop
```

The GIC (interrupt controller) wasn't configured correctly, so the timer interrupt for timeout detection never fired. The driver waited forever for a PCIe device that would never enumerate without proper interrupt routing.

**Tips for UEFI debugging:**
1. Use DEBUG builds — they print driver GUIDs and load addresses
2. Cross-reference GUIDs with `dec` files to identify which driver is loading
3. The DXE dispatcher loads drivers in dependency order — check `Depex` satisfaction
4. Use `EFI_D_ERROR` level messages for crash forensics

---

### 4.5: Debugging the Linux Kernel

**What it is:** The operating system. It runs at EL1 in the normal world.

**When you need to debug it:**
- Kernel hangs before console output
- Panic during early boot
- Driver initialization issues
- Runtime crashes

**Symbol files:**
```
vmlinux:     The uncompressed kernel ELF with full debug info
System.map:  Symbol table (text file, one symbol per line)
```

**Load address:**

The kernel runs at a virtual address in the `TTBR1_EL1` space:
```
_text = 0xFFFF800008000000  (typical kernel virtual base)
```

But in physical memory, it's loaded at:
```
Physical load address: determined by UEFI's LoadImage
Typically: 0xE0000000 - 0xF0000000 range
```

**Connecting GDB to the kernel:**
```bash
aarch64-linux-gnu-gdb vmlinux
(gdb) target remote localhost:7101
(gdb) bt                           # Backtrace
(gdb) info threads                 # See all CPUs
(gdb) thread 2                     # Switch to CPU1
```

**Using System.map for address resolution:**

When you can't use GDB (or just want a quick lookup), `System.map` provides all kernel symbols:

```bash
# Find what function an address belongs to
grep -B1 "ffff8000080209ec" System.map
# Or find the function containing an address:
awk '$1 <= "ffff8000080209ec"' System.map | tail -1
# → ffff8000080209cc T finalise_el2
# (PC = base + 0x20 → offset into finalise_el2)
```

**Real scenario — Silent kernel boot:**

The kernel loaded but produced zero output. Using Iris:

```python
# Where is the kernel right now?
pc = cpu0.read_register('PC')         # 0xFF01843C (in BL31! Not kernel!)
elr_el3 = cpu0.read_register('ELR_EL3')  # Return address that trapped to EL3

# Check ELR_EL2 — where in EL2 was the kernel when it trapped?
elr_el2 = cpu0.read_register('ELR_EL2')  # 0xFFFF8000080209EC
```

Cross-reference with `System.map`:
```
ffff8000080209cc T finalise_el2
→ ELR_EL2 = finalise_el2 + 0x20
```

The kernel was trying to configure SVE in the EL2 hyp-stub, and the SVE access trapped to EL3. BL31 had no handler → infinite loop.

**Reading kernel data structures:**

```python
# Resolve boot_command_line using System.map
# boot_command_line is at VA 0xFFFF800009B25008
# Need to translate VA → PA first using page table walk

ttbr1 = cpu0.read_register('TTBR1_EL1')  # Kernel page table base

# 4-level page table walk for VA 0xFFFF800009B25008:
# (See ARM64 VA layout with 4KB granule, 48-bit VA)
# Level 0 index = VA[47:39]
# Level 1 index = VA[38:30]  
# Level 2 index = VA[29:21]
# Level 3 index = VA[20:12]
# Page offset   = VA[11:0]

# After walking, found PA = 0xEF725008
data = cpu0.read_memory(0xEF725008, 256)
cmd_line = bytes(data).split(b'\x00')[0].decode()
print(f"boot_command_line: '{cmd_line}'")
# Result: '' (empty!) → setup_arch() was never called
```

---

## Part V: ARM64 Exception Levels — The Debugging Map

### The Exception Level Hierarchy

```
┌─────────────────────────────────────────────────────┐
│  EL3 — Secure Monitor (TF-A BL31)                   │
│  • Highest privilege                                 │
│  • Manages transitions between Secure/Non-Secure    │
│  • Handles SMC calls                                │
│  • Runs SPMC (when SPMC_AT_EL3=1)                  │
├─────────────────────────────────────────────────────┤
│  EL2 — Hypervisor                                    │
│  • Virtualization control                            │
│  • On bare-metal Linux: thin hyp-stub only          │
│  • Handles HVC calls                                │
├─────────────────────────────────────────────────────┤
│  EL1 — OS Kernel (Linux)                            │
│  • Operating system privilege                        │
│  • Page table management, interrupt handling        │
│  • SVC from user space lands here                   │
├─────────────────────────────────────────────────────┤
│  S-EL1 — Secure OS / SPMC (when at SEL2)           │
│  • Secure world kernel                              │
│  • OP-TEE runs here (when not using SPMC at EL3)   │
├─────────────────────────────────────────────────────┤
│  EL0 / S-EL0 — Applications / Secure Partitions    │
│  • Normal: user-space processes                     │
│  • Secure: StandaloneMM runs here (S-EL0)          │
└─────────────────────────────────────────────────────┘
```

### Which Registers Belong to Which Level

Every exception level has its own banked set of key registers:

| Register | EL3 | EL2 | EL1 | Purpose |
|----------|-----|-----|-----|---------|
| `ELR_ELn` | ✓ | ✓ | ✓ | Exception Link Register — return address |
| `ESR_ELn` | ✓ | ✓ | ✓ | Exception Syndrome — what happened |
| `FAR_ELn` | ✓ | ✓ | ✓ | Fault Address — where (for aborts) |
| `SPSR_ELn` | ✓ | ✓ | ✓ | Saved Program State — flags + source EL |
| `SP_ELn` | ✓ | ✓ | ✓ | Stack Pointer for that level |
| `SCTLR_ELn` | ✓ | ✓ | ✓ | System Control — MMU, endianness |
| `TTBR0_ELn` | — | ✓ | ✓ | Translation Table Base 0 |
| `TTBR1_EL1` | — | — | ✓ | Translation Table Base 1 (kernel) |
| `VBAR_ELn` | ✓ | ✓ | ✓ | Vector Base Address — exception table |
| `SCR_EL3` | ✓ | — | — | Secure Configuration |
| `HCR_EL2` | — | ✓ | — | Hypervisor Configuration |

### The Exception Flow

When an exception occurs, the CPU:
1. Saves the current PC into `ELR_ELn` (target EL)
2. Saves the current PSTATE into `SPSR_ELn`
3. Writes the syndrome into `ESR_ELn`
4. For address faults: writes the address into `FAR_ELn`
5. Jumps to `VBAR_ELn + offset` (the exception vector)

**The offset depends on the source:**
```
VBAR + 0x000: Synchronous, from current EL with SP_EL0
VBAR + 0x080: IRQ, from current EL with SP_EL0
VBAR + 0x100: FIQ, from current EL with SP_EL0
VBAR + 0x180: SError, from current EL with SP_EL0
VBAR + 0x200: Synchronous, from current EL with SP_ELx
VBAR + 0x280: IRQ, from current EL with SP_ELx
VBAR + 0x400: Synchronous, from lower EL (AArch64)
VBAR + 0x480: IRQ, from lower EL (AArch64)
VBAR + 0x600: Synchronous, from lower EL (AArch32)
```

---

## Part VI: The Three Calling Conventions — SMC, HVC, SVC

These are the software-triggered exceptions used to call into higher exception levels.

### SVC — Supervisor Call (EL0 → EL1)

**What:** User-space process requests kernel service (syscall).

**Example:**
```asm
// Linux system call from user space
mov    x8, #64        // syscall number (write)
mov    x0, #1         // fd = stdout
ldr    x1, =message   // buffer
mov    x2, #13        // length
svc    #0             // Trap to EL1
```

**Exception level:** EL0 → EL1  
**ESR.EC:** 0x15 (SVC from AArch64)  
**Handler:** Linux kernel's `el0_sync` vector

**Debugging:** If a process hangs on a syscall:
```python
# Check if CPU is in EL1 handling a syscall
esr = cpu0.read_register('ESR_EL1')
ec = (esr >> 26) & 0x3F
if ec == 0x15:
    # SVC trap — read syscall number from X8
    syscall_nr = cpu0.read_register('X8')
    print(f"Stuck in syscall #{syscall_nr}")
```

### HVC — Hypervisor Call (EL1 → EL2)

**What:** The kernel requests hypervisor service (PSCI on bare-metal, or hypercall under KVM).

**Example:**
```asm
// PSCI CPU_ON call (kernel asking to power on a secondary core)
ldr    x0, =0xC4000003   // PSCI CPU_ON function ID
mov    x1, #0x100         // Target CPU MPIDR
ldr    x2, =entry_point   // Where to start executing
mov    x3, #0             // Context ID
hvc    #0                  // Trap to EL2
```

**Exception level:** EL1 → EL2  
**ESR.EC:** 0x16 (HVC from AArch64)  
**Handler:** Hypervisor's `el1_sync` vector (hyp-stub or KVM)

**Important:** On most bare-metal ARM systems, the EL2 hyp-stub immediately forwards PSCI calls up to EL3 via SMC. So the actual flow is:
```
Kernel (EL1) → HVC → Hyp-stub (EL2) → SMC → BL31 (EL3) → PSCI handler
```

**The hyp-stub's finalise_el2 scenario:**

During early kernel boot, `finalise_el2()` uses HVC to drop into EL2 to configure system registers that are only accessible from EL2 (like `ZCR_EL2`, `HCR_EL2`). This is where the SVE trap hit us:
```
Kernel EL1 → HVC → hyp-stub EL2 → MSR ZCR_EL2, X1 → TRAP → EL3 (no handler) → hang
```

### SMC — Secure Monitor Call (EL1/EL2 → EL3)

**What:** Calls into the Secure Monitor (TF-A BL31) for secure services — PSCI, SCMI, FF-A, platform-specific services.

**Example — PSCI SYSTEM_OFF:**
```asm
ldr    x0, =0x84000008   // PSCI SYSTEM_OFF
smc    #0                  // Trap to EL3
// Never returns
```

**Example — FF-A message to StandaloneMM:**
```asm
// FF-A direct message send (variable write to MM)
ldr    x0, =0x84000070   // FFA_MSG_SEND_DIRECT_REQ (32-bit)
mov    x1, #0x0000        // Source: NS world
mov    x2, #0x8001        // Destination: SP ID 0x8001 (StandaloneMM)
mov    x3, #0             // Flags
// x4-x7: message payload (variable operation details)
smc    #0                  // Trap to EL3 → SPMC dispatches to MM
```

**Exception level:** EL1/EL2 → EL3  
**ESR.EC:** 0x17 (SMC from AArch64)  
**Handler:** BL31's runtime service dispatcher

**Debugging SMC issues:**

When an SMC hangs or returns an error:
```python
# Catch the SMC call parameters
x0 = cpu0.read_register('X0')  # Function ID
x1 = cpu0.read_register('X1')  # Arg 1
x2 = cpu0.read_register('X2')  # Arg 2
x3 = cpu0.read_register('X3')  # Arg 3

print(f"SMC Function ID: 0x{x0:08X}")

# Common function IDs:
# 0x84000000 = PSCI_VERSION
# 0x84000001 = PSCI_CPU_SUSPEND
# 0x84000002 = PSCI_CPU_OFF
# 0xC4000003 = PSCI_CPU_ON (64-bit)
# 0x84000008 = PSCI_SYSTEM_OFF
# 0x84000009 = PSCI_SYSTEM_RESET
# 0x84000060 = FFA_ERROR
# 0x84000061 = FFA_SUCCESS
# 0x84000070 = FFA_MSG_SEND_DIRECT_REQ (32-bit calling convention)
# 0xC4000070 = FFA_MSG_SEND_DIRECT_REQ (64-bit calling convention)
```

### The Complete Call Chain for Variable Write

Here's the full exception level traversal when Linux writes a UEFI variable:

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. User space (EL0)                                              │
│    write() syscall to /sys/firmware/efi/efivars/Name-GUID        │
│    → SVC #0 → traps to EL1                                      │
├──────────────────────────────────────────────────────────────────┤
│ 2. Linux kernel (EL1)                                            │
│    efivarfs_file_write() → efi.set_variable()                   │
│    → tee_client_invoke_func() → optee_smc_do_call()            │
│    → arm_smccc_1_1_smc(FFA_MSG_SEND_DIRECT_REQ, ...)           │
│    → SMC #0 → traps to EL3                                      │
├──────────────────────────────────────────────────────────────────┤
│ 3. TF-A BL31 / SPMC (EL3)                                       │
│    spmd_smc_handler() → FFA_MSG_SEND_DIRECT_REQ handler         │
│    Validates caller, looks up destination SP (0x8001)            │
│    Switches context to Secure Partition                          │
│    → ERET to S-EL0                                               │
├──────────────────────────────────────────────────────────────────┤
│ 4. StandaloneMM (S-EL0)                                         │
│    MmEntryPoint() → receives FF-A message                       │
│    VariableServiceSetVariable()                                  │
│    → FtwWrite() → NorFlashWrite(0x1054000000 + offset, data)   │
│    Returns result via FF-A direct response                      │
│    → SVC #0 → back to EL3 (SPMC)                               │
├──────────────────────────────────────────────────────────────────┤
│ 5. Back through the chain                                        │
│    EL3: SPMC returns FFA_MSG_SEND_DIRECT_RESP to NS world       │
│    → ERET to EL1                                                 │
│    EL1: SMC returns, kernel gets EFI_SUCCESS                    │
│    → return to EL0                                               │
│    EL0: write() syscall returns success                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Part VII: Decoding ESR — The Exception Syndrome Register

The ESR (Exception Syndrome Register) is the **single most important register** for debugging crashes. It tells you *exactly* what went wrong.

### ESR Layout

```
Bits [31:26] = EC  (Exception Class — WHAT type of exception)
Bit  [25]    = IL  (Instruction Length — 0=16-bit, 1=32-bit)
Bits [24:0]  = ISS (Instruction Specific Syndrome — details)
```

### Common EC Values (Exception Class)

| EC | Hex | Meaning | Typical Cause |
|----|-----|---------|---------------|
| 0x00 | 0x00 | Unknown reason | |
| 0x01 | 0x01 | WFI/WFE trapped | HCR_EL2.TWI=1 |
| 0x07 | 0x07 | SVE/SIMD/FP access | CPACR_EL1 or CPTR_EL3 trap |
| 0x15 | 0x15 | SVC from AArch64 | System call |
| 0x16 | 0x16 | HVC from AArch64 | Hypervisor call |
| 0x17 | 0x17 | SMC from AArch64 | Secure Monitor call |
| 0x18 | 0x18 | MSR/MRS/System instruction trap | |
| 0x19 | 0x19 | SVE access trapped | CPTR_EL3.EZ=0 |
| 0x20 | 0x20 | Instruction Abort (from lower EL) | Code fetch from unmapped VA |
| 0x21 | 0x21 | Instruction Abort (same EL) | Code fetch from unmapped VA |
| 0x24 | 0x24 | Data Abort (from lower EL) | Load/store to unmapped VA |
| 0x25 | 0x25 | Data Abort (same EL) | Load/store to unmapped VA |
| 0x26 | 0x26 | SP alignment fault | SP not 16-byte aligned |
| 0x2C | 0x2C | FP trap | FPEN not enabled |
| 0x30 | 0x30 | Breakpoint (lower EL) | Software breakpoint |
| 0x32 | 0x32 | Software step (lower EL) | Single-step debug |
| 0x34 | 0x34 | Watchpoint (lower EL) | Data watchpoint hit |
| 0x3C | 0x3C | BRK instruction | Debug breakpoint in code |

### Decoding Data Aborts (EC = 0x24 / 0x25)

For data aborts, the ISS field contains the **DFSC** (Data Fault Status Code):

```
ISS[5:0] = DFSC:
  0x00 = Address size fault, level 0
  0x01 = Address size fault, level 1
  0x04 = Translation fault, level 0    ← Page table entry missing (L0)
  0x05 = Translation fault, level 1    ← Page table entry missing (L1)
  0x06 = Translation fault, level 2    ← Page table entry missing (L2)
  0x07 = Translation fault, level 3    ← Page table entry missing (L3)
  0x09 = Access flag fault, level 1    ← AF bit not set
  0x0A = Access flag fault, level 2
  0x0B = Access flag fault, level 3
  0x0D = Permission fault, level 1     ← Read/write/exec permission denied
  0x0E = Permission fault, level 2
  0x0F = Permission fault, level 3
  0x10 = Synchronous external abort    ← Bus error (device not present)
  0x21 = Alignment fault               ← Unaligned access with SCTLR.A=1

ISS[6]   = WnR: 0=read fault, 1=write fault
ISS[24]  = ISV: 1=valid syndrome info in ISS[23:14]
```

### Real Decoding Examples from Our Debugging

**Example 1: StandaloneMM crash (ESR_EL1 = 0x92000044)**
```
EC    = 0x24 = Data Abort from lower EL
IL    = 1    = 32-bit instruction
DFSC  = 0x04 = Translation fault Level 0
WnR   = 1    = Write access

Meaning: Tried to WRITE to an address whose Level 0 page table entry
         doesn't exist. FAR = 0xFFFFFFFFFFFFFD90 (the garbage SP).
         → Stack was never initialized!
```

**Example 2: SVE trap (ESR_EL3 = 0x66000000)**
```
EC    = 0x19 = SVE functionality trapped
IL    = 1    = 32-bit instruction

Meaning: An SVE instruction (MSR ZCR_EL2, X1) executed at EL2,
         but CPTR_EL3.EZ=0 traps all SVE access to EL3.
         → Need ENABLE_SVE_FOR_NS=1 in TF-A build.
```

**Example 3: SMC from StandaloneMM back to SPMC (ESR_EL3 = 0x5E000000)**
```
EC    = 0x17 = SMC from AArch64
IL    = 1    = 32-bit instruction

Meaning: Normal behavior — SP called SMC to return control to SPMC.
         But combined with the EL1 data abort, this tells us the SP
         faulted, then called back into EL3 as part of its own
         exception handling (or SPMC caught the nested fault).
```

---

## Part VIII: Debugging Strategy — A Systematic Approach

### The Decision Tree

When something hangs on the ARM FVP:

```
System hung
│
├─ Can you see serial output? 
│  ├─ YES → Last message tells you which component is running
│  │        Read registers to find EXACTLY where in that component
│  └─ NO  → Attach debugger, read PC first
│
├─ Where is PC?
│  ├─ In BL31 address range (0xFF000000-0xFF1FFFFF)
│  │  → Exception trapped to EL3. Read ESR_EL3 to decode.
│  │  → Check ELR_EL3 to see where the fault came FROM.
│  ├─ In UEFI address range (0xF0000000-0xF1FFFFFF)
│  │  → UEFI driver hung. Match PC to DXE module addresses.
│  ├─ In Kernel address range (0xFFFF800008000000+)
│  │  → Kernel issue. Use System.map to resolve.
│  └─ In BL1/BL2 address range (0x00000000-0x040FFFFF)
│     → Early boot issue. Load bl1.elf/bl2.elf symbols.
│
├─ Read ESR_ELn at the current EL:
│  ├─ EC = 0x24/0x25 (Data Abort)
│  │  → Read FAR_ELn for the faulting address
│  │  → Decode DFSC for the type of fault
│  │  → Is it a stack issue? (FAR ≈ SP)
│  │  → Is it an MMIO issue? (FAR = device address)
│  │  → Is it a NULL dereference? (FAR ≈ 0)
│  ├─ EC = 0x20/0x21 (Instruction Abort)
│  │  → Code tried to execute from unmapped memory
│  │  → Usually a bad function pointer or corrupted return address
│  ├─ EC = 0x19 (SVE trap)
│  │  → CPTR_EL3.EZ=0, need ENABLE_SVE_FOR_NS=1
│  ├─ EC = 0x17 (SMC)
│  │  → Normal FF-A/PSCI call, check X0 for function ID
│  └─ EC = 0x07 (FP/SIMD trap)
│     → CPACR_EL1.FPEN or CPTR_EL3.TFP issue
│
└─ Still stuck? Get the full register dump and map to source.
```

### The Five Questions

For every hang or crash, answer these five questions:

1. **WHERE** is the CPU? (PC → which component, which function)
2. **WHAT** happened? (ESR → exception class and syndrome)
3. **WHY** did it happen? (FAR, SP, system register state)
4. **WHO** caused it? (SPSR/ELR → trace back to the original caller)
5. **WHEN** in the boot sequence? (UART logs → last known good state)

### Common Patterns

| Symptom | Likely Cause | First Check |
|---------|-------------|-------------|
| PC in WFI loop in BL31 | Unhandled trap from lower EL | ESR_EL3 → decode EC |
| PC in infinite loop, ESR=SMC | SP crashed, called back to SPMC | ESR_EL1 + FAR_EL1 |
| FAR_EL1 = SP_EL0 | Stack pointer never initialized | Check SP setup code |
| FAR_EL1 ≈ 0x0000 | NULL pointer dereference | Find the dereferencing instruction |
| FAR_ELn = valid MMIO address | Device access fault (permissions) | Check page table mapping |
| ESR EC=0x19 | SVE trapped | CPTR_EL3.EZ=0 |
| ESR EC=0x07 | FP/SIMD trapped | CPTR_EL3.TFP or CPACR_EL1.FPEN |
| PC in kernel, SP barely used | Crashed very early in boot | Check EL2 hyp-stub first |

---

## Part IX: Advanced Techniques

### Page Table Walking

When you need to translate a virtual address to physical (to read memory via Iris):

```python
def translate_va_to_pa(cpu, va, ttbr):
    """4-level ARM64 page table walk (4KB granule, 48-bit VA)"""
    
    # Extract indices from VA
    l0_idx = (va >> 39) & 0x1FF  # bits [47:39]
    l1_idx = (va >> 30) & 0x1FF  # bits [38:30]
    l2_idx = (va >> 21) & 0x1FF  # bits [29:21]
    l3_idx = (va >> 12) & 0x1FF  # bits [20:12]
    offset = va & 0xFFF           # bits [11:0]
    
    # L0 table
    l0_base = ttbr & 0xFFFFFFFFF000
    l0_entry_addr = l0_base + l0_idx * 8
    l0_entry = read_u64(cpu, l0_entry_addr)
    if not (l0_entry & 1):  # Valid bit
        return None  # Translation fault L0!
    
    # L1 table
    l1_base = l0_entry & 0xFFFFFFFFF000
    l1_entry_addr = l1_base + l1_idx * 8
    l1_entry = read_u64(cpu, l1_entry_addr)
    if not (l1_entry & 1):
        return None  # Translation fault L1!
    if (l1_entry & 3) == 1:  # Block entry (1GB page)
        return (l1_entry & 0xFFFFC0000000) | (va & 0x3FFFFFFF)
    
    # L2 table
    l2_base = l1_entry & 0xFFFFFFFFF000
    l2_entry_addr = l2_base + l2_idx * 8
    l2_entry = read_u64(cpu, l2_entry_addr)
    if not (l2_entry & 1):
        return None  # Translation fault L2!
    if (l2_entry & 3) == 1:  # Block entry (2MB page)
        return (l2_entry & 0xFFFFFFE00000) | (va & 0x1FFFFF)
    
    # L3 table
    l3_base = l2_entry & 0xFFFFFFFFF000
    l3_entry_addr = l3_base + l3_idx * 8
    l3_entry = read_u64(cpu, l3_entry_addr)
    if not (l3_entry & 3) == 3:  # Page entry
        return None  # Translation fault L3!
    
    return (l3_entry & 0xFFFFFFFFF000) | offset

def read_u64(cpu, phys_addr):
    """Read 8 bytes from physical address, return as uint64"""
    data = cpu.read_memory(phys_addr, 8)
    return int.from_bytes(bytes(data), 'little')
```

### Breakpoint Strategies

**Break on exception entry (catch all traps):**
```
(gdb) break *0xFF000800     # VBAR_EL3 + 0x400 (sync from lower EL, AArch64)
```

**Break on a specific SMC function:**
```
(gdb) break *smc_handler if $x0 == 0x84000070    # FF-A direct message
```

**Break on NOR flash write:**
```
(gdb) watch *(uint32_t *)0x1054000000    # Watchpoint on first flash word
```

**Break when StandaloneMM receives a variable request:**
```
(gdb) add-symbol-file VariableStandaloneMm.dll 0xFF240000
(gdb) break VariableServiceSetVariable
```

### Memory-Mapped I/O Debugging

When a driver hangs talking to hardware, read the device registers:

```python
# PL011 UART status check
def check_uart(cpu, base):
    """Check PL011 UART state"""
    uartdr  = read_u32(cpu, base + 0x000)  # Data Register
    uartfr  = read_u32(cpu, base + 0x018)  # Flag Register
    uartcr  = read_u32(cpu, base + 0x030)  # Control Register
    uartimsc = read_u32(cpu, base + 0x038) # Interrupt Mask
    
    print(f"UART at 0x{base:08X}:")
    print(f"  CR:   0x{uartcr:04X} (UARTEN={'Y' if uartcr&1 else 'N'}, "
          f"TXE={'Y' if uartcr&0x100 else 'N'}, RXE={'Y' if uartcr&0x200 else 'N'})")
    print(f"  FR:   0x{uartfr:04X} (TXFE={'Y' if uartfr&0x80 else 'N'}, "
          f"RXFE={'Y' if uartfr&0x10 else 'N'}, BUSY={'Y' if uartfr&0x08 else 'N'})")

# NS UART (kernel console)
check_uart(cpu0, 0x2A400000)

# Secure UART (TF-A/MM console)
check_uart(cpu0, 0x2A410000)

# NOR Flash register check
flash_rwen = read_u32(cpu0, 0x0C01004C)
print(f"FLASH_RWEN: 0x{flash_rwen:08X} (write enabled: {bool(flash_rwen & 1)})")
```

### Multi-Exception-Level Trace

When you need to trace a call across exception levels:

```python
def full_exception_trace(cpu):
    """Print the complete exception chain state"""
    
    print("=== Current State ===")
    pc = cpu.read_register('PC')
    current_el = cpu.read_register('CurrentEL')
    print(f"PC:        0x{pc:016X}")
    print(f"CurrentEL: EL{current_el >> 2}")
    print()
    
    print("=== EL3 Exception State ===")
    print(f"ELR_EL3:  0x{cpu.read_register('ELR_EL3'):016X}  (return addr into EL3)")
    esr3 = cpu.read_register('ESR_EL3')
    print(f"ESR_EL3:  0x{esr3:08X}  (EC=0x{(esr3>>26)&0x3F:02X})")
    spsr3 = cpu.read_register('SPSR_EL3')
    source_el = (spsr3 >> 2) & 3
    print(f"SPSR_EL3: 0x{spsr3:08X}  (came from EL{source_el})")
    print()
    
    print("=== EL2 Exception State ===")
    print(f"ELR_EL2:  0x{cpu.read_register('ELR_EL2'):016X}")
    esr2 = cpu.read_register('ESR_EL2')
    print(f"ESR_EL2:  0x{esr2:08X}  (EC=0x{(esr2>>26)&0x3F:02X})")
    print()
    
    print("=== EL1 Exception State ===")
    print(f"ELR_EL1:  0x{cpu.read_register('ELR_EL1'):016X}")
    esr1 = cpu.read_register('ESR_EL1')
    far1 = cpu.read_register('FAR_EL1')
    sp0 = cpu.read_register('SP_EL0')
    print(f"ESR_EL1:  0x{esr1:08X}  (EC=0x{(esr1>>26)&0x3F:02X}, DFSC=0x{esr1&0x3F:02X})")
    print(f"FAR_EL1:  0x{far1:016X}")
    print(f"SP_EL0:   0x{sp0:016X}")
    
    if far1 == sp0 and sp0 > 0xFFFFFFFF00000000:
        print("\n⚠️  FAR_EL1 == SP_EL0 (GARBAGE) → Stack was never initialized!")
    elif far1 < 0x1000:
        print(f"\n⚠️  FAR_EL1 near zero → likely NULL pointer dereference")
```

---

## Part X: Putting It All Together — A Complete Debugging Session

Here's how all of this came together when we debugged the StandaloneMM crash. This is the exact sequence of investigation:

### Step 1: Observe the Symptom

Secure UART (`uart_sec.log`) showed:
```
Secure Partition (0x8001) init start
```
Then silence. The old MM binary (without NOR drivers) works fine with the same TF-A.

### Step 2: Connect and Read State

```python
model = NetworkModel('localhost', 7100)
cpu0 = model.get_target('component.RD_N2.css.cluster0.cpu0')

pc = cpu0.read_register('PC')
# → 0xFF01843C (inside BL31 — CPU is trapped in EL3!)
```

### Step 3: Decode the EL3 Exception

```python
esr_el3 = cpu0.read_register('ESR_EL3')  # 0x5E000000
# EC = 0x17 = SMC from AArch64
# The SP called SMC to return to SPMC (normal after a fault)
```

### Step 4: Look at the SP's Fault (EL1)

```python
esr_el1 = cpu0.read_register('ESR_EL1')  # 0x92000044
# EC = 0x24 = Data Abort from lower EL
# DFSC = 0x04 = Translation Fault Level 0

far_el1 = cpu0.read_register('FAR_EL1')  # 0xFFFFFFFFFFFFFD90
sp_el0  = cpu0.read_register('SP_EL0')   # 0xFFFFFFFFFFFFFD90
```

### Step 5: The Eureka Moment

```
FAR_EL1 = SP_EL0 = 0xFFFFFFFFFFFFFD90
```

The SP tried to use its stack, but the stack is at an address that has no page table mapping (Level 0 translation fault). **The stack pointer was never set to a valid address.**

### Step 6: Find the Root Cause in Source

`ModuleEntryPoint.S`:
```asm
ldr    w8, PcdGet32(PcdFfaEnable)
cbz    w8, FfaNotEnabled      ← Branches here when PcdFfaEnable=0
// ... stack init code SKIPPED ...
FfaNotEnabled:
// Falls through to C code with garbage SP
```

Platform DSC: `DEFINE EDK2_ENABLE_FFA = FALSE` → `PcdFfaEnable = 0`

### Step 7: Fix and Verify

```bash
build ... -D EDK2_ENABLE_FFA=TRUE
```

Result: Full MM initialization with all drivers loaded. Variables work. Persistence confirmed.

**Total debugging time with Iris: about 5 minutes.** Without it, we might have spent hours guessing about NOR flash addresses, memory maps, or driver initialization order.

---

## Appendix A: Key Addresses on RD-N2 FVP

| Component | Physical Address | Size | Notes |
|-----------|-----------------|------|-------|
| BL31 (EL3) | 0xFF000000 | 2MB | When ARM_BL31_IN_DRAM=1 |
| StandaloneMM | 0xFF200000 | 3MB | SP code region |
| NOR Flash 0 (FIP) | 0x0C000000 | 64MB | Contains FIP image |
| NOR Flash 2 (Variables) | 0x1054000000 | 64MB | MM variable store |
| NS UART (PL011) | 0x2A400000 | 4KB | Kernel console |
| Secure UART (PL011) | 0x2A410000 | 4KB | TF-A/MM console |
| GIC Distributor | 0x30000000 | 64KB | Interrupt controller |
| GIC Redistributor | 0x300C0000 | 1MB | Per-CPU interface |
| SCP SRAM | 0x0BD80000 | 256KB | SCP RAM firmware |
| DRAM base | 0x80000000 | 2GB+ | Main memory |
| FLASH_RWEN register | 0x0C01004C | 4B | NOR write enable |

## Appendix B: FVP Launch Flags for Debugging

```powershell
$PARAMS = @(
    # Iris debug server (REQUIRED for debugging)
    "-I",                    # Enable Iris server
    "-p",                    # Print ports
    "-R",                    # Run immediately (don't wait for debugger)
    
    # For GDB specifically (alternative to -I):
    # "--cadi-server",       # CADI debug interface
    
    # TrustZone bypass (allows Iris to read secure memory):
    "-C", "css.tzc0.tzc400.rst_gate_keeper=0x0f",
    "-C", "css.tzc0.tzc400.rst_region_attributes_0=0xc000000f",
    "-C", "css.tzc0.tzc400.rst_region_id_access_0=0xffffffff",
    # (repeat for tzc1-tzc7)
    
    # Auto-exit on shutdown (for flash persistence):
    "-C", "css.pl011_ns_uart_ap.shutdown_tag=reboot: Power down",
    
    # Flash write-back:
    "-C", "board.flashloader2.fnameWrite=$IMGDIR\nor2_flash.img",
)
```

## Appendix C: Quick Debugging Cheat Sheet

```python
# === CRASH ANALYSIS TEMPLATE ===
# Copy-paste this after connecting to a hung FVP

import sys
sys.path.insert(0, r'C:\Program Files\ARM\FVP_RD_N2\Iris\Python')
from iris.debug.Model import NetworkModel

model = NetworkModel('localhost', 7100)
cpu = model.get_target('component.RD_N2.css.cluster0.cpu0')

# 1. Where are we?
pc = cpu.read_register('PC')
print(f"PC: 0x{pc:016X}")
if   0xFF000000 <= pc <= 0xFF1FFFFF: print("  → In BL31 (EL3)")
elif 0xFF200000 <= pc <= 0xFF4FFFFF: print("  → In StandaloneMM")
elif 0xF0000000 <= pc <= 0xF1FFFFFF: print("  → In UEFI")
elif pc >= 0xFFFF000000000000:       print("  → In Linux kernel")
else:                                 print(f"  → Unknown region")

# 2. What happened?
for el in [3, 2, 1]:
    esr = cpu.read_register(f'ESR_EL{el}')
    elr = cpu.read_register(f'ELR_EL{el}')
    ec = (esr >> 26) & 0x3F
    if esr != 0:
        print(f"\nEL{el}: ESR=0x{esr:08X} (EC=0x{ec:02X}), ELR=0x{elr:016X}")

# 3. Fault details
far1 = cpu.read_register('FAR_EL1')
sp0 = cpu.read_register('SP_EL0')
print(f"\nFAR_EL1: 0x{far1:016X}")
print(f"SP_EL0:  0x{sp0:016X}")
if far1 == sp0: print("⚠️  STACK FAULT — SP never initialized!")
```

---

## Final Thoughts

The ARM64 architecture is complex — four exception levels, two security worlds, multiple calling conventions, nested page tables, and a zoo of system registers. But the debugging tools match that complexity:

- **Iris** gives you instant, non-intrusive access to the complete machine state
- **GDB** gives you source-level visibility when you need to trace logic
- **ESR/FAR/ELR** registers tell you exactly what went wrong, where, and why
- **The exception level model** means every fault leaves breadcrumbs at the level that caught it

The key insight from our bring-up experience: **don't guess, measure.** Every hour we spent guessing ("maybe it's the NOR flash address?", "maybe it's a memory map issue?") could have been a 30-second Iris query. The debugger doesn't lie. The registers don't have opinions. Read them, decode them, follow the evidence.

---

*This is Part 4 in a series on ARM RD-N2 FVP firmware development:*
1. *[Bringing Up Firmware — A Chronicle of Silent Failures](/posts/arm-rdn2-fvp-firmware-bringup-story/)*
2. *[The Hunt for the Silent Kernel — Debugging the SVE Trap](/posts/debugging-kernel-hang-arm-fvp-sve-trap/)*
3. *[Making Variables Survive Reboot — Enabling StandaloneMM](/posts/enabling-persistent-uefi-variables-arm-fvp-standalonemm/)*
4. *The ARM64 Debugger's Handbook — Iris, GDB, and Live Debugging (this post)*
