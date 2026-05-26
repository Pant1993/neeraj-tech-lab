---
title: "Making Variables Survive Reboot — Enabling StandaloneMM on ARM RD-N2 FVP"
date: 2026-05-26T18:00:00+05:30
draft: false
tags: ["ARM", "FVP", "StandaloneMM", "UEFI", "FF-A", "SPMC", "TF-A", "secure-partition", "NOR-flash", "debugging", "Iris", "persistence", "efivarfs"]
categories: ["Debugging Stories"]
description: "The complete story of enabling persistent UEFI variables on the ARM RD-N2 FVP — from understanding the Secure Partition Manager to debugging a stack pointer that was never initialized, and finally watching a variable survive a full power cycle."
cover:
  image: ""
  alt: "StandaloneMM persistent variables on ARM FVP"
ShowToc: true
TocOpen: true
weight: 0
---

> A story about a Secure Partition that crashed before executing its first C instruction, a stack pointer pointing to `0xFFFFFFFFFFFFFD90`, and the single build flag that made the difference.

---

## The Problem

After [days of firmware bring-up](/posts/arm-rdn2-fvp-firmware-bringup-story/) and [debugging the SVE kernel trap](/posts/debugging-kernel-hang-arm-fvp-sve-trap/), we had a fully booting ARM RD-N2 FVP system:

```
SCP → TF-A (BL1 → BL2 → BL31) → UEFI → GRUB → Linux (buildroot)
```

Everything worked. But there was one glaring limitation: **UEFI variables didn't persist across reboots**.

Every time the FVP powered off and restarted, the EFI variable store was wiped clean. Boot order preferences, OS settings, any runtime configuration — all gone. The reason? Without a Secure Partition handling NOR flash writes, UEFI had no mechanism to commit variables to non-volatile storage in a security-compliant way.

The solution: **StandaloneMM** (Management Mode) — a Secure Partition that runs inside the Secure Partition Manager Core (SPMC), handles variable read/write requests from the normal world via FF-A (Firmware Framework for Arm), and persists them to NOR flash.

How hard could it be?

---

## Chapter 1: Understanding the Architecture

### What is StandaloneMM?

In the ARM firmware architecture, UEFI variables marked as Non-Volatile (NV) need to be written to persistent storage. But on TrustZone-enabled systems, the flash is in the secure world — the normal-world OS (Linux) can't write to it directly.

The flow is:

```
Linux userspace (efivarfs)
    ↓ ioctl
Linux kernel (EFI runtime services)
    ↓ SMC (Secure Monitor Call)
TF-A BL31 / SPMC (EL3)
    ↓ FF-A message passing
StandaloneMM Secure Partition (S-EL0/S-EL1)
    ↓ MMIO
NOR Flash (0x1054000000)
```

StandaloneMM is an EDK2-based firmware binary that:
1. Receives variable read/write requests via FF-A from the normal world
2. Validates them (authentication, size limits, garbage collection)
3. Writes to NOR flash using the Fault Tolerant Write (FTW) protocol
4. Returns status back through the same FF-A channel

### The Components

To enable this, we needed to modify TF-A to include the **SPMC** (Secure Partition Manager Core) at EL3, which would:
- Load the StandaloneMM binary (BL32) during boot
- Set up the secure partition's memory regions and device access
- Handle FF-A calls between the normal world and the SP

### The Golden Rule

We had a golden backup of our working firmware at `C:\Users\neerajpant\rdn2-golden\`. The rule was simple: **never modify the golden copy**. All experiments happen on a working copy.

---

## Chapter 2: Building StandaloneMM

### First Attempt — CPU-Only MM (No Storage)

The initial build was straightforward. EDK2's platform code for SGI/RD-N2 already had a `PlatformStandaloneMm.dsc` file:

```bash
cd /root/rdn2/uefi/edk2 && source edksetup.sh
export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-
export PACKAGES_PATH=/root/rdn2/uefi/edk2:/root/rdn2/uefi/edk2/edk2-platforms

build -a AARCH64 -t GCC5 -p Platform/ARM/SgiPkg/PlatformStandaloneMm.dsc -b DEBUG
```

This produced `BL32_AP_MM.fd` — a minimal MM binary with just the `StandaloneMmCpu` driver (handles FF-A communication) but no storage drivers.

### TF-A with SPMC

The TF-A build needed significant new flags:

```bash
make -j$(nproc) PLAT=rdn2 CROSS_COMPILE=aarch64-linux-gnu- DEBUG=1 LOG_LEVEL=40 \
  TRUSTED_BOARD_BOOT=1 GENERATE_COT=1 CREATE_KEYS=1 \
  ROT_KEY=plat/arm/board/common/rotpk/arm_rotprivk_rsa.pem \
  ARM_ROTPK_LOCATION=devel_rsa MBEDTLS_DIR=/root/rdn2/mbedtls \
  CTX_INCLUDE_FPREGS=0 ENABLE_SVE_FOR_NS=1 ENABLE_SME_FOR_NS=0 \
  ENABLE_SVE_FOR_SWD=1 \
  ARM_BL31_IN_DRAM=1 SPMC_AT_EL3=1 SPD=spmd SPMD_SPM_AT_SEL2=0 \
  EL3_EXCEPTION_HANDLING=1 PLAT_RO_XLAT_TABLES=1 \
  BL32=/root/rdn2/uefi/edk2/Build/SgiMmStandalone/DEBUG_GCC5/FV/BL32_AP_MM.fd \
  BL33=/root/rdn2/output/rdn2/components/css-common/uefi.bin \
  all fip
```

Key new flags:
- `SPMC_AT_EL3=1` — Run the Secure Partition Manager at EL3 (inside BL31)
- `SPD=spmd` — Use the SPM Dispatcher
- `SPMD_SPM_AT_SEL2=0` — Don't use S-EL2 (no secure hypervisor)
- `EL3_EXCEPTION_HANDLING=1` — Handle exceptions from secure partitions
- `ENABLE_SVE_FOR_SWD=1` — Critical! Clear CPTR_EL3.TFP so the SP can use FP/SIMD
- `BL32=...BL32_AP_MM.fd` — The StandaloneMM binary to embed

### First Boot with SPMC — The FP Trap

The first SPMC-enabled boot hung immediately. The secure UART showed:

```
Secure Partition (0x8001) init start
```

And then silence. We'd seen this pattern before with the SVE kernel trap. The `CPTR_EL3.TFP` bit was set, trapping floating-point access from the secure world. The fix was `ENABLE_SVE_FOR_SWD=1` which clears TFP for secure world dispatch.

After that fix, the CPU-only MM booted fine. But variables still weren't persistent — because there were no NOR flash drivers.

---

## Chapter 3: Adding Storage Drivers — The Crash

### Building with SECURE_STORAGE_ENABLE

The EDK2 platform DSC gates the NOR flash, FTW, and Variable drivers behind a build flag:

```bash
build -a AARCH64 -t GCC5 -p Platform/ARM/SgiPkg/PlatformStandaloneMm.dsc \
  -D SECURE_STORAGE_ENABLE -b DEBUG
```

This adds three critical drivers to the MM binary:
- **NorFlashStandaloneMm.efi** — Intel P30 StrataFlash driver for the NOR at `0x1054000000`
- **FaultTolerantWriteStandaloneMm.efi** — Crash-safe write protocol
- **VariableStandaloneMm.efi** — The actual UEFI variable service

### The Immediate Hang

Rebuilt TF-A with the new MM binary, deployed, booted. The secure UART showed:

```
Secure Partition (0x8001) init start
```

And then — **nothing**. Again. But this time `ENABLE_SVE_FOR_SWD` was already set. The old MM binary (without storage drivers) still worked with the exact same TF-A build. Something in the new MM binary was crashing before it could even log a message.

### What We Knew

- Same TF-A build works with old MM → problem is in the new MM binary
- Crash happens before any C-level logging
- The SP manifest (`rdn2_stmm_config.dts`) has correct NOR2 device region:
  ```dts
  nor_flash2 {
      base-address = <0x00000010 0x54000000>;
      pages-count = <16384>;  /* 64MB */
      attributes = <0x3>;     /* read-write */
      "smccc-id" = <0x00000000>;
  };
  ```
- NOR2 address 0x1054000000 is correct (matches hardware configuration)

The guessing game began. Was it a memory mapping issue? A driver initialization problem? An incompatible PCD?

---

## Chapter 4: The Debugger — No More Guessing

We'd been down this road before with the SVE trap. The answer then was "use the debugger." The answer now was the same.

### Connecting via Iris

The ARM FVP comes with **Iris** — a Python-based debug API accessible over TCP port 7100:

```python
import sys
sys.path.insert(0, r'C:\Program Files\ARM\FVP_RD_N2\Iris\Python')
from iris.debug.Model import NetworkModel

model = NetworkModel('localhost', 7100)
cpu0 = model.get_target('component.RD_N2.css.cluster0.cpu0')
```

### Reading the Exception State

Since the SP had crashed and trapped back into EL3, we could read the exception registers:

```python
pc     = cpu0.read_register('PC')        # 0xFF01843C — infinite loop in BL31
elr_el3 = cpu0.read_register('ELR_EL3')  # The return address into EL3
esr_el3 = cpu0.read_register('ESR_EL3')  # 0x5E000000 — SMC from lower EL
```

But we needed the **SP's** exception state — the registers at the moment it faulted:

```python
elr_el1 = cpu0.read_register('ELR_EL1')  # 0xFF207DA0 — faulting instruction
esr_el1 = cpu0.read_register('ESR_EL1')  # 0x92000044
far_el1 = cpu0.read_register('FAR_EL1')  # 0xFFFFFFFFFFFFFD90
sp_el0  = cpu0.read_register('SP_EL0')   # 0xFFFFFFFFFFFFFD90
```

### Decoding the Exception

**ESR_EL1 = 0x92000044:**
- Bits [31:26] EC = 0x24 → **Data Abort from same exception level**
- Bits [5:0] DFSC = 0x04 → **Translation fault at Level 0**

**FAR_EL1 = 0xFFFFFFFFFFFFFD90** — the address that caused the fault.

And there it was. The **Fault Address Register** equaled the **Stack Pointer**:

```
FAR_EL1 = SP_EL0 = 0xFFFFFFFFFFFFFD90
```

The SP tried to access its own stack, and the stack was at an **unmapped, garbage address**. This wasn't a driver bug. This wasn't a NOR flash issue. **The stack pointer was never initialized.**

---

## Chapter 5: The Root Cause — One Build Flag

### Following the Assembly

StandaloneMM's entry point is in `ModuleEntryPoint.S`:

```asm
// StandaloneMmPkg/Library/StandaloneMmCoreEntryPoint/Arm/AArch64/ModuleEntryPoint.S

    .section .data.stack
    .balign 4096
StackBase:
    .fill 8192, 1, 0    // 8KB embedded stack
StackEnd:

_ModuleEntryPoint:
    // ... register setup ...
    
    // Check if FFA is enabled
    ldr    w8, PcdGet32(PcdFfaEnable)
    cbz    w8, FfaNotEnabled      // <-- THIS BRANCH
    
    // FFA enabled path: set memory permissions via SVC, then init stack
    mov    x0, #0
    adr    x1, StackBase
    // ... FfaMemPermSet call ...
    
    adr    x0, StackEnd
    mov    sp, x0                 // <-- STACK INITIALIZED HERE
    
    b      SetupGdt

FfaNotEnabled:
    // SKIPS EVERYTHING ABOVE
    // Falls through to C code with WHATEVER was in SP_EL0
```

When `PcdFfaEnable = 0` (FALSE), the code takes the `FfaNotEnabled` branch which **skips stack initialization entirely**. The SP jumps to C code with whatever garbage the SPMC left in SP_EL0.

### The PCD Chain

In `SgiPlatformMm.dsc.inc` (line 20):
```ini
DEFINE EDK2_ENABLE_FFA = FALSE    # The default!
```

This propagates to:
```ini
gArmTokenSpaceGuid.PcdFfaEnable|0    # When EDK2_ENABLE_FFA=FALSE
```

Which the assembly reads at boot:
```asm
ldr    w8, PcdGet32(PcdFfaEnable)   # Loads 0
cbz    w8, FfaNotEnabled             # Branch taken! Skip stack setup!
```

### Why It Worked Before

The old MM binary (without `SECURE_STORAGE_ENABLE`) was smaller and by coincidence, the SPMC initialized SP_EL0 to a region that happened to be mapped. The moment we added NOR flash drivers, the binary layout changed, and SP_EL0 pointed to unmapped space. **The bug was always there** — it was just latent.

### The Fix

One flag:

```bash
build -a AARCH64 -t GCC5 -p Platform/ARM/SgiPkg/PlatformStandaloneMm.dsc \
  -D SECURE_STORAGE_ENABLE -D EDK2_ENABLE_FFA=TRUE -b DEBUG
```

That's it. `-D EDK2_ENABLE_FFA=TRUE` overrides the platform default, sets `PcdFfaEnable=1`, and the entry point properly initializes the stack from the embedded 8KB `StackBase`/`StackEnd` region.

---

## Chapter 6: StandaloneMM Comes Alive

With `EDK2_ENABLE_FFA=TRUE`, the secure UART log transformed:

```
Secure Partition (0x8001) init start
INFO:    Mapping Software delegated exception access page
INFO:    Software Delegated Exception buffer setup done
Booting Secure Partition (0x8001)...

StandaloneMm[0x8001] entry point
NorFlashPlatformInitialization: enabling NOR flash write (FLASH_RWEN at 0x0C01004C)
NorFlashInitialise: found NOR flash device at 0x1054000000 (256KB blocks, 64MB)
FvbInitialize: Locating FVB header in NOR flash...
ValidateFvHeader: Initializing new variable store (fresh flash)
InitializeFvAndVariableStoreHeaders: Writing FV + variable headers
FtwInitialize: Fault Tolerant Write initialized with spare block at offset 0x80000
Variable driver will work with auth variable support!
StandaloneMmCpu: FF-A communication driver loaded

INFO:    Secure Partition (0x8001) initialized successfully
```

All four drivers loaded:
1. ✅ **NorFlashStandaloneMm** — Initialized NOR2 at 0x1054000000
2. ✅ **FaultTolerantWrite** — Configured spare block for crash recovery
3. ✅ **Variable** — Ready to service authenticated variable operations
4. ✅ **StandaloneMmCpu** — FF-A message handler active

The full system booted all the way to Linux:

```
BL1 → BL2 → BL31/SPMC → StandaloneMM (initialized) → UEFI → GRUB → Linux
```

---

## Chapter 7: Writing Our First Persistent Variable

### From Linux

The Linux kernel exposes UEFI variables through `efivarfs`:

```bash
# Login to buildroot
buildroot login: root

# Mount the EFI variable filesystem
mount -t efivarfs efivarfs /sys/firmware/efi/efivars

# Write a test variable
# Format: 4 bytes attributes + data
# 0x07 = NV (0x01) + BS (0x02) + RT (0x04) = Non-Volatile + Boot Service + Runtime
printf '\x07\x00\x00\x00REBOOT_OK' > \
  /sys/firmware/efi/efivars/BootSurvival-12345678-1234-1234-1234-123456789abc

# Verify it was written
hexdump -C /sys/firmware/efi/efivars/BootSurvival-12345678-1234-1234-1234-123456789abc
```

Output:
```
00000000  07 00 00 00 52 45 42 4f  4f 54 5f 4f 4b           |....REBOOT_OK|
0000000d
```

The write went through! The FF-A call chain worked:
```
Linux efivarfs → kernel SMC → BL31 SPMC → StandaloneMM → NOR flash write
```

But the real question: **does it survive a reboot?**

---

## Chapter 8: The Persistence Problem — FVP Flash Write-Back

### The Discovery

We issued `poweroff` from Linux. The kernel shut down cleanly:

```
The system is going down NOW!
Stopping network: OK
Stopping klogd: OK
reboot: Power down
```

But the FVP process **didn't exit**. It just stopped the simulation. And when we force-killed the FVP and rebooted — the variable was gone.

### Understanding FVP Flash Behavior

The FVP's NOR flash model has two parameters:
- `board.flashloader2.fname` — File to load flash contents FROM at startup
- `board.flashloader2.fnameWrite` — File to save flash contents TO **on simulation exit**

The key word is **"on simulation exit."** The FVP only writes flash contents back to disk when it performs a **graceful simulation termination**. Force-killing the process (`Kill()`) skips the write-back entirely.

### What Doesn't Work

We tried everything:
- **`CloseMainWindow()`** — Sends WM_CLOSE but FVP doesn't exit after guest poweroff
- **`Kill()` / `Stop-Process`** — Force terminates, no write-back
- **Iris `model.release()`** — Disconnects debug client, doesn't end simulation
- **Guest `poweroff`** — Halts CPUs but FVP process stays alive

### The Solution: `shutdown_tag`

Buried in the FVP's parameter list (`--list-params`) was exactly what we needed:

```
css.pl011_ns_uart_ap.shutdown_tag=    # Shutdown simulation when a string is transmitted
```

When the kernel prints `"reboot: Power down"` during shutdown, the UART transmits those exact bytes. If we set `shutdown_tag` to match that string, the FVP will **end the simulation gracefully** — triggering the flash write-back!

```powershell
"-C", "css.pl011_ns_uart_ap.shutdown_tag=reboot: Power down",
```

### The Write-Back in Action

With `shutdown_tag` configured, the FVP output on poweroff:

```
Info: /OSCI/SystemC: Simulation stopped by user.
Info: RD_N2: RD_N2.board.flashloader1: FlashLoader: Saved 64MB to file 'nor1_flash.img'
Info: RD_N2: RD_N2.board.flashloader2: FlashLoader: Saved 64MB to file 'nor2_flash.img'
```

**Flash saved!** And an important detail: the FVP saves flash in **gzip-compressed format** (file starts with `1F 8B 08 00`), but transparently decompresses it on load.

---

## Chapter 9: The Final Test — Does It Survive?

### Boot 1: Write the Variable

```bash
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
printf '\x07\x00\x00\x00REBOOT_OK' > \
  /sys/firmware/efi/efivars/BootSurvival-12345678-1234-1234-1234-123456789abc
hexdump -C /sys/firmware/efi/efivars/BootSurvival-12345678-1234-1234-1234-123456789abc
# 00000000  07 00 00 00 52 45 42 4f  4f 54 5f 4f 4b  |....REBOOT_OK|
poweroff
```

FVP exits gracefully. Flash written to disk. Hash changes from all-zeros to `2D24384A...`.

### Boot 2: Read It Back

Fresh FVP launch with the same `nor2_flash.img`:

```bash
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
ls /sys/firmware/efi/efivars/ | grep BootSurvival
# BootSurvival-12345678-1234-1234-1234-123456789abc

hexdump -C /sys/firmware/efi/efivars/BootSurvival-12345678-1234-1234-1234-123456789abc
# 00000000  07 00 00 00 52 45 42 4f  4f 54 5f 4f 4b  |....REBOOT_OK|
```

**🎉 PERSISTENCE CONFIRMED.**

The variable `REBOOT_OK` survived a complete power cycle. Written on boot 1, read back identically on boot 2. The entire chain works:

```
Write: Linux → SMC → SPMC → StandaloneMM → NOR flash → disk (via shutdown_tag)
Read:  Disk → FVP flash model → StandaloneMM → FF-A → Linux efivarfs
```

---

## The Final Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Linux (Normal World)                   │
│  efivarfs: /sys/firmware/efi/efivars/                   │
│  ┌─────────────────────────────────────────────────┐    │
│  │  printf '\x07\x00\x00\x00DATA' > VarName-GUID  │    │
│  └──────────────────────┬──────────────────────────┘    │
└─────────────────────────┼───────────────────────────────┘
                          │ SMC (FF-A)
┌─────────────────────────┼───────────────────────────────┐
│  TF-A BL31 / SPMC (EL3) │                               │
│  ┌──────────────────────┼──────────────────────────┐    │
│  │  SPMD dispatches FF-A message to SP 0x8001      │    │
│  └──────────────────────┼──────────────────────────┘    │
└─────────────────────────┼───────────────────────────────┘
                          │ FF-A msg_send_direct
┌─────────────────────────┼───────────────────────────────┐
│  StandaloneMM Secure Partition (S-EL0)                   │
│  ┌──────────────┐ ┌──────────┐ ┌───────────────────┐   │
│  │ Variable.efi │→│ FTW.efi  │→│ NorFlash.efi      │   │
│  │ Auth + GC    │ │ Crash-   │ │ P30 StrataFlash   │   │
│  │              │ │ safe     │ │ @ 0x1054000000    │   │
│  └──────────────┘ └──────────┘ └─────────┬─────────┘   │
└───────────────────────────────────────────┼─────────────┘
                                            │ MMIO
┌───────────────────────────────────────────┼─────────────┐
│  NOR Flash (64MB, 256KB blocks)           │             │
│  ┌────────────────────────────────────────┼────────┐    │
│  │  FV Header │ Var Store │ FTW Spare │ ...       │    │
│  │  (block 0) │ (block 1) │ (block 2) │          │    │
│  └────────────────────────────────────────────────┘    │
│  Backed by: nor2_flash.img (gzip compressed)           │
│  Saved on: FVP graceful exit (shutdown_tag trigger)    │
└─────────────────────────────────────────────────────────┘
```

---

## Lessons Learned

### 1. Build Defaults Are Dangerous

The platform DSC had `EDK2_ENABLE_FFA = FALSE` as the default. This made perfect sense for legacy (non-FF-A) boot flows where the SPMC initializes the SP's stack. But when using the modern FF-A SPMC at EL3, the SP must initialize its own stack. **The default was wrong for our use case**, and the failure was completely silent — no assert, no error log, just a crash before the first line of C code ran.

### 2. The Debugger Beats Guessing Every Time

We could have spent hours theorizing about NOR flash addresses, memory map conflicts, or driver initialization order. Instead, 30 seconds with the Iris debugger gave us:
- The exact faulting instruction (`ELR_EL1 = 0xFF207DA0`)
- The exception type (Data Abort, Translation Fault Level 0)
- The smoking gun (`FAR_EL1 = SP_EL0 = garbage`)

The call stack told us everything: **the SP never had a valid stack**.

### 3. FVP Flash Requires Graceful Exit

The FVP's flash write-back only happens on graceful simulation termination. Force-killing the process loses all flash changes. The `shutdown_tag` parameter is the elegant solution — set it to match a kernel shutdown message, and the FVP auto-exits with flash saved:

```
css.pl011_ns_uart_ap.shutdown_tag=reboot: Power down
```

### 4. Latent Bugs Hide in Success

The `EDK2_ENABLE_FFA=FALSE` bug was present in the CPU-only MM build too. It "worked" by accident because the SPMC happened to leave SP_EL0 in a mapped region. Adding NOR flash drivers changed the binary layout just enough to expose the real bug. **A working system isn't necessarily a correct system.**

---

## Quick Reference

### Build Commands

**StandaloneMM (with persistent storage):**
```bash
cd /root/rdn2/uefi/edk2 && source edksetup.sh
export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-
export PACKAGES_PATH=/root/rdn2/uefi/edk2:/root/rdn2/uefi/edk2/edk2-platforms

build -a AARCH64 -t GCC5 -p Platform/ARM/SgiPkg/PlatformStandaloneMm.dsc \
  -D SECURE_STORAGE_ENABLE -D EDK2_ENABLE_FFA=TRUE -b DEBUG
```

**TF-A with SPMC:**
```bash
make -j$(nproc) PLAT=rdn2 CROSS_COMPILE=aarch64-linux-gnu- DEBUG=1 LOG_LEVEL=40 \
  TRUSTED_BOARD_BOOT=1 GENERATE_COT=1 CREATE_KEYS=1 \
  ROT_KEY=plat/arm/board/common/rotpk/arm_rotprivk_rsa.pem \
  ARM_ROTPK_LOCATION=devel_rsa MBEDTLS_DIR=/root/rdn2/mbedtls \
  CTX_INCLUDE_FPREGS=0 ENABLE_SVE_FOR_NS=1 ENABLE_SME_FOR_NS=0 \
  ENABLE_SVE_FOR_SWD=1 \
  ARM_BL31_IN_DRAM=1 SPMC_AT_EL3=1 SPD=spmd SPMD_SPM_AT_SEL2=0 \
  EL3_EXCEPTION_HANDLING=1 PLAT_RO_XLAT_TABLES=1 \
  BL32=/root/rdn2/uefi/edk2/Build/SgiMmStandalone/DEBUG_GCC5/FV/BL32_AP_MM.fd \
  BL33=/root/rdn2/output/rdn2/components/css-common/uefi.bin \
  all fip
```

### FVP Parameters for Persistence

```powershell
# In run_rdn2.ps1:
"-C", "board.flashloader2.fname=$IMGDIR\nor2_flash.img",
"-C", "board.flashloader2.fnameWrite=$IMGDIR\nor2_flash.img",
"-C", "css.pl011_ns_uart_ap.shutdown_tag=reboot: Power down",
```

### Writing Variables from Linux

```bash
mount -t efivarfs efivarfs /sys/firmware/efi/efivars

# Attributes: 0x07 = NV + BS + RT
printf '\x07\x00\x00\x00YOUR_DATA' > \
  /sys/firmware/efi/efivars/VarName-12345678-1234-1234-1234-123456789abc

# Read back
hexdump -C /sys/firmware/efi/efivars/VarName-12345678-1234-1234-1234-123456789abc

# Delete (required before overwrite)
chattr -i /sys/firmware/efi/efivars/VarName-*  # Remove immutable flag
rm /sys/firmware/efi/efivars/VarName-*
```

### Iris Debugger Quick Reference

```python
import sys
sys.path.insert(0, r'C:\Program Files\ARM\FVP_RD_N2\Iris\Python')
from iris.debug.Model import NetworkModel

model = NetworkModel('localhost', 7100)
cpu0 = model.get_target('component.RD_N2.css.cluster0.cpu0')

# Read exception state
pc      = cpu0.read_register('PC')
elr_el1 = cpu0.read_register('ELR_EL1')  # Faulting instruction
esr_el1 = cpu0.read_register('ESR_EL1')  # Exception syndrome
far_el1 = cpu0.read_register('FAR_EL1')  # Fault address
sp_el0  = cpu0.read_register('SP_EL0')   # Stack pointer

# ESR decoding:
# EC [31:26]: 0x24 = Data Abort, 0x25 = Data Abort (same EL)
# DFSC [5:0]: 0x04 = Translation fault L0, 0x05 = L1, 0x06 = L2, 0x07 = L3
```

---

## What's Next

With persistent UEFI variables working, the RD-N2 FVP platform is now a realistic emulation of a real ARM server:
- Boot preferences persist across power cycles
- Secure variable storage via TrustZone + FF-A
- Crash-safe writes via Fault Tolerant Write protocol
- Graceful shutdown with automatic flash save

The natural next steps:
- **Secure Boot**: Use persistent variables to store PK/KEK/db for UEFI Secure Boot
- **RELEASE build**: Switch from DEBUG to RELEASE for faster simulation
- **Automated testing**: Script the boot-write-reboot-verify cycle
- **Variable authentication**: Test authenticated variable writes (db/dbx updates)

From "FVP installed" to "persistent secure variables" — a journey through SCP hangs, UEFI DXE stalls, SVE traps, certificate errors, a crash-before-first-instruction bug, and FVP flash semantics. Every layer of the ARM firmware stack had its own surprise. But in the end, the variable survived the reboot. And that made it all worth it.

---

*This is Part 3 in a series on ARM RD-N2 FVP firmware development:*
1. *[Bringing Up Firmware — A Chronicle of Silent Failures](/posts/arm-rdn2-fvp-firmware-bringup-story/)*
2. *[The Hunt for the Silent Kernel — Debugging the SVE Trap](/posts/debugging-kernel-hang-arm-fvp-sve-trap/)*
3. *Making Variables Survive Reboot — Enabling StandaloneMM (this post)*
