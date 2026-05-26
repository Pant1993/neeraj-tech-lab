---
title: "The Hunt for the Silent Kernel — Debugging a Linux Boot Hang on ARM RD-N2 FVP"
date: 2026-05-26T12:10:00+05:30
draft: false
tags: ["ARM", "FVP", "Linux", "debugging", "SVE", "TF-A", "kernel", "UEFI", "Iris", "firmware", "EL3", "exception-levels"]
categories: ["Debugging Stories"]
description: "A detailed debugging story of tracking down why Linux produced zero console output after 'Exiting boot services' on an ARM RD-N2 FVP — from GRUB parameters to page table walks to finding an SVE trap in EL3."
cover:
  image: ""
  alt: "Debugging kernel hang on ARM FVP"
ShowToc: true
TocOpen: true
weight: 1
---

> A story about a kernel that refused to speak, a UART that was perfectly fine, and an SVE instruction that trapped into the abyss.

---

## The Setup

We had come a long way. After days of building firmware from source, fixing SCP PCID assertions, patching UEFI DXE hangs, and wrestling with TBBR certificates, we finally had a working boot chain on the ARM Neoverse RD-N2 Fixed Virtual Platform (FVP):

```
SCP → TF-A (BL1 → BL2 → BL31) → UEFI → GRUB → Linux kernel
```

GRUB loaded. It found the kernel. The EFI stub ran. And then:

```
Loading driver at 0x000F02A0000 EntryPoint=0x000F1D538D8
EFI stub: Booting Linux Kernel...
EFI stub: EFI_RNG_PROTOCOL unavailable
EFI stub: Using DTB from configuration table
EFI stub: Exiting boot services...
```

And then... **nothing**. No earlycon. No panic. No `start_kernel`. No dmesg. Complete radio silence.

The kernel had entered a black hole.

---

## Chapter 1: The Obvious Suspects — Console Parameters

The first question was obvious: **did we configure the console correctly?**

### Attempt 1: Basic earlycon

The original GRUB config had no console parameters at all:

```
linux /Image acpi=force ip=dhcp root=PARTUUID=9c53a91b-e182-4ff1-aeac-6ee2c432ae94 rootwait verbose debug
```

We added the PL011 UART address (0x2A400000 — the NS UART on RD-N2):

```
earlycon=pl011,0x2A400000 console=ttyAMA0,115200
```

**Result:** Nothing.

### Attempt 2: mmio32 format

ARM PL011 on RD-N2 uses 32-bit MMIO access. Maybe the earlycon driver needs the explicit access type:

```
earlycon=pl011,mmio32,0x2A400000
```

**Result:** Still nothing.

### Attempt 3: The kitchen sink

We threw everything at it:

```
acpi=force earlycon=pl011,mmio32,0x2A400000 console=ttyAMA0,115200 loglevel=8 ignore_loglevel keep_bootcon nokaslr earlyprintk=serial,ttyS0,115200
```

**Result:** Nothing. Not a single byte.

### Attempt 4: Plain earlycon (let SPCR handle it)

The UEFI SPCR (Serial Port Console Redirection) ACPI table should tell the kernel about the PL011 at 0x2A400000. Maybe explicit parameters were interfering:

```
acpi=force earlycon console=ttyAMA0,115200
```

**Result:** Nothing.

### What we verified

- ✅ Kernel Image has `CONFIG_SERIAL_AMBA_PL011=y`
- ✅ Kernel has `CONFIG_SERIAL_AMBA_PL011_CONSOLE=y`
- ✅ Kernel has `CONFIG_SERIAL_EARLYCON=y`
- ✅ Kernel has `CONFIG_EFI_EARLYCON=y`
- ✅ Kernel has `CONFIG_ACPI=y`
- ✅ Kernel is valid ARM64 (MZ+ARMd magic, 39MB, Linux 6.1.0)
- ✅ All UART log files checked (scp, mcp, nsec, sec, soc0) — zero kernel output anywhere

**Conclusion:** The console parameters are not the problem. The kernel is not reaching the point where it *parses* these parameters.

---

## Chapter 2: FVP UART Configuration Experiments

Maybe the FVP model itself was misconfigured. We experimented with several model parameters:

### UART enable flag

```powershell
"-C", "css.pl011_ns_uart_ap.uart_enable=true"
```

Added explicitly to ensure the UART IP is enabled in the model. Didn't help.

### DC4 toggle

The `enable_dc4` parameter controls whether the DC4 character (0x14) pauses UART output. We tried both:

```powershell
"-C", "css.pl011_ns_uart_ap.enable_dc4=1"   # match official config
"-C", "css.pl011_ns_uart_ap.enable_dc4=0"   # disable entirely
```

Neither made a difference.

### Flow control mask

```powershell
"-C", "css.pl011_ns_uart_ap.flow_ctrl_mask_en=1"
```

Already present — confirmed correct per ARM's reference scripts.

### What we learned from ARM's reference scripts

A research agent dug through ARM's official boot scripts and found:

1. **Official grub.cfg uses NO earlycon** — console comes only via ACPI SPCR
2. ARM's `boot.sh` waits up to **7200 seconds** (2 hours!) for the busybox prompt
3. `enable_dc4=1` IS required for NS UART on newer FVP versions
4. `flow_ctrl_mask_en=1` is critical

**The question became:** Is the kernel just extremely slow, or is it actually stuck?

---

## Chapter 3: Enter the Iris Debugger

The FVP comes with an **Iris debug interface** — a Python API that lets you inspect CPU state, read/write registers, and access memory while the simulation runs. This was our breakthrough tool.

### Enabling Iris

Add `-I -p` to the FVP launch command:

```powershell
$PARAMS = @(
    "-I", "-p", "-R",   # Iris server on port 7100, run immediately
    ...
)
```

### Connecting via Python

```python
import sys
sys.path.insert(0, r"C:\Program Files\ARM\FVP_RD_N2\Iris\Python")
import iris.iris as iris

with iris.connect("RD_N2", 7100, "localhost", timeoutInMs=10000) as model:
    caller = model.irisCall()
    # CPU0 instance ID = 15 (component.RD_N2.css.cluster0.cpu0)
```

### First observation: CPU0 is trapped in EL3

```python
# Read PC and CPSR
pc = read_reg("PC")      # → 0x040347D0
cpsr = read_reg("CPSR")  # → 0x620002CD (EL3h)
```

Wait — **EL3**? The kernel runs at EL1 (or EL2 for the hyp-stub). If the CPU is at EL3 executing BL31 code, that means an exception trapped execution back to the secure monitor. The kernel isn't running at all — it's stuck.

But we also saw:
```
ELR_EL2 = 0xFFFF8000080209EC  (valid kernel virtual address!)
```

So the kernel *did* start. Something happened that bounced the CPU back to EL3.

---

## Chapter 4: Is the UART Even Working?

Before diving deeper into the trap, we needed to verify the UART hardware wasn't broken. Using Iris, we read the PL011 registers directly:

```python
# PL011 NS UART at physical address 0x2A400000
UARTCR = memory_read(0x2A400030, 4)  # → 0x0B01
UARTFR = memory_read(0x2A400018, 4)  # → 0x97
```

Decoding:
- **UARTCR = 0x0B01**: UARTEN=1 (enabled), TXE=1 (TX enabled), RXE=1 (RX enabled) ✅
- **UARTFR = 0x97**: TXFF=0 (TX FIFO not full), TXFE=1 (TX FIFO empty) ✅

The UART is fully configured and ready to accept data. Can we write to it directly?

```python
# Write 'H', 'i', '\n' directly to UART data register
memory_write(0x2A400000, 4, [0x48])  # 'H'
memory_write(0x2A400000, 4, [0x69])  # 'i'
memory_write(0x2A400000, 4, [0x0A])  # '\n'
```

Checked `uart_nsec.log`:
```
Hi
```

**UART hardware works perfectly.** The kernel simply never writes to it.

---

## Chapter 5: Walking the Kernel's Page Tables

If the kernel started but isn't printing anything, where exactly did it stop? We used the Iris debugger to walk the kernel's page tables and inspect internal kernel data structures.

### The setup

```
TTBR1_EL1 = 0xF0266000  (kernel page tables)
SCTLR_EL1: MMU = ON
```

ARM64 page tables use 4KB granule, 4-level walk (L0→L1→L2→L3). For kernel VAs (0xFFFF...), we use TTBR1_EL1.

### Virtual-to-Physical translation

We resolved key kernel symbols using `System.map` (159,102 symbols):

| Symbol | Virtual Address | Physical Address | Value |
|--------|----------------|-----------------|-------|
| `_text` | 0xFFFF800008000000 | 0xEDC00000 | Valid MZ header ✅ |
| `boot_command_line` | 0xFFFF800009B25008 | 0xEF725008 | **EMPTY (all zeros)** ❌ |
| `__log_buf` | 0xFFFF80000A5E1258 | 0xF01E1258 | **ALL ZEROS** ❌ |
| `jiffies_64` | — | — | **0xFFFEDB08 = INITIAL_JIFFIES** ❌ |

This was devastating evidence:

- **`boot_command_line` is empty** → `setup_arch()` was never called
- **`__log_buf` is zeros** → `printk()` was never called
- **`jiffies_64` = INITIAL_JIFFIES** → The timer was never started
- **`_text` has valid kernel code** → The kernel IS loaded correctly

The kernel image is sitting in memory, perfectly loaded, but it never reached `start_kernel()`. Something crashed it before that point.

### Stack analysis

```
SP_EL1 = 0xFFFF80000A203EA0
init_thread_union = 0xFFFF80000A200000 (THREAD_SIZE = 0x4000)
```

Stack usage: `0xA204000 - 0xA203EA0 = 0x160` bytes. That's **only 352 bytes** of stack used — the kernel barely started executing.

---

## Chapter 6: The Trap — Finding the Smoking Gun

Now we knew the kernel crashed *extremely* early. Let's look at the exception registers:

```python
ELR_EL2 = 0xFFFF8000080209EC   # Where EL1 kernel was when it HVC'd to EL2
ELR_EL3 = 0xEEC9C89C           # Where EL2 code was when it trapped to EL3
SPSR_EL3 = 0x620003C9          # Previous state: EL2h (Hypervisor mode)
ESR_EL3 = 0x66000000           # THE EXCEPTION SYNDROME
```

### Decoding ESR_EL3 = 0x66000000

```
Bits [31:26] = EC (Exception Class) = 0x19
```

**EC = 0x19 = "Access to SVE functionality trapped"**

An SVE (Scalable Vector Extension) register access from EL2 trapped to EL3!

### The register that caused it

```python
CPTR_EL3 = 0x0   # Coprocessor Trap Register at EL3
```

**CPTR_EL3 bit[8] (EZ) = 0** → SVE instructions from ANY lower exception level trap to EL3.

### Identifying the faulting instruction

Using `System.map`, we resolved the addresses:

```
ELR_EL2 = 0xFFFF8000080209EC = finalise_el2 + 0x20
```

The kernel's `finalise_el2()` at EL1 does an HVC to drop into the EL2 hyp-stub, which runs `__finalise_el2`. This function configures various EL2 features including SVE:

```asm
// From arch/arm64/kernel/hyp-stub.S
__finalise_el2:
    ...
    // SVE configuration
    mov    x1, #0xF              // ZCR_EL2.LEN = max
    msr    ZCR_EL2, x1          // ← THIS INSTRUCTION TRAPS!
    ...
```

The physical address 0xEEC9C89C corresponds to `__finalise_el2 + 0x4C` — the `MSR ZCR_EL2, X1` instruction.

### The complete trap chain

```
1. Kernel (EL1) calls finalise_el2()
2. finalise_el2() does HVC → enters EL2 hyp-stub
3. __finalise_el2 at EL2 executes: MSR ZCR_EL2, X1
4. CPTR_EL3.EZ = 0 → SVE access traps to EL3
5. BL31's EL3 exception handler receives EC=0x19
6. BL31 has NO handler for SVE traps
7. CPU enters infinite loop at PC=0x040347D0 (WFI)
8. Kernel is dead. No output will ever appear.
```

**This happens BEFORE `start_kernel()` is even called.** That's why every console parameter we tried was irrelevant — the kernel never gets far enough to parse them.

---

## Chapter 7: Why Does This Happen?

### The ARM exception level architecture

```
EL3 (Secure Monitor) — TF-A BL31
EL2 (Hypervisor)     — Hyp-stub (temporary, configures EL2 then returns to EL1)
EL1 (Kernel)         — Linux
EL0 (User)           — Applications
```

### CPTR_EL3 controls what EL2/EL1 can access

`CPTR_EL3` is a register at EL3 that controls trapping of specific instruction classes:
- **Bit 8 (EZ)**: When 0, SVE instructions from EL2/EL1 trap to EL3
- **Bit 12 (ESM)**: When 0, SME instructions from EL2/EL1 trap to EL3

TF-A sets `CPTR_EL3 = 0x0` during BL31 initialization. This means SVE is completely blocked for the non-secure world.

### Why doesn't BL31 handle the trap?

TF-A's EL3 exception handler handles specific trap types:
- SMC calls (for PSCI, SCMI proxy, etc.)
- FIQ interrupts (for secure world)
- Synchronous exceptions from secure partitions

But **EC=0x19 (SVE trap) is not handled**. The default handler just loops forever — which is what we observed at PC=0x040347D0.

### The platform config that caused it

In TF-A's source:

```makefile
# plat/arm/css/sgi/sgi-common.mk line 30:
CTX_INCLUDE_FPREGS:=1
```

And in the main Makefile:
```makefile
ifeq (${CTX_INCLUDE_FPREGS},1)
    ifeq (${ENABLE_SVE_FOR_NS},1)
        $(warning "ENABLE_SVE_FOR_NS cannot be used with CTX_INCLUDE_FPREGS")
        $(warning "Forced ENABLE_SVE_FOR_NS=0")
        override ENABLE_SVE_FOR_NS:= 0
    endif
endif
```

The SGI platform (which RD-N2 inherits from) sets `CTX_INCLUDE_FPREGS=1` for FP/NEON context switching. But this flag is **mutually exclusive** with `ENABLE_SVE_FOR_NS=1`. So SVE support for the non-secure world is forcibly disabled, and CPTR_EL3.EZ stays 0.

### The kernel's assumption

Linux 6.1.0 with `CONFIG_ARM64_SVE=y` (which our kernel has) unconditionally tries to configure ZCR_EL2 during early boot in `__finalise_el2`. It assumes either:
1. SVE is allowed from EL2 (CPTR_EL3.EZ=1), or
2. The hypervisor handles the trap gracefully

Neither is true here. BL31 just dies silently.

---

## Chapter 8: The Fix

We had three options:

| Option | Approach | Trade-off |
|--------|----------|-----------|
| A | Rebuild TF-A with `ENABLE_SVE_FOR_NS=1` | Proper fix, enables SVE for Linux |
| B | Rebuild kernel with `CONFIG_ARM64_SVE=n` | Loses SVE support |
| C | Patch BL31 to handle the trap | Complex, still wouldn't enable SVE |

We chose **Option A** — the proper fix.

### The rebuild command

```bash
cd /root/rdn2/arm-tf

make -j$(nproc) \
  PLAT=rdn2 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  DEBUG=0 \
  TRUSTED_BOARD_BOOT=1 \
  GENERATE_COT=1 \
  CREATE_KEYS=1 \
  ROT_KEY=plat/arm/board/common/rotpk/arm_rotprivk_rsa.pem \
  ARM_ROTPK_LOCATION=devel_rsa \
  MBEDTLS_DIR=/root/rdn2/mbedtls \
  CTX_INCLUDE_FPREGS=0 \
  ENABLE_SVE_FOR_NS=1 \
  ENABLE_SME_FOR_NS=1 \
  ARM_BL31_IN_DRAM=1 \
  BL33=/root/rdn2/output/rdn2/components/css-common/uefi.bin \
  all fip
```

Key changes from the previous build:
- **`CTX_INCLUDE_FPREGS=0`** — Override the platform default to allow SVE
- **`ENABLE_SVE_FOR_NS=1`** — Set CPTR_EL3.EZ=1, allow SVE from EL2/EL1
- **`ENABLE_SME_FOR_NS=1`** — Also enable SME (the kernel configures this too in `__finalise_el2`)

### Deploy and boot

```powershell
# Copy to Windows
cp build/rdn2/release/bl1.bin /mnt/c/Users/neerajpant/rdn2-images/tf-bl1.bin
cp build/rdn2/release/fip.bin /mnt/c/Users/neerajpant/rdn2-images/fip-uefi.bin
```

### The result

```
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd490]
[    0.000000] Linux version 6.1.0-g4810bccbf96e ...
[    0.000000] ACPI: SPCR: console: pl011,mmio32,0x2a400000,115200
[    0.000000] earlycon: pl11 at MMIO32 0x000000002a400000 (options '115200')
[    0.000000] Kernel command line: BOOT_IMAGE=/Image acpi=force earlycon console=ttyAMA0,115200 ...
...
[    0.386538] Freeing unused kernel memory: 7936K
[    0.386556] Run /init as init process
Starting syslogd: OK
Starting klogd: OK
Running sysctl: OK
Starting network: OK

Welcome to Buildroot
buildroot login: root
#
```

**1607 lines of kernel output.** Full boot to shell. The kernel was never slow — it was trapped from the very first instruction that touched SVE.

---

## Chapter 9: The Debugging Toolkit

Here's a summary of the tools and techniques that made this possible:

### Iris Python Debug API

The FVP's built-in debug interface was indispensable:

```python
import iris.iris as iris

with iris.connect("RD_N2", 7100, "localhost", timeoutInMs=10000) as model:
    caller = model.irisCall()
    
    # Get register list
    regs = caller.callIrisFunction("resource_getList", instId=15)
    
    # Read register by ID
    val = caller.callIrisFunction("resource_read", instId=15, rscIds=[reg_id])
    
    # Read physical memory
    data = caller.callIrisFunction("memory_read", instId=15, 
                                    spaceId=5,        # Non-Secure Physical
                                    address=0x2A400000, 
                                    count=1, 
                                    byteWidth=4)
    
    # Write physical memory (e.g., UART data register)
    caller.callIrisFunction("memory_write", instId=15,
                           spaceId=5, address=0x2A400000,
                           count=1, byteWidth=4, data=[0x41])
```

Key details:
- **CPU0 instance ID**: 15 (`component.RD_N2.css.cluster0.cpu0`)
- **Memory spaces**: spaceId=4 (Secure Physical), spaceId=5 (Non-Secure Physical)
- **Register IDs**: Get from `resource_getList`, index by name
- **Port**: 7100 (enable with `-I -p` flags)

### Page Table Walking

ARM64 4KB granule, 48-bit VA, 4-level tables:

```
VA bits [47:39] → L0 index (into TTBR1_EL1 table for kernel VAs)
VA bits [38:30] → L1 index
VA bits [29:21] → L2 index (2MB block descriptor, or table pointer)
VA bits [20:12] → L3 index (4KB page descriptor)
VA bits [11:0]  → Page offset
```

For our kernel:
- `_text` at VA `0xFFFF800008000000` → PA `0xEDC00000`
- VA-to-PA formula: `PA = VA - 0xFFFF7FFF1A400000`

### System.map for Symbol Resolution

```bash
# 159,102 symbols available
grep "finalise_el2" System.map
# ffff8000080209cc T finalise_el2
# ffff80000909c850 t __finalise_el2
```

### ESR (Exception Syndrome Register) Decoding

```
ESR_EL3 = 0x66000000
Bits [31:26] = EC = 0b011001 = 0x19 = "SVE access trap"
Bits [25]    = IL = 1 (32-bit instruction)
Bits [24:0]  = ISS = 0 (no additional syndrome info for this EC)
```

Reference: ARM Architecture Reference Manual, Table D17-1.

---

## Chapter 10: The Complete Boot Chain Fix List

Here's every issue we fixed to go from "FVP installed" to "Linux shell prompt":

| # | Problem | Symptom | Root Cause | Fix |
|---|---------|---------|------------|-----|
| 1 | SCP bus fault | Silent hang, no UART output | CMN-700 PCID mismatch (FVP r1 vs firmware r0 values) | Comment out `return FWK_E_DATA` in `mod_sid.c` |
| 2 | SCP CMN access blocked | "Domain is power down" warnings, SCP RAM hangs | FVP power domain not fully modeled | Add `powerStateGate.gate_behaviour=1` |
| 3 | UEFI DXE hang | Boot stuck after 13 DXE drivers | `InitIoVirtSocExpBlkUartControllers()` accessing address 0 | Set `PcdIoVirtSocExpBlkUartEnable\|0` in RdN2.dsc |
| 4 | TF-A TBBR cert failure | "Failed to load image id 4 (-2)" | FIP missing TOS certificates for BL32 | Rebuild TF-A without SPM_MM, with MBEDTLS |
| 5 | **Kernel SVE trap** | **Zero console output after "Exiting boot services"** | **CPTR_EL3.EZ=0, MSR ZCR_EL2 traps to unhandled EL3 exception** | **Rebuild TF-A with `ENABLE_SVE_FOR_NS=1 CTX_INCLUDE_FPREGS=0`** |

---

## Chapter 11: GRUB Configuration (Final Working)

```
set default=0
set timeout=3

menuentry "RD-N2 Buildroot" {
    linux /Image acpi=force earlycon console=ttyAMA0,115200 loglevel=8 ignore_loglevel keep_bootcon nokaslr root=PARTUUID=9c53a91b-e182-4ff1-aeac-6ee2c432ae94 rootwait
}
```

### Parameter explanations

| Parameter | Purpose |
|-----------|---------|
| `acpi=force` | Force ACPI mode (RD-N2 kernel 6.1 has no DTS for this platform) |
| `earlycon` | Let SPCR ACPI table configure earlycon automatically |
| `console=ttyAMA0,115200` | Main console on PL011 UART |
| `loglevel=8` | Maximum kernel log verbosity |
| `ignore_loglevel` | Print all messages regardless of level |
| `keep_bootcon` | Don't unregister boot console when real console starts |
| `nokaslr` | Disable KASLR (helps debugging with fixed addresses) |
| `rootwait` | Wait indefinitely for root device (virtio block is slow on FVP) |

---

## Chapter 12: FVP Launch Script (Final Working)

```powershell
$MODEL = "C:\Program Files\ARM\FVP_RD_N2\models\Win64_VC2019\FVP_RD_N2.exe"
$IMGDIR = "C:\Users\neerajpant\rdn2-images"

$PARAMS = @(
    "-I", "-p", "-R",  # Iris debug server + run immediately

    # Firmware loading
    "--data", "css.scp.armcortexm7ct=$IMGDIR\scp_ramfw.bin@0x0BD80000",
    "--data", "css.mcp.armcortexm7ct=$IMGDIR\mcp_ramfw.bin@0x0BF80000",
    "-C", "css.mcp.ROMloader.fname=$IMGDIR\mcp_romfw.bin",
    "-C", "css.scp.ROMloader.fname=$IMGDIR\scp_romfw.bin",
    "-C", "css.trustedBootROMloader.fname=$IMGDIR\tf-bl1.bin",
    "-C", "board.flashloader0.fname=$IMGDIR\fip-uefi.bin",
    "-C", "board.virtioblockdevice.image_path=$IMGDIR\grub-buildroot.img",

    # UART configuration
    "-C", "css.scp.pl011_uart_scp.out_file=$IMGDIR\uart_scp.log",
    "-C", "css.scp.pl011_uart_scp.unbuffered_output=1",
    "-C", "css.scp.pl011_uart_scp.uart_enable=true",
    "-C", "css.pl011_ns_uart_ap.out_file=$IMGDIR\uart_nsec.log",
    "-C", "css.pl011_ns_uart_ap.unbuffered_output=1",
    "-C", "css.pl011_ns_uart_ap.flow_ctrl_mask_en=1",
    "-C", "css.pl011_ns_uart_ap.enable_dc4=1",

    # Power state gate bypass (SCP CMN700 access)
    "-C", "css.scp.powerStateGate.gate_behaviour=1",
    "-C", "css.scp.powerStateGate_cmn.gate_behaviour=1",
    "-C", "css.mcp.powerStateGate.gate_behaviour=1",
    "-C", "css.mcp.powerStateGate_cmn.gate_behaviour=1",

    # GIC + PCIe
    "-C", "css.gic_distributor.ITS-device-bits=20",
    "-C", "pcie_group_0.pciex16.hierarchy_file_name=<default>",

    # TZC-400 bypass (all 8 instances)
    "-C", "css.tzc0.tzc400.rst_gate_keeper=0x0f",
    "-C", "css.tzc0.tzc400.rst_region_attributes_0=0xc000000f",
    "-C", "css.tzc0.tzc400.rst_region_id_access_0=0xffffffff"
    # ... repeat for tzc1 through tzc7
)

& $MODEL @PARAMS
```

---

## Chapter 13: Lessons Learned

### 1. Silent failures are the hardest bugs

The SCP PCID assertion killed SCP before UART initialized — completely silent. The SVE trap killed the kernel before `start_kernel()` — completely silent. Both had the same symptom: "nothing happens." The only way to debug them was to attach a low-level debugger and inspect raw CPU state.

### 2. "Wait longer" is sometimes wrong

ARM's scripts wait 7200 seconds. We initially assumed the FVP was just slow. But a stuck CPU at EL3 in a WFI loop will *never* produce output, no matter how long you wait. The Iris debugger proved within seconds that the CPU was trapped, not slow.

### 3. Firmware configuration has invisible dependencies

`CTX_INCLUDE_FPREGS=1` (set by the platform makefile) silently forces `ENABLE_SVE_FOR_NS=0`. This is documented in TF-A's Makefile as a warning, not an error. But the consequence — a kernel that instantly dies — is catastrophic and completely non-obvious.

### 4. The FVP's Iris API is incredibly powerful

Being able to read/write registers, walk page tables, inspect memory, and even write directly to UART hardware from Python — all while the simulation runs — made this debugging possible. Without it, we'd still be guessing.

### 5. Exception registers tell the story

`ESR_EL3`, `ELR_EL3`, `SPSR_EL3`, `CPTR_EL3` — these four registers told us exactly what happened:
- What trapped (SVE access, EC=0x19)
- Where it trapped from (ELR_EL3 → `__finalise_el2+0x4C`)
- What exception level was running (SPSR_EL3 → EL2h)
- Why it trapped (CPTR_EL3.EZ = 0)

---

## Epilogue

The complete debugging journey took us from "the console parameters must be wrong" through "maybe the FVP is slow" to "let me read the CPU registers" to "the kernel is stuck in a firmware exception handler because of an SVE instruction that traps to EL3."

The fix was a single build flag: `ENABLE_SVE_FOR_NS=1`. But finding that flag required:

1. Proving the UART worked (direct memory writes via Iris)
2. Proving the kernel never executed `start_kernel()` (page table walk → empty `__log_buf`)
3. Proving the CPU was trapped at EL3 (register inspection)
4. Decoding the exception syndrome (ESR_EL3 → EC=0x19 → SVE trap)
5. Finding the trap control register (CPTR_EL3.EZ = 0)
6. Tracing back to the build system (`CTX_INCLUDE_FPREGS=1` → forced SVE disable)

Every step eliminated a hypothesis and narrowed the search space. No guessing — just systematic forensics.

```
buildroot login: root
#
```

*Finally.*

---

## Appendix: Key Addresses and Constants

| Item | Value |
|------|-------|
| NS UART PL011 base | 0x2A400000 |
| BL31 WFI loop (stuck PC) | 0x040347D0 |
| `finalise_el2` (EL1 wrapper) | 0xFFFF8000080209CC |
| `__finalise_el2` (EL2 handler) | 0xFFFF80000909C850 |
| Faulting instruction PA | 0xEEC9C89C |
| Kernel `_text` PA | 0xEDC00000 |
| Kernel VA-to-PA offset | 0xFFFF7FFF1A400000 |
| TTBR1_EL1 | 0xF0266000 |
| CPTR_EL3 (before fix) | 0x0 |
| CPTR_EL3 (after fix) | 0x100 (EZ=1) |
| ESR_EL3 (trap syndrome) | 0x66000000 (EC=0x19) |
| CPU0 Iris instId | 15 |
| Iris port | 7100 |

---

*Written while debugging on the ARM RD-N2 FVP, May 2026.*
