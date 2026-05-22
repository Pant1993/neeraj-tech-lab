---
title: "Booting Linux on the ARM RD-N2 FVP — A Step-by-Step Guide"
date: 2026-05-22T14:00:00+05:30
draft: false
tags: ["ARM", "FVP", "Linux", "boot", "firmware", "SCP", "TF-A", "UEFI", "tutorial"]
categories: ["Tutorials"]
description: "A friendly step-by-step guide to booting Linux on the ARM Neoverse RD-N2 Fixed Virtual Platform — from installation through firmware build to successful boot."
cover:
  image: ""
  alt: "Boot Linux on ARM FVP"
ShowToc: true
TocOpen: true
weight: 2
---

<style>
.svg-diagram {
  margin: 1.5em 0;
  padding: 1em;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  overflow-x: auto;
}
.svg-diagram svg {
  width: 100%;
  height: auto;
  min-width: 600px;
}
</style>


> Arm RD-N2 FVP • Linux bring-up • Friendly walkthrough

If you’ve ever stared at a virtual platform, a pile of firmware binaries, and a terminal full of warnings and thought, *“okay… where do I even start?”* — you are absolutely in the right place. In this guide, we’ll go from a Windows machine to a booting Linux system on the Arm RD-N2 FVP together, one calm, practical step at a time.

- **16** — Neoverse N2 cores in the model
- **4** — Major stages in the boot chain
- **5002** — Telnet port for the Linux console

> 💡 **Tip:** how to install the FVP on Windows, set up a WSL2 Ubuntu build environment, fetch the exact source manifest, build the firmware + OS stack, prepare boot images, launch the RD-N2 model, and connect to the UART consoles without losing your mind.

## 1 Introduction

Let’s start with the obvious question: **what is an FVP?** FVP stands for **Fixed Virtual Platform**. Think of it as a detailed software model of a hardware platform. You get something that behaves like a real SoC platform — enough to boot firmware, UEFI, Linux, and user space — without needing the physical board on your desk.

For RD-N2, this is especially nice because you can explore the full server-style boot flow: SCP firmware, Trusted Firmware-A, UEFI, GRUB, Linux, and a root filesystem. That means you can debug platform bring-up, firmware interactions, and boot scripts long before real silicon (or before your hardware setup cooperates).

By the end of this guide, you’ll have a repeatable setup for booting Linux on the **Arm RD-N2 FVP**. And yes — we’ll also talk about the sharp edges, because there are a few, and pretending otherwise would be rude.

> 🎉 **Milestone:** if your goal is “I want a virtual RD-N2 platform that boots Linux and gives me a working console,” that is exactly the road we’re walking here.

## 2 Prerequisites

Here’s the setup we’re assuming. Nothing exotic — but getting these basics right saves a lot of frustration later.

### Windows host

Use Windows to install and run the FVP model itself. In this guide the model lives under `C:\Program Files\ARM\FVP_RD_N2\`.

### WSL2 + Ubuntu 22.04

We’ll do the source sync and builds inside WSL2, because that’s the path of least resistance for the Linux toolchain.

### Arm FVP installed

Specifically the Neoverse N2 Reference Design FVP from `developer.arm.com`.

- A Windows machine with enough disk space for the source tree and build outputs.
- WSL2 configured and running **Ubuntu 22.04**.
- Basic comfort with PowerShell and the Ubuntu shell.
- Internet access for downloading the model and syncing sources.

> 💡 **Tip:** keep your generated binaries in one predictable Windows folder. In this walkthrough we use `C:\Users\neerajpant\rdn2-images`. That makes the final FVP launch command much easier to read and maintain.

## 3 Installing the FVP

Head to **developer.arm.com** and download the **Neoverse N2 Reference Design FVP**. Install it on Windows at:

```text
C:\Program Files\ARM\FVP_RD_N2\
```

That gives you the model executable we’ll eventually launch from PowerShell. On a typical install, the RD-N2 model binary is here:

```text
C:\Program Files\ARM\FVP_RD_N2\models\Win64_VC2019\FVP_RD_N2.exe
```

The reason we install the FVP on Windows instead of inside WSL is simple: the Windows package is the native path for running the model, while WSL is better suited for building the Linux-side software stack.

> ⚠️ **Warning:** make sure the install path has no surprises. If you move the model elsewhere, update every launch script accordingly. Boot issues caused by a stale path are the kind that make you question your life choices for 20 minutes.

## 4 What’s inside the FVP

The RD-N2 FVP models a server-class platform. It is not trying to be a phone, tablet, or graphics-heavy board. In fact, one of the easiest things to misunderstand is this:

> 💡 **Tip:** RD-N2 is a server platform. That’s why the interesting parts are compute, coherency, interrupts, memory security, and platform control firmware — not graphics output.

### Main IP blocks

- **16× Neoverse N2 cores**
- **CMN-700** coherent mesh interconnect
- **GIC-700** interrupt controller
- **SMMUv3** for I/O address translation
- **TZC-400** trust zone controller
- **PCIe** for I/O
- **MHU** mailbox/message units
- **SCP** based on Cortex-M7

### Why this matters

When Linux finally boots, it didn’t get there by magic. The SCP initialized platform pieces, TF-A set up the secure world handoff, UEFI did platform bring-up, and then GRUB loaded the kernel and root filesystem. The model reflects that full chain.

<div class="svg-diagram">
<svg width="100%" viewBox="0 0 1060 470" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="RD-N2 FVP architecture overview">
<defs>
<linearGradient id="g1" x1="0" x2="1">
<stop offset="0%" stop-color="#2952ff"/>
<stop offset="100%" stop-color="#6e7dff"/>
</linearGradient>
</defs>
<rect x="20" y="20" width="940" height="430" rx="26" fill="#ffffff" stroke="#dbe3f1"/>
<text x="50" y="58" font-size="28" font-family="Segoe UI, Arial" fill="#1c2434" font-weight="700">RD-N2 FVP architecture overview</text>
<rect x="52" y="95" width="290" height="110" rx="18" fill="#edf2ff" stroke="#cfd9ff"/>
<text x="72" y="130" font-size="24" fill="#2952ff" font-weight="700">Compute complex</text>
<text x="72" y="165" font-size="20" fill="#334155">16× Neoverse N2 cores</text>
<text x="72" y="192" font-size="16" fill="#64748b">Clustered server-class CPU subsystem</text>
<rect x="365" y="95" width="250" height="110" rx="18" fill="#eefcf7" stroke="#c8efe2"/>
<text x="388" y="130" font-size="24" fill="#0f9d7a" font-weight="700">CMN-700</text>
<text x="388" y="165" font-size="20" fill="#334155">Coherent mesh fabric</text>
<text x="388" y="192" font-size="16" fill="#64748b">System coherency + memory routing</text>
<rect x="640" y="95" width="340" height="110" rx="18" fill="#fff7ed" stroke="#fed7aa"/>
<text x="662" y="130" font-size="22" fill="#d97706" font-weight="700">Platform I/O + security</text>
<text x="662" y="165" font-size="18" fill="#334155">GIC-700, SMMUv3, TZC-400, PCIe</text>
<text x="662" y="192" font-size="16" fill="#64748b">Interrupts, isolation, DMA translation</text>
<rect x="90" y="270" width="350" height="110" rx="18" fill="#f8fafc" stroke="#dbe3f1"/>
<text x="116" y="306" font-size="24" fill="#1c2434" font-weight="700">Control subsystem</text>
<text x="116" y="338" font-size="20" fill="#334155">SCP + MCP (Cortex-M7)</text>
<text x="116" y="364" font-size="16" fill="#64748b">Power, clocks, topology, system coordination</text>
<rect x="490" y="270" width="420" height="110" rx="18" fill="#f5f3ff" stroke="#ddd6fe"/>
<text x="515" y="306" font-size="22" fill="#6d28d9" font-weight="700">Boot-facing storage + comms</text>
<text x="515" y="338" font-size="18" fill="#334155">NOR flash, virtio block, MHU mailboxes</text>
<text x="515" y="364" font-size="16" fill="#64748b">Firmware images, boot media, AP↔SCP signaling</text>
<path d="M342 150 H365" stroke="url(#g1)" stroke-width="6" stroke-linecap="round"/>
<path d="M615 150 H640" stroke="url(#g1)" stroke-width="6" stroke-linecap="round"/>
<path d="M490 325 H440" stroke="#94a3b8" stroke-width="6" stroke-linecap="round"/>
<path d="M615 205 V250 H677" stroke="#94a3b8" stroke-width="6" fill="none" stroke-linecap="round"/>
<path d="M230 205 V270" stroke="#94a3b8" stroke-width="6" fill="none" stroke-linecap="round"/>
</svg>
</div>

## 5 Setting up the build environment

Now we switch to **WSL2 Ubuntu 22.04**. This is where we’ll install dependencies and build the software stack. The package list is a little long, but it’s the honest version of events — and I’d rather give you the full list than let you discover missing tools one by one during a 40-minute build.

```bash
sudo apt update
sudo apt install -y \
  build-essential git python3 cmake libfdt-dev unzip autoconf automake \
  autopoint libtool mtools rsync cpio bc wget file device-tree-compiler \
  acpica-tools uuid-dev python3-pyelftools
```

You’ll also want Google’s `repo` tool, because the RD-N2 software stack is pulled through a manifest rather than a single Git repository.

```bash
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo -o ~/bin/repo
chmod a+x ~/bin/repo
export PATH="$HOME/bin:$PATH"
```

If you want that path to persist, add the export line to your `~/.bashrc`. Future-you will appreciate it.

## 6 Getting the source code

We’re going to use `repo` with the manifest tag **RD-INFRA-2022.12.22** and the manifest file **pinned-rdn2.xml**. That combination gives you a known-good baseline of repositories that work together.

```bash
mkdir -p ~/rdn2
cd ~/rdn2
repo init -u https://git.gitlab.arm.com/arm-reference-solutions/manifest.git \
  -b RD-INFRA-2022.12.22 \
  -m pinned-rdn2.xml
repo sync -j$(nproc)
```

So what actually comes down? Not just one project. You’re syncing a whole platform stack, typically including:

- `SCP firmware`
- `Trusted Firmware-A`
- `UEFI / EDK2`
- `Linux kernel`
- `GRUB`
- `buildroot`

This is one of those moments where the architecture starts to click: we’re not “just building Linux.” We’re building the *entire boot story* that gets Linux onto the platform.

> 🎉 **Milestone:** once `repo sync` completes, you have the raw ingredients. The kitchen is messy, but the ingredients are there. That counts.

## 7 Building the stack

The exact top-level build command can vary with the reference solution layout you synced, but the important thing is understanding **what gets produced**. Here’s the stack we care about and the files we expect to end up with.

| Component | What it does | Output files |
| --- | --- | --- |
| SCP firmware | Boots first on the Cortex-M7 control processor, initializes platform resources, powers up application cores. | `scp_romfw.bin`, `scp_ramfw.bin`, `mcp_romfw.bin`, `mcp_ramfw.bin` |
| Trusted Firmware-A | Secure boot chain and early EL3 runtime for the application processors. | `tf-bl1.bin` |
| FIP image | Combined firmware package that carries BL2, BL31, and UEFI for the AP boot path. | `fip-uefi.bin` |
| Linux + buildroot | Kernel plus a practical root filesystem to get to a shell quickly. | Kernel, DTB, buildroot filesystem artifacts |
| GRUB + disk image | Bootable disk image that hands control to Linux. | `grub-buildroot.img` |

<div class="svg-diagram">
<svg width="100%" viewBox="0 0 1100 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Boot flow diagram">
<rect x="20" y="35" width="170" height="92" rx="18" fill="#eef2ff" stroke="#c7d2fe"/>
<text x="52" y="72" font-size="24" fill="#2952ff" font-weight="700">SCP firmware</text>
<text x="52" y="102" font-size="18" fill="#475569">Bring-up + power-on</text>
<rect x="230" y="35" width="160" height="92" rx="18" fill="#f8fafc" stroke="#dbe3f1"/>
<text x="275" y="72" font-size="24" fill="#0f172a" font-weight="700">TF-A</text>
<text x="255" y="102" font-size="18" fill="#475569">BL1 → BL2 → BL31</text>
<rect x="430" y="35" width="170" height="92" rx="18" fill="#eefcf7" stroke="#bbf7d0"/>
<text x="484" y="72" font-size="24" fill="#0f9d7a" font-weight="700">UEFI</text>
<text x="468" y="102" font-size="18" fill="#475569">DXE + boot manager</text>
<rect x="640" y="35" width="160" height="92" rx="18" fill="#fff7ed" stroke="#fed7aa"/>
<text x="690" y="72" font-size="24" fill="#d97706" font-weight="700">GRUB</text>
<text x="673" y="102" font-size="18" fill="#475569">Loads kernel + cmdline</text>
<rect x="840" y="35" width="220" height="92" rx="18" fill="#f5f3ff" stroke="#ddd6fe"/>
<text x="910" y="72" font-size="24" fill="#6d28d9" font-weight="700">Linux</text>
<text x="876" y="102" font-size="16" fill="#475569">Kernel + buildroot userspace</text>
<path d="M190 81 H230" stroke="#6e7dff" stroke-width="6" stroke-linecap="round"/>
<path d="M390 81 H430" stroke="#6e7dff" stroke-width="6" stroke-linecap="round"/>
<path d="M600 81 H640" stroke="#6e7dff" stroke-width="6" stroke-linecap="round"/>
<path d="M800 81 H840" stroke="#6e7dff" stroke-width="6" stroke-linecap="round"/>
<text x="20" y="180" font-size="20" fill="#334155">Firmware images used by the FVP:</text>
<text x="20" y="214" font-size="15" fill="#64748b">scp_romfw.bin / scp_ramfw.bin • mcp_romfw.bin / mcp_ramfw.bin • tf-bl1.bin • fip-uefi.bin • grub-buildroot.img</text>
</svg>
</div>

A very practical habit here: once the build completes, copy the final artifacts into your Windows image directory so the model launch path stays clean and deterministic.

## 8 Common build issues & fixes

This is the “let’s save each other some time” section. A clean build is wonderful when it happens. But if it doesn’t, these are the issues most likely to get in your way.

### 8.1 kvmtool libfdt issue

One annoying failure shows up around `kvmtool` and `libfdt`. The workaround is to patch the Makefile so the build stops assuming the host-side layout and instead uses the correct cross-compiled paths.

Patch the Makefile around line 345 so that:

```makefile
ifneq (y,y)
```

Then update lines 348–351 to point at the cross-compiled `libfdt` include/library paths used by your build. The exact paths depend on where your build tree places them, but the intention is always the same: stop the build from resolving the wrong `libfdt`.

### 8.2 Missing package errors

If the build stops with “command not found” or missing header/library errors, go back to the package list in the build environment section and install the missing dependency explicitly. The most common misses are:

- `device-tree-compiler`
- `acpica-tools`
- `uuid-dev`
- `python3-pyelftools`
- `libfdt-dev`

> ⚠️ **Warning:** the build can get surprisingly far before a missing package finally breaks something. So if a late step fails, it does not automatically mean your earlier steps were wrong.

### 8.3 SCP bus fault with old firmware vs new FVP

This one is sneaky. If you use older SCP firmware with a newer FVP build, you can hit an SCP bus fault early in the boot process. The fix is usually to rebuild the SCP firmware from the latest master and drop in the fresh binary.

Use these commands exactly:

```bash
cd /root/scp-new
git clone https://github.com/ARM-software/SCP-firmware.git .
git submodule update --init contrib/cmsis/git
cmake -B build-scp/scp_ramfw -DSCP_FIRMWARE_SOURCE_DIR:PATH=/root/scp-new/product/neoverse-rd/rdn2/scp_ramfw -DSCP_PLATFORM_VARIANT=0 -DCMAKE_BUILD_TYPE=Release
cmake --build build-scp/scp_ramfw -j$(nproc)
```

After that, replace the SCP RAM firmware in your FVP image directory with the newly built one. If your failure was caused by the firmware/model mismatch, this is often the moment where the whole thing suddenly behaves better and you get to enjoy a tiny, well-earned victory.

> 🎉 **Milestone:** getting past firmware mismatch issues is real bring-up work. It may not feel glamorous, but it’s exactly the kind of problem-solving that makes the platform finally come alive.

## 9 Preparing boot images

Before launching the model, create two NOR flash images — each **64 MB**. If they don’t already exist, PowerShell can create them as zero-filled files:

```powershell
$IMGDIR = "C:\Users\neerajpant\rdn2-images"

if (!(Test-Path "$IMGDIR\nor1_flash.img")) {
    $bytes = New-Object byte[] (64*1024*1024)
    [System.IO.File]::WriteAllBytes("$IMGDIR\nor1_flash.img", $bytes)
}

if (!(Test-Path "$IMGDIR\nor2_flash.img")) {
    $bytes = New-Object byte[] (64*1024*1024)
    [System.IO.File]::WriteAllBytes("$IMGDIR\nor2_flash.img", $bytes)
}
```

Then copy your key binaries from WSL2/build outputs into your Windows image folder:

- `scp_romfw.bin`
- `scp_ramfw.bin`
- `mcp_romfw.bin`
- `mcp_ramfw.bin`
- `tf-bl1.bin`
- `fip-uefi.bin`
- `grub-buildroot.img`

The idea is simple: by the time you launch the FVP, every image it needs is already sitting in one place. No last-minute path archaeology.

## 10 Launching the FVP

Alright — the fun part. Here is the full PowerShell launch command pattern. This is the moment where all the build artifacts finally get threaded into the model.

```powershell
$MODEL = "C:\Program Files\ARM\FVP_RD_N2\models\Win64_VC2019\FVP_RD_N2.exe"
$IMGDIR = "C:\Users\neerajpant\rdn2-images"

& $MODEL \
  --data "css.scp.armcortexm7ct=$IMGDIR\scp_ramfw.bin@0x0BD80000" \
  --data "css.mcp.armcortexm7ct=$IMGDIR\mcp_ramfw.bin@0x0BF80000" \
  -C "css.scp.ROMloader.fname=$IMGDIR\scp_romfw.bin" \
  -C "css.mcp.ROMloader.fname=$IMGDIR\mcp_romfw.bin" \
  -C "css.trustedBootROMloader.fname=$IMGDIR\tf-bl1.bin" \
  -C "board.flashloader0.fname=$IMGDIR\fip-uefi.bin" \
  -C "board.flashloader1.fname=$IMGDIR\nor1_flash.img" \
  -C "board.flashloader1.fnameWrite=$IMGDIR\nor1_flash.img" \
  -C "board.flashloader2.fname=$IMGDIR\nor2_flash.img" \
  -C "board.flashloader2.fnameWrite=$IMGDIR\nor2_flash.img" \
  -C "board.virtioblockdevice.image_path=$IMGDIR\grub-buildroot.img" \
  -C "css.gic_distributor.ITS-device-bits=20" \
  -C "css.tzc0.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc0.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc0.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc1.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc1.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc1.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc2.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc2.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc2.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc3.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc3.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc3.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc4.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc4.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc4.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc5.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc5.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc5.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc6.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc6.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc6.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "css.tzc7.tzc400.rst_gate_keeper=0x0f" \
  -C "css.tzc7.tzc400.rst_region_attributes_0=0xc000000f" \
  -C "css.tzc7.tzc400.rst_region_id_access_0=0xffffffff" \
  -C "disable_visualisation=true" \
  -R
```

A few important notes about that command:

- `--data css.scp.armcortexm7ct=...@0x0BD80000` places the SCP RAM firmware at the address the model expects.
- `-C css.trustedBootROMloader.fname=tf-bl1.bin` points BL1 at the Trusted Firmware-A entry point.
- `-C board.flashloader0.fname=fip-uefi.bin` loads the FIP image containing **BL2 + BL31 + UEFI**.
- `-C board.virtioblockdevice.image_path=grub-buildroot.img` gives the platform a bootable disk image for GRUB and the root filesystem.
- The TZC settings bypass restrictive default protection so the software stack can boot cleanly in this setup.
- `-R` means **run from reset**.
- `-C disable_visualisation=true` keeps the model headless, which is exactly what we want for a server-style workflow.

> 💡 **Tip:** UARTs are typically exposed on telnet ports **5000** (SCP), **5001** (MCP), **5002** (non-secure / Linux), and **5003** (secure world output).

## 11 Connecting via telnet

Once the FVP is running, the UARTs are your window into what the platform is doing. If Telnet Client is not enabled on Windows yet, turn it on first:

```powershell
dism /online /Enable-Feature /FeatureName:TelnetClient
```

Then open separate terminals and connect to the interesting UARTs:

```bash
telnet localhost 5000   # SCP console
telnet localhost 5002   # Linux / non-secure AP console
```

If you’re curious, you can also watch the MCP and secure UARTs:

```bash
telnet localhost 5001   # MCP console
telnet localhost 5003   # Secure console
```

The SCP console is wonderful when you’re debugging early bring-up. The Linux console is where the reward shows up — kernel boot logs, init, and hopefully a shell prompt.

## 12 What you’ll see during boot

This is the part where the abstract architecture becomes a visible sequence of messages. Here’s the rough flow you should expect:

1. **SCP firmware starts first** on the Cortex-M7 and initializes platform services.
2. It configures pieces like **CMN-700** and powers up the application processor cores.
3. **TF-A** takes over on the AP side: **BL1 → BL2 → BL31**.
4. **StandaloneMM / UEFI** comes up and begins its DXE phase.
5. **GRUB** loads the Linux kernel and root filesystem from the virtio block image.
6. **Linux boots**, starts init, and lands you in buildroot userspace.

> ⚠️ **Warning:** the FVP is **very slow** compared to real hardware — think on the order of **~100× slower**. If you are used to a board booting in seconds, the model can feel like it’s moving through syrup. That does not necessarily mean it’s stuck.

So if the platform seems to pause, give it a little grace before assuming disaster. Many bring-up sessions have been derailed by impatience more than by actual bugs.

## 13 Troubleshooting

Here’s a quick practical checklist for the most common “it doesn’t boot and I’m tired” situations.

| Symptom | Likely cause | What to try |
| --- | --- | --- |
| No output on any UART | Wrong model path, failed launch, or console ports not exposed | Re-run the PowerShell command, verify `FVP_RD_N2.exe` exists, and confirm telnet connections to `localhost:5000-5003`. |
| SCP dies early with a bus fault | Old SCP firmware with a newer FVP build | Rebuild SCP from the latest master and replace `scp_ramfw.bin`. |
| Build fails on missing commands or headers | Dependency gap in Ubuntu | Install the missing packages from the prerequisites section and retry the failed step. |
| UEFI appears, but Linux never boots | Broken disk image, GRUB issue, or missing kernel/rootfs content | Rebuild or replace `grub-buildroot.img` and verify the virtio block path in the FVP command. |
| Weird permission/flash-related failures | NOR images missing or not writable | Make sure both 64 MB NOR images exist and that `fnameWrite` points to the same files. |
| Platform appears frozen | It may just be slow | Wait longer than feels reasonable. Seriously. FVP performance can make a healthy boot look suspiciously dramatic. |

> 💡 **Tip:** watch both the SCP console and the Linux console. If Linux never wakes up, the SCP side usually tells you why the AP world never got a clean start.

## 14 Conclusion

If you’ve made it this far, you’ve done something genuinely substantial: you set up a mixed Windows + WSL2 workflow, installed the RD-N2 FVP, synced a pinned platform manifest, built a multi-stage firmware and OS stack, prepared the flash and disk images, launched the model, and connected to the boot consoles.

That’s not “just booting Linux.” That’s understanding a modern Arm platform boot flow end to end.

> 🎉 **Milestone:** you now have a repeatable RD-N2 virtual bring-up path. That means you can move on to kernel changes, firmware experiments, boot parameter tweaks, device-tree work, UEFI investigations, or plain old platform exploration with a lot more confidence.

Good next steps from here:

- Customize the kernel command line and GRUB config.
- Swap in your own kernel build or root filesystem.
- Experiment with TF-A or UEFI changes.
- Trace power-up and mailbox interactions through the SCP logs.
- Document a one-click launch script once your flow stabilizes.

And honestly? Take a second to enjoy the moment when the Linux console finally comes alive. Booting a platform stack like this is part engineering, part archaeology, and part stubbornness. If you got it working, you earned that smile.
