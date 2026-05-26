---
title: "Bringing Up Firmware on the ARM RD-N2 FVP — A Chronicle of Silent Failures"
date: 2026-05-26T12:25:00+05:30
draft: false
tags: ["ARM", "FVP", "firmware", "SCP", "TF-A", "UEFI", "TBBR", "debugging", "CMN-700", "bring-up", "GRUB"]
categories: ["Debugging Stories"]
description: "The complete story of bringing up the ARM RD-N2 FVP from scratch — every silent hang, every cryptic failure, and every fix needed to get from 'FVP installed' to 'Exiting boot services'. SCP PCID assertions, power gate bypasses, UEFI DXE hangs, and TBBR certificate wrestling."
cover:
  image: ""
  alt: "ARM FVP firmware bring-up"
ShowToc: true
TocOpen: true
weight: 2
---

> Four firmware components. Five silent failures. Zero error messages. Welcome to platform bring-up.

---

## Prologue: What We're Building

The ARM Neoverse RD-N2 Fixed Virtual Platform (FVP) is a cycle-approximate software model of a server-class ARM SoC. It has:

- **16 Neoverse N2 cores** in 16 clusters
- **CMN-700** coherent mesh network (the interconnect backbone)
- **GIC-700** interrupt controller with ITS
- **8× TZC-400** TrustZone controllers
- **5× SMMUv3** I/O MMU instances
- **PCIe Gen4** x16/x8/x4 root complex
- **SCP** (System Control Processor) — a Cortex-M7 that orchestrates power and clocks
- **MCP** (Management Control Processor) — another Cortex-M7 for platform management

The boot chain is:

```
SCP ROM → SCP RAM → (powers AP cores) → TF-A BL1 → BL2 → BL31 → UEFI → GRUB → Linux
```

Every single one of these transitions broke at least once. This is the story of fixing them all.

---

## Chapter 1: The Build Environment

### Setting up the workspace

The ARM Reference Design firmware is built from multiple source repositories synchronized via Google's `repo` tool:

```bash
# WSL2 Ubuntu 22.04, as root
mkdir /root/rdn2 && cd /root/rdn2
repo init -u https://git.gitlab.arm.com/arm-reference-solutions/arm-reference-solutions-manifest.git \
    -m pinned-rdn2.xml -b refs/tags/RD-INFRA-2022.12.22
repo sync -j8
```

This pulls down:
- `scp/` — SCP firmware (Cortex-M7 bare-metal)
- `arm-tf/` — Trusted Firmware-A (BL1, BL2, BL31)
- `uefi/` — EDK2 UEFI implementation
- `linux/` — Linux kernel source
- `grub/` — GRUB bootloader
- `buildroot/` — Minimal root filesystem
- `build-scripts/` — ARM's build orchestration scripts
- `model-scripts/` — FVP launch scripts

### The dependency gauntlet

The build system assumes a well-provisioned CI machine. On a fresh WSL2 Ubuntu, we needed:

```bash
apt install -y \
    build-essential cmake git python3 \
    aarch64-linux-gnu-gcc arm-none-eabi-gcc \
    libfdt-dev autoconf automake autopoint libtool \
    mtools rsync cpio bc wget file unzip uuid-dev \
    python3-distutils libssl-dev
```

### Cross-compilation landmines

**kvmtool** needed a cross-compiled `libfdt` for aarch64. The build system's check assumed a native build. Fix:

```makefile
# Makefile line 345: bypass the libfdt check
ifneq (y,y)    # was: ifneq ($(call try-build,...),y)
```

Plus explicit include/library paths for the aarch64 libfdt we built from the `dtc` source.

**SCP firmware** required the ARM embedded toolchain (`arm-none-eabi-gcc`), not the Linux cross-compiler. And it needed the CMSIS submodule:

```bash
cd scp/
git submodule update --init contrib/cmsis/git
```

**CMN-700 module** had `-Werror` with unused variable warnings under GCC 10.3:

```cmake
# module/cmn700/CMakeLists.txt — append at end:
target_compile_options(module-cmn700 PRIVATE -Wno-error)
```

### The full build

After fixing all dependencies, the complete stack build took about 45 minutes:

```bash
./build-scripts/build-all.sh -p rdn2 -f buildroot all
```

Output: a set of binaries ready for the FVP, plus a GRUB disk image with Linux kernel and buildroot rootfs.

---

## Chapter 2: First Boot — The SCP Bus Fault

### The FVP installation

On Windows, the FVP installs to:
```
C:\Program Files\ARM\FVP_RD_N2\models\Win64_VC2019\FVP_RD_N2.exe
```

Version: **Fast Models 11.25.23** (April 2024).

### Launch script

We created a PowerShell launch script that loads all firmware and configures the model:

```powershell
$PARAMS = @(
    "--data", "css.scp.armcortexm7ct=$IMGDIR\scp_ramfw.bin@0x0BD80000",
    "--data", "css.mcp.armcortexm7ct=$IMGDIR\mcp_ramfw.bin@0x0BF80000",
    "-C", "css.scp.ROMloader.fname=$IMGDIR\scp_romfw.bin",
    "-C", "css.mcp.ROMloader.fname=$IMGDIR\mcp_romfw.bin",
    "-C", "css.trustedBootROMloader.fname=$IMGDIR\tf-bl1.bin",
    "-C", "board.flashloader0.fname=$IMGDIR\fip-uefi.bin",
    "-C", "board.virtioblockdevice.image_path=$IMGDIR\grub-buildroot.img",
    # ... UART config, TZC bypass, etc.
)
& $MODEL @PARAMS
```

### The symptom: absolute silence

We launched the FVP, connected to telnet port 5000 (SCP UART), and saw... **nothing**. No SCP boot messages. No error. The model consumed CPU but produced zero output.

The SCP firmware on real hardware always prints version info and initialization progress. Something was killing it before UART initialized.

### The detective work

We knew the SCP boots from ROM, initializes its module framework, then brings up peripherals (including UART). If it dies before UART init, there's no way to see what happened — unless you look at the FVP model's own warnings:

```
Warning: RD_N2: css.scp.powerStateGate: Domain is power down,
  transaction to address 0x6000800, will be aborted.
```

Address `0x6000800` — that's the **CMN-700 configuration space**. The SCP was trying to access CMN-700 registers, but the model reported the power domain as "down" and **aborted the transaction**.

But wait — this was the `powerStateGate` warning. There was a *different* failure happening even before this. The SCP was silently returning an error from a module that initializes very early.

### Root cause: SCP SID Module PCID Check

Deep in the SCP module initialization sequence, the **SID (System Identification)** module runs `mod_pcid_check_registers()`. This function reads hardware PCID registers and compares them against compiled-in expected values.

The problem: **FVP 11.25.23 models CMN-700 revision r1p0**, but the firmware source (`RD-INFRA-2022.12.22`) was built expecting **CMN-700 r0p0** PCID values.

The function fails, returns `FWK_E_DATA`, and the SCP framework **stops initialization**. Since the SID module initializes *before* the PL011 UART module, the failure is **completely silent**.

```c
// module/sid/src/mod_sid.c, lines 70-72 (the killer)
if (!mod_pcid_check_registers()) {
    return FWK_E_DATA;  // ← SCP dies here, silently
}
```

Reference: [ARM-software/SCP-firmware GitHub Issue #781](https://github.com/ARM-software/SCP-firmware/issues/781)

### The fix

Comment out the fatal return:

```c
if (!mod_pcid_check_registers()) {
    // return FWK_E_DATA;  // Non-fatal: allow boot to continue
}
```

This must be applied to **both** `scp_romfw` and `scp_ramfw` builds, since both contain the SID module.

### Rebuild SCP

```bash
cd /root/rdn2/scp

# ROM firmware
cmake -B cmake-build/rdn2/scp_romfw \
    -DSCP_FIRMWARE_SOURCE_DIR:PATH=rdn2/scp_romfw \
    -DSCP_TOOLCHAIN=GNU \
    -DSCP_PLATFORM_VARIANT=0
cmake --build cmake-build/rdn2/scp_romfw -j$(nproc)

# RAM firmware (same flags, different source dir)
cmake -B cmake-build/rdn2/scp_ramfw \
    -DSCP_FIRMWARE_SOURCE_DIR:PATH=rdn2/scp_ramfw \
    -DSCP_TOOLCHAIN=GNU \
    -DSCP_PLATFORM_VARIANT=0
cmake --build cmake-build/rdn2/scp_ramfw -j$(nproc)
```

Output binaries: `rdn2-bl1.bin` (ROM) and `rdn2-bl2.bin` (RAM).

---

## Chapter 3: Second Silence — The Power State Gate

After the PCID fix, we relaunched. The SCP ROM firmware booted! We could see initialization messages in `uart_scp.log`. But then SCP RAM firmware loaded and... hung again.

### The symptom

SCP ROM completes, hands off to SCP RAM firmware. RAM firmware begins module initialization, gets to the CMN-700 module, and stops dead. Again, no error message.

### The FVP warning

```
Warning: RD_N2: css.scp.powerStateGate: Domain is power down,
  transaction to address 0x6000800, will be aborted.
```

The CMN-700 module in SCP RAM firmware tries to configure the mesh network. But the FVP model's **power state gate** thinks the CMN-700 power domain isn't up yet. The gate intercepts the SCP's memory transactions and **aborts them** — causing silent hangs or corrupt reads.

### Why this happens

On real hardware, the SCP controls power domains directly through PPU (Power Policy Unit) registers. It turns on the CMN-700 power domain *before* accessing CMN registers. But the FVP model's power domain simulation doesn't perfectly track the SCP's PPU writes. There's a timing gap where the model still thinks the domain is powered down.

### The fix: bypass the power gate

The FVP has a model parameter to change the gate's behavior from "abort" to "allow":

```powershell
"-C", "css.scp.powerStateGate.gate_behaviour=1",      # SCP core gate
"-C", "css.scp.powerStateGate_cmn.gate_behaviour=1",   # SCP CMN gate
"-C", "css.mcp.powerStateGate.gate_behaviour=1",       # MCP core gate
"-C", "css.mcp.powerStateGate_cmn.gate_behaviour=1",   # MCP CMN gate
```

`gate_behaviour=1` means "allow transactions even if the domain appears powered down." The model still logs warnings, but the transactions complete.

### Result

SCP fully boots:
```
[    0.000001] [SCP]: Firmware version b34e5ce
[    0.001203] [CMN700]: Mesh configuration complete
[    0.001587] [SCMI]: Protocol discovery complete
[    0.002505] [PCIE]: Found device: 8:0:1
...
```

SCP powers up the application processor cores via SCMI. TF-A BL1 starts executing.

---

## Chapter 4: TF-A Boots, UEFI Hangs — The DXE Disaster

### TF-A success

With SCP working, the boot chain progressed beautifully:

```
NOTICE:  BL1: v2.8(release):v2.8-354-g...
NOTICE:  BL1: Built : ...
NOTICE:  BL1: Booting BL2
NOTICE:  BL2: v2.8(release):v2.8-354-g...
NOTICE:  BL31: v2.8(release):v2.8-354-g...
```

BL1 loaded BL2 from the FIP (Firmware Image Package). BL2 loaded BL31. BL31 set up the secure monitor and launched UEFI (BL33).

### UEFI partial success

UEFI started and loaded DXE (Driver Execution Environment) drivers. We could see:

```
LocateAndInstallAcpiFromFv: installed ACPI tables
InitVirtioDevices: Installed Virtio Block device
InitVirtioDevices: Installed Virtio Network device
```

Then... silence. The log stopped growing. UEFI was stuck.

### Six days of running

We initially thought the FVP was just slow (it runs at ~1/100th real-time). We left it running for **a few hours**. The log never grew past 8484 bytes. Same 13 DXE drivers loaded, nothing more.

This was a genuine hang, not slowness.

### Source code analysis

The last function called in the successful log was `InitVirtioDevices()`. Looking at the PlatformDxe source:

```c
// Platform/ARM/SgiPkg/Drivers/PlatformDxe/PlatformDxe.c
EFI_STATUS ArmSgiPkgEntryPoint(...) {
    LocateAndInstallAcpiFromFv();   // ✅ works
    InitVirtioDevices();             // ✅ works
    InitIoVirtSocExpBlkUartControllers();  // ← HANGS HERE
    ...
}
```

### Root cause: ghost UART addresses

`InitIoVirtSocExpBlkUartControllers()` reads a PCD (Platform Configuration Database) value for UART addresses on an "IO Virtualization SoC Expansion Block." This PCD is declared as DYNAMIC and defaults to `{0x0}`:

```ini
# RdN2.dsc
gArmSgiTokenSpaceGuid.PcdIoVirtSocExpBlkUartEnable|1    # Enabled!
gArmSgiTokenSpaceGuid.PcdIoVirtSocExpBlkUarts|{0x0}     # But address = 0
```

The function is enabled (`PcdIoVirtSocExpBlkUartEnable=1`) but the UART addresses are all zeros. When UEFI tries to access PL011 registers at physical address **0x0**, the FVP's bus model hangs — no bus fault, no abort, just eternal silence.

### The fix

Disable the entire function:

```ini
# Platform/ARM/SgiPkg/RdN2/RdN2.dsc
gArmSgiTokenSpaceGuid.PcdIoVirtSocExpBlkUartEnable|0
```

### Rebuilding UEFI

```bash
cd /root/rdn2/uefi/edk2

# Set up EDK2 build environment
export PACKAGES_PATH=/root/rdn2/uefi/edk2:/root/rdn2/uefi/edk2/edk2-platforms
source edksetup.sh
export PATH=$PWD/BaseTools/BinWrappers/PosixLike:$PATH

# Build for RD-N2
build -n $(nproc) -a AARCH64 -t GCC5 -b RELEASE \
    -p Platform/ARM/SgiPkg/RdN2/RdN2.dsc
```

Output: `Build/RdN2/RELEASE_GCC5/FV/BL33_AP_UEFI.fd` — our new UEFI binary.

---

## Chapter 5: The TBBR Certificate Nightmare

### What is TBBR?

ARM's **Trusted Board Boot Requirements** (TBBR) is a secure boot chain. Every binary in the FIP is accompanied by X.509 certificates that contain the hash of the image. BL1 verifies BL2's certificate, BL2 verifies BL31's and BL33's (UEFI), and so on.

The certificates are signed with a Root of Trust (ROT) key. If you change any binary, its hash no longer matches the certificate, and the boot chain **rejects it**.

### The problem

We rebuilt UEFI (BL33). But the FIP still contained the *old* certificate for the old UEFI binary. The hash wouldn't match.

### Attempt 1: Build FIP without TBBR

```bash
make PLAT=rdn2 TRUSTED_BOARD_BOOT=0 ... fip
```

**Result:** BL1 immediately errors:
```
ERROR: Loading of FW_CONFIG failed -2
```

BL1 was built *with* TBBR. It expects the certificate structure in the FIP. A FIP built without TBBR has a different layout that BL1 can't parse.

### Attempt 2: Rebuild TF-A without TBBR

If BL1 expects TBBR, rebuild BL1 without it:

```bash
make PLAT=rdn2 TRUSTED_BOARD_BOOT=0 SPM_MM=1 ...
```

**Result:** BL2 panics at `plat_get_next_bl_params`:
```
PANIC at PC : 0x0405e040
```

The StandaloneMM binary (BL32/`mm_standalone.bin`) was built for the SPMC framework, which is incompatible with a fresh TF-A build that uses SPM_MM.

### Attempt 3: Rebuild TF-A with SPMC_AT_EL3

```bash
make PLAT=rdn2 SPMC_AT_EL3=1 SPD=spmd ...
```

**Result:** BL31 overflows its RAM region by 4-20KB:
```
region `RAM' overflowed by 15832 bytes
```

Even with `ARM_BL31_IN_DRAM=1`, the linker script region was too small for a DEBUG build with SPMC.

### Attempt 4: Build without SPM entirely

The `mm_standalone.bin` was causing compatibility issues no matter what. What if we just... don't include it?

```bash
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
    ARM_BL31_IN_DRAM=1 \
    BL33=/root/rdn2/output/rdn2/components/css-common/uefi.bin \
    all fip
```

No SPM_MM. No BL32. Just BL1 + BL2 + BL31 + BL33 (UEFI). TBBR enabled with auto-generated certificates.

**Result:** SUCCESS! New FIP builds cleanly. All certificates generated. BL1 verifies BL2, BL2 loads BL31 and BL33.

### The key insight

The TBBR system is self-contained IF you let TF-A generate all certificates from scratch using `GENERATE_COT=1 CREATE_KEYS=1`. You don't need to manually run `cert_create` and `fiptool` separately — the build system handles it all.

The catch: you lose StandaloneMM (secure variable storage). UEFI variables won't persist across reboots. But the boot works.

### The ROT key

TF-A ships a development RSA key for exactly this purpose:
```
plat/arm/board/common/rotpk/arm_rotprivk_rsa.pem
```

**Never use this in production.** For FVP development, it's perfect.

---

## Chapter 6: UEFI to GRUB to Kernel — Finally!

### The breakthrough moment

With the new FIP (PCID-fixed SCP + power-gate-bypassed model + no-SPM TF-A + fixed UEFI):

```
NOTICE:  BL1: Booting BL2
NOTICE:  BL2: Booting BL31
NOTICE:  BL31: Preparing for EL3 exit to normal world
...
[UEFI] Loading DXE drivers...
[UEFI] PlatformDxe loaded (no IoVirt UART hang!)
[UEFI] BDS: Booting GRUB
...
                    GNU GRUB  version 2.06

 RD-N2 Buildroot
```

GRUB loaded! It found the kernel on the virtio block device!

```
EFI stub: Booting Linux Kernel...
EFI stub: EFI_RNG_PROTOCOL unavailable
EFI stub: Using DTB from configuration table
EFI stub: Exiting boot services...
```

The Linux kernel was running. The EFI stub executed successfully, used the ACPI tables provided by UEFI, and exited boot services.

And then... nothing.

*(But that's a [different debugging story]({{< relref "debugging-kernel-hang-arm-fvp-sve-trap" >}}).)*

---

## Chapter 7: The TZC-400 Bypass

One critical piece of the puzzle that makes everything work: **TrustZone Controller bypass**.

The RD-N2 has 8 TZC-400 instances protecting different memory regions. By default, they block all non-secure access — which means the application cores (running non-secure Linux) can't access DRAM.

On real hardware, TF-A configures the TZC during BL2. But on the FVP, we can pre-configure them via model parameters:

```powershell
# For each TZC (0 through 7):
"-C", "css.tzc0.tzc400.rst_gate_keeper=0x0f",
"-C", "css.tzc0.tzc400.rst_region_attributes_0=0xc000000f",
"-C", "css.tzc0.tzc400.rst_region_id_access_0=0xffffffff",
```

What these mean:
- `rst_gate_keeper=0x0f` — All 4 filters enabled at reset
- `rst_region_attributes_0=0xc000000f` — Region 0 covers full address space, secure+non-secure read/write
- `rst_region_id_access_0=0xffffffff` — All IDs (masters) can access

This effectively makes all of DRAM accessible from both secure and non-secure worlds. Essential for FVP development — you'd never do this on real hardware.

---

## Chapter 8: UART Configuration — The Invisible Parameters

Getting UART output from the FVP requires several non-obvious parameters:

```powershell
# SCP UART — must explicitly enable
"-C", "css.scp.pl011_uart_scp.uart_enable=true",
"-C", "css.scp.pl011_uart_scp.out_file=$IMGDIR\uart_scp.log",
"-C", "css.scp.pl011_uart_scp.unbuffered_output=1",

# NS UART (Linux console) — flow control and DC4 critical
"-C", "css.pl011_ns_uart_ap.out_file=$IMGDIR\uart_nsec.log",
"-C", "css.pl011_ns_uart_ap.unbuffered_output=1",
"-C", "css.pl011_ns_uart_ap.flow_ctrl_mask_en=1",
"-C", "css.pl011_ns_uart_ap.enable_dc4=1",
```

### Why each parameter matters

| Parameter | What happens without it |
|-----------|------------------------|
| `uart_enable=true` | SCP UART produces no output at all |
| `unbuffered_output=1` | Output appears only after model shutdown |
| `flow_ctrl_mask_en=1` | UART stops transmitting (CTS held low) |
| `enable_dc4=1` | DC4 character (0x14) in data stream pauses output indefinitely |
| `out_file=...` | No file logging; must use telnet only |

The `flow_ctrl_mask_en=1` is especially insidious — without it, the PL011 model respects hardware flow control, and since nothing drives CTS high, the UART just... never transmits. Zero error indication.

---

## Chapter 9: The GRUB Disk Image

### Image structure

The `grub-buildroot.img` is a GPT disk with two partitions:

```
Partition 1: 20MB FAT32 (offset 1048576 bytes)
  ├── EFI/BOOT/BOOTAA64.EFI    (GRUB EFI binary)
  └── grub/grub.cfg             (boot configuration)

Partition 2: 200MB ext4 (offset 22020096 bytes)
  ├── Image                     (Linux kernel, 39MB)
  └── rootfs content            (buildroot filesystem)
```

### Editing grub.cfg

The cfg is inside a FAT partition inside a disk image. To edit it:

```bash
# Mount partition 1 using offset
mount -o offset=1048576,sizelimit=20971520 grub-buildroot.img /tmp/grubmnt
# Edit
vim /tmp/grubmnt/grub/grub.cfg
# Unmount
umount /tmp/grubmnt
```

### The final working grub.cfg

```
set default=0
set timeout=3

menuentry "RD-N2 Buildroot" {
    search --no-floppy --fs-uuid --set=root 535add81-5875-4b4a-b44a-464aee5f5cbd
    linux /Image acpi=force earlycon console=ttyAMA0,115200 loglevel=8 ignore_loglevel keep_bootcon nokaslr root=PARTUUID=9c53a91b-e182-4ff1-aeac-6ee2c432ae94 rootwait
}
```

### Why `acpi=force`?

Linux kernel 6.1.0 doesn't have a Device Tree Source for the RD-N2 platform. UEFI provides an ACPI SPCR table (Serial Port Console Redirection) that tells the kernel about the PL011 at 0x2A400000. Without `acpi=force`, the kernel might try to use a missing DTB and fail to find the console.

---

## Chapter 10: The Version Mismatch Problem

### The timeline gap

- **Firmware source:** RD-INFRA-2022.12.22 (December 2022)
- **FVP model:** Fast Models 11.25.23 (April 2024)

That's a **16-month gap**. During that time, ARM:
- Updated CMN-700 from r0p0 to r1p0 (changing PCID register values)
- Updated the power domain model behavior
- Changed PCIe hierarchy naming
- Modified UART flow control defaults

The firmware expected one version of the hardware; the model provided another. Every incompatibility manifested as a silent hang.

### Why ARM doesn't catch this

ARM's CI presumably tests each firmware release against a matching FVP version. When you download the latest FVP but use an older firmware tag (because the newer ones require authentication), you're in uncharted territory. There's no compatibility matrix published for the open-access releases.

---

## Chapter 11: The Complete Fix Summary

Here's every issue we encountered, in boot order:

### Issue 1: SCP SID PCID assertion (silent)
```
Symptom: Zero SCP UART output
Module: module/sid/src/mod_sid.c
Root cause: CMN-700 r1p0 PCID values ≠ hardcoded r0p0 values
Fix: Comment out `return FWK_E_DATA` in mod_pcid_check_registers()
Both ROM and RAM firmware affected
```

### Issue 2: SCP power domain gate (silent)
```
Symptom: SCP RAM firmware hangs during CMN-700 init
Warning: "Domain is power down, transaction will be aborted"
Root cause: FVP power domain model not tracking SCP's PPU writes
Fix: css.scp.powerStateGate.gate_behaviour=1 (and _cmn variant, and MCP variants)
```

### Issue 3: UEFI DXE hang (silent)
```
Symptom: Boot stuck after 13 DXE drivers, log stops growing
Function: InitIoVirtSocExpBlkUartControllers()
Root cause: PCD address = 0x0, FVP hangs on access to address 0
Fix: Set PcdIoVirtSocExpBlkUartEnable|0 in RdN2.dsc
Requires UEFI rebuild
```

### Issue 4: TBBR certificate mismatch
```
Symptom: "Failed to load image id 4 (-2)" or BL2 PANIC
Root cause: Modified UEFI binary doesn't match its certificate hash
Fix: Rebuild entire TF-A with GENERATE_COT=1 CREATE_KEYS=1
Must drop SPM_MM (StandaloneMM incompatible with fresh build)
Requires MBEDTLS_DIR and ARM_ROTPK_LOCATION=devel_rsa
```

### Issue 5: Missing MBEDTLS for TBBR
```
Symptom: TF-A build error about missing crypto functions
Root cause: TBBR needs mbedtls for certificate generation and verification
Fix: MBEDTLS_DIR=/root/rdn2/mbedtls (already present in repo sync)
```

---

## Chapter 12: The Working Launch Script

After all fixes, this is the complete PowerShell script that boots the RD-N2 FVP through the entire chain:

```powershell
## RD-N2 FVP Boot Script
$MODEL = "C:\Program Files\ARM\FVP_RD_N2\models\Win64_VC2019\FVP_RD_N2.exe"
$IMGDIR = "C:\Users\neerajpant\rdn2-images"

# Create NOR flash images (64MB each)
if (!(Test-Path "$IMGDIR\nor1_flash.img")) {
    $bytes = New-Object byte[] (64*1024*1024)
    [System.IO.File]::WriteAllBytes("$IMGDIR\nor1_flash.img", $bytes)
}
if (!(Test-Path "$IMGDIR\nor2_flash.img")) {
    $bytes = New-Object byte[] (64*1024*1024)
    [System.IO.File]::WriteAllBytes("$IMGDIR\nor2_flash.img", $bytes)
}

$PARAMS = @(
    "-I", "-p", "-R",   # Iris debug + run immediately

    # Firmware loading
    "--data", "css.scp.armcortexm7ct=$IMGDIR\scp_ramfw.bin@0x0BD80000",
    "--data", "css.mcp.armcortexm7ct=$IMGDIR\mcp_ramfw.bin@0x0BF80000",
    "-C", "css.mcp.ROMloader.fname=$IMGDIR\mcp_romfw.bin",
    "-C", "css.scp.ROMloader.fname=$IMGDIR\scp_romfw.bin",
    "-C", "css.trustedBootROMloader.fname=$IMGDIR\tf-bl1.bin",
    "-C", "board.flashloader0.fname=$IMGDIR\fip-uefi.bin",
    "-C", "board.flashloader1.fname=$IMGDIR\nor1_flash.img",
    "-C", "board.flashloader1.fnameWrite=$IMGDIR\nor1_flash.img",
    "-C", "board.flashloader2.fname=$IMGDIR\nor2_flash.img",
    "-C", "board.flashloader2.fnameWrite=$IMGDIR\nor2_flash.img",
    "-C", "board.virtioblockdevice.image_path=$IMGDIR\grub-buildroot.img",

    # UART configuration (all essential)
    "-C", "css.scp.pl011_uart_scp.out_file=$IMGDIR\uart_scp.log",
    "-C", "css.scp.pl011_uart_scp.unbuffered_output=1",
    "-C", "css.scp.pl011_uart_scp.uart_enable=true",
    "-C", "css.mcp.pl011_uart_mcp.out_file=$IMGDIR\uart_mcp.log",
    "-C", "css.mcp.pl011_uart_mcp.unbuffered_output=1",
    "-C", "css.pl011_ns_uart_ap.out_file=$IMGDIR\uart_nsec.log",
    "-C", "css.pl011_ns_uart_ap.unbuffered_output=1",
    "-C", "css.pl011_ns_uart_ap.flow_ctrl_mask_en=1",
    "-C", "css.pl011_ns_uart_ap.enable_dc4=1",
    "-C", "css.pl011_s_uart_ap.out_file=$IMGDIR\uart_sec.log",
    "-C", "css.pl011_s_uart_ap.unbuffered_output=1",

    # Power state gate bypass (SCP + MCP)
    "-C", "css.scp.powerStateGate.gate_behaviour=1",
    "-C", "css.scp.powerStateGate_cmn.gate_behaviour=1",
    "-C", "css.mcp.powerStateGate.gate_behaviour=1",
    "-C", "css.mcp.powerStateGate_cmn.gate_behaviour=1",

    # GIC + PCIe
    "-C", "css.gic_distributor.ITS-device-bits=20",
    "-C", "pcie_group_0.pciex16.hierarchy_file_name=<default>",
    "-C", "pcie_group_0.pciex16.pcie_rc.ahci0.endpoint.ats_supported=true",

    # TZC-400 bypass (all 8 instances)
    "-C", "css.tzc0.tzc400.rst_gate_keeper=0x0f",
    "-C", "css.tzc0.tzc400.rst_region_attributes_0=0xc000000f",
    "-C", "css.tzc0.tzc400.rst_region_id_access_0=0xffffffff"
    # ... repeat for tzc1 through tzc7
)

& $MODEL @PARAMS
```

---

## Chapter 13: Lessons for Firmware Bring-Up

### 1. Assume nothing about error reporting

Three of our five failures were **completely silent**. No error message, no crash dump, no bus fault indication. If your firmware has a failure path before UART initialization, you will never see it through normal means.

### 2. Version compatibility is never tested for your exact combination

ARM releases FVP models and firmware independently. The combinations they test internally may not include the one you downloaded. When mixing versions, expect at least one incompatibility that nobody has documented.

### 3. FVP model parameters are the hidden fourth firmware

Beyond SCP, TF-A, and UEFI, the FVP model itself needs correct configuration. Parameters like `powerStateGate.gate_behaviour`, `flow_ctrl_mask_en`, `enable_dc4`, and TZC bypass are **not optional** — they're part of the platform configuration that would be handled by hardware on real silicon.

### 4. TBBR is all-or-nothing

You cannot mix a TBBR-built BL1 with a non-TBBR FIP. You cannot swap one binary without regenerating its certificate. The certificate chain is brittle by design — that's the point of secure boot. When developing, use `GENERATE_COT=1 CREATE_KEYS=1` so TF-A handles everything in one build.

### 5. SPM_MM compatibility is fragile

The Secure Partition Manager (SPM_MM) and its associated BL32 (`mm_standalone.bin`) must be built in lockstep with TF-A. A `mm_standalone.bin` from one build configuration cannot be reused with a different TF-A build. When in doubt, drop SPM entirely for bring-up.

### 6. The SCP is the invisible kingmaker

Everything depends on the SCP. If it doesn't boot, nothing boots. If it doesn't power up the AP cores, TF-A never runs. If it doesn't configure CMN-700, memory access fails. And it can fail in ways that produce zero output because its UART initializes *after* the modules that might fail.

---

## Appendix A: File Inventory

### Working firmware set

| File | Size | Source |
|------|------|--------|
| `scp_romfw.bin` | 50,652 bytes | Rebuilt with PCID fix |
| `scp_ramfw.bin` | 128,052 bytes | Rebuilt with PCID fix |
| `mcp_romfw.bin` | 44,044 bytes | Pre-built from RD-INFRA package |
| `mcp_ramfw.bin` | 47,836 bytes | Pre-built from RD-INFRA package |
| `tf-bl1.bin` | 67,633 bytes | Rebuilt without SPM_MM, with SVE fix |
| `fip-uefi.bin` | 2,223,291 bytes | Contains BL2+BL31+UEFI+TBBR certs |
| `grub-buildroot.img` | 232,783,872 bytes | GRUB + Linux 6.1.0 + buildroot rootfs |
| `nor1_flash.img` | 67,108,864 bytes | Blank NOR flash (64MB zeros) |
| `nor2_flash.img` | 67,108,864 bytes | Blank NOR flash (64MB zeros) |

### Build environment

| Component | Version | Path |
|-----------|---------|------|
| FVP Model | 11.25.23 (Apr 2024) | `C:\Program Files\ARM\FVP_RD_N2\` |
| Source tag | RD-INFRA-2022.12.22 | `/root/rdn2/` |
| Cross-compiler | GCC 11.2 (ARM) | `/root/rdn2/tools/gcc/` |
| Embedded compiler | GCC 10.3 (apt) | `/usr/bin/arm-none-eabi-gcc` |
| Linux kernel | 6.1.0-g4810bccbf96e | Built May 18, 2026 |
| Host OS | Windows + WSL2 Ubuntu 22.04 | — |

---

## Appendix B: Telnet Ports

| Port | UART | Content |
|------|------|---------|
| 5000 | SCP | SCP firmware messages, module init, SCMI |
| 5001 | MCP | MCP firmware messages |
| 5002 | NS UART AP | **Linux console** — TF-A, UEFI, GRUB, kernel |
| 5003 | S UART AP | Secure world output (StandaloneMM) |
| 5004-5005 | IO Macro | PCIe/IO subsystem |
| 5006-5009 | SoC | Various SoC-level UARTs |

---

## Appendix C: The Boot Chain Explained

```
┌─────────────────────────────────────────────────────┐
│ PHASE 1: SCP Boot (Cortex-M7)                       │
│                                                     │
│ SCP ROM → loads SCP RAM from 0x0BD80000             │
│ SCP RAM → init CMN-700, power domains, SCMI         │
│         → powers up Application Processor cores     │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 2: Trusted Firmware-A (Cortex-N2 at EL3)      │
│                                                     │
│ BL1 (ROM) → authenticates + loads BL2               │
│ BL2 (RAM) → authenticates + loads BL31, BL33        │
│ BL31      → installs secure monitor at EL3          │
│           → sets up PSCI, SCMI proxy                │
│           → jumps to BL33 (UEFI) at EL2             │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 3: UEFI (EL2 → EL1)                          │
│                                                     │
│ DXE → loads drivers, enumerates devices             │
│ BDS → finds boot option (GRUB on virtio disk)       │
│     → loads GRUB EFI binary                         │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 4: GRUB + Linux                               │
│                                                     │
│ GRUB → reads grub.cfg, loads kernel Image           │
│ EFI stub → sets up memory map, gets ACPI tables     │
│          → calls ExitBootServices()                 │
│ Linux → start_kernel() → init → shell              │
└─────────────────────────────────────────────────────┘
```

---

## What Comes Next

With the firmware chain fully working, the next challenge was the kernel itself — it reached "Exiting boot services" but produced zero console output. That turned out to be an SVE register trap at EL3, which is [its own debugging story]({{< relref "debugging-kernel-hang-arm-fvp-sve-trap" >}}).

---

*Written from the trenches of ARM platform bring-up, May 2026.*
