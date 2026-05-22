---
title: "ARM RD-N2 SoC Architecture — A Friendly Deep Dive"
date: 2026-05-22T15:00:00+05:30
draft: false
tags: ["ARM", "Neoverse", "SoC", "CMN-700", "SCP", "firmware", "power-management", "DVFS", "PLL"]
categories: ["Architecture"]
description: "A comprehensive deep-dive into the ARM RD-N2 reference design SoC — covering the SCP, CMN-700 mesh, bus hierarchy, MHU, SCMI, power domains, PLL, and DVFS — explained like a friend over coffee."
cover:
  image: ""
  alt: "ARM RD-N2 Architecture"
ShowToc: true
TocOpen: true
weight: 1
---

<style>
:root {
  --svg-grid: #cbd5e1;
  --svg-text: #1e293b;
}
.dark {
  --svg-grid: #475569;
  --svg-text: #e2e8f0;
}
.svg-diagram {
  margin: 1.5em 0;
  padding: 1em;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  overflow-x: auto;
}
.dark .svg-diagram {
  background: #1e293b;
  border-color: #334155;
}
.svg-diagram svg {
  width: 100%;
  height: auto;
  min-width: 600px;
}
</style>

*Architecture deep dive*

# ARM RD-N2 SoC Architecture, Explained Like We’re Chatting Over Coffee

Let’s unpack the ARM RD-N2 platform the friendly way: what it is, why it exists, how the System Control Processor bossily wakes up first, why the CMN-700 mesh is the real traffic cop of the party, and how boot, power, clocks, coherency, SCMI, MHU, TZC, GIC, and SMMU all fit together without anyone flipping the table.

*Deep-dive article · Conversational tone · Inline SVG diagrams · Responsive modern HTML*

- **16** — Neoverse N2 cores
- **8** — TZC-400 instances / DRAM channels
- **1** — SCP that boots first
- **CHI** — Coherent fabric protocol at the top

## 1. Introduction — So, What Exactly Is RD-N2?

Think of **RD-N2** as ARM’s reference design for a serious server-class system-on-chip built around **Neoverse N2** CPUs. It is not just “some chip.” It is closer to a cookbook, blueprint, and demo home all rolled together: a platform that shows partners how to build a real infrastructure SoC using ARM IP blocks that are meant to work together at scale.

If you’ve ever looked at a modern server SoC and thought, “Okay, but how do all these blocks actually cooperate without chaos?”, RD-N2 is one of the cleanest answers. It exists so silicon vendors, platform teams, firmware teams, and software folks can start from a proven architecture instead of improvising the digital equivalent of assembling IKEA furniture without the manual.

Who uses it? Usually:

- **SoC architects** evaluating top-level partitioning and IP choices.
- **Firmware engineers** working on SCP, TF-A, UEFI, SCMI, and boot policy.
- **Kernel and platform teams** bringing up Linux and validating interrupts, memory, and I/O.
- **Partners building infrastructure chips** who want a practical starting point for a coherent ARM server platform.

> ✅ **Key insight:** RD-N2 is interesting because it shows the *whole stack*: compute, coherency, security boundaries,
> power control, boot flow, message passing, and software handoff. In other words: not just the
> engine, but also the ignition, dashboard, brakes, and cup holders.

## 2. SoC Overview — The Cast of Characters

At the top level, the RD-N2 SoC brings together a pretty muscular set of building blocks:

- **16 × Neoverse N2 cores** for the application-processing side.
- **CMN-700 mesh** as the coherent interconnect backbone.
- **GIC-700** as the interrupt controller.
- **8 × TZC-400**, one per DRAM channel, to police secure vs non-secure access.
- **SMMUv3** to translate I/O addresses and keep DMA civilized.
- **PCIe and I/O macros** for external devices and acceleration paths.
- **MHU** to provide tiny but useful doorbell-style messaging between domains.
- **SCP based on Cortex-M7** to handle power, clocks, system policy, and early bring-up.

| Block | What it does | Why you care |
| --- | --- | --- |
| Neoverse N2 cores | Run the OS and workloads | The muscle |
| CMN-700 | Maintains coherency and routes traffic | The traffic system |
| SCP (Cortex-M7) | Manages power, clocks, policies, boot prep | The stage manager |
| GIC-700 | Interrupt distribution and prioritization | The alarm dispatcher |
| TZC-400 | TrustZone memory protection per DRAM path | The bouncer |
| SMMUv3 | I/O VA→PA translation and isolation | The customs checkpoint |
| MHU + shared memory | AP↔SCP signaling and message transport | The buzzer + whiteboard combo |

*Inline SVG: RD-N2 SoC Block Diagram*
<div class="svg-diagram">
<svg viewBox="0 0 1100 640" role="img" aria-label="RD-N2 SoC block diagram">
<defs>
<linearGradient id="socGrad" x1="0" x2="1">
<stop offset="0%" stop-color="#5b7cfa" />
<stop offset="100%" stop-color="#20c997" />
</linearGradient>
</defs>
<rect x="10" y="10" width="1080" height="620" rx="24" fill="none" stroke="var(--svg-grid)" stroke-width="2" />
<rect x="250" y="180" width="600" height="240" rx="22" fill="url(#socGrad)" opacity="0.15" stroke="#5b7cfa" stroke-width="3"/>
<text x="550" y="214" text-anchor="middle" fill="var(--svg-text)" font-size="24" font-weight="800">CMN-700 Coherent Mesh</text>
<g fill="#8ea5ff" stroke="#3759d7" stroke-width="2">
<rect x="290" y="245" width="110" height="56" rx="12"/>
<rect x="420" y="245" width="110" height="56" rx="12"/>
<rect x="550" y="245" width="110" height="56" rx="12"/>
<rect x="680" y="245" width="110" height="56" rx="12"/>
</g>
<g fill="#b2f5ea" stroke="#0f9f7c" stroke-width="2">
<rect x="290" y="320" width="110" height="56" rx="12"/>
<rect x="420" y="320" width="110" height="56" rx="12"/>
<rect x="550" y="320" width="110" height="56" rx="12"/>
<rect x="680" y="320" width="110" height="56" rx="12"/>
</g>
<g fill="var(--svg-text)" font-size="16" font-weight="700">
<text x="345" y="278" text-anchor="middle">Cluster 0</text>
<text x="475" y="278" text-anchor="middle">Cluster 1</text>
<text x="605" y="278" text-anchor="middle">Cluster 2</text>
<text x="735" y="278" text-anchor="middle">Cluster 3</text>
<text x="345" y="352" text-anchor="middle">4× N2 cores</text>
<text x="475" y="352" text-anchor="middle">4× N2 cores</text>
<text x="605" y="352" text-anchor="middle">4× N2 cores</text>
<text x="735" y="352" text-anchor="middle">4× N2 cores</text>
</g>
<g>
<rect x="40" y="90" width="170" height="90" rx="16" fill="#ffd166" stroke="#c98900" stroke-width="2"/>
<text x="125" y="126" text-anchor="middle" fill="#422800" font-size="20" font-weight="800">SCP</text>
<text x="125" y="152" text-anchor="middle" fill="#422800" font-size="15">Cortex-M7</text>
<rect x="40" y="220" width="170" height="90" rx="16" fill="#ff9b9b" stroke="#d9485f" stroke-width="2"/>
<text x="125" y="255" text-anchor="middle" fill="#3f1111" font-size="20" font-weight="800">GIC-700</text>
<text x="125" y="279" text-anchor="middle" fill="#3f1111" font-size="15">Interrupt fabric</text>
<rect x="40" y="350" width="170" height="90" rx="16" fill="#a7f3d0" stroke="#0f9f7c" stroke-width="2"/>
<text x="125" y="385" text-anchor="middle" fill="#0a2b21" font-size="20" font-weight="800">MHU + SRAM</text>
<text x="125" y="409" text-anchor="middle" fill="#0a2b21" font-size="15">SCMI transport</text>
</g>
<g>
<rect x="890" y="95" width="160" height="90" rx="16" fill="#c4b5fd" stroke="#7c3aed" stroke-width="2"/>
<text x="970" y="130" text-anchor="middle" fill="#29124d" font-size="20" font-weight="800">SMMUv3</text>
<text x="970" y="154" text-anchor="middle" fill="#29124d" font-size="15">I/O translation</text>
<rect x="890" y="225" width="160" height="90" rx="16" fill="#fdba74" stroke="#ea580c" stroke-width="2"/>
<text x="970" y="260" text-anchor="middle" fill="#4a2207" font-size="20" font-weight="800">PCIe / IO</text>
<text x="970" y="284" text-anchor="middle" fill="#4a2207" font-size="15">RN-I / RN-D traffic</text>
<rect x="890" y="355" width="160" height="145" rx="16" fill="#93c5fd" stroke="#2563eb" stroke-width="2"/>
<text x="970" y="388" text-anchor="middle" fill="#10203b" font-size="20" font-weight="800">8 × TZC-400</text>
<text x="970" y="414" text-anchor="middle" fill="#10203b" font-size="15">Per DRAM channel</text>
<text x="970" y="438" text-anchor="middle" fill="#10203b" font-size="15">Protect Secure regions</text>
<text x="970" y="462" text-anchor="middle" fill="#10203b" font-size="15">Feeds memory system</text>
</g>
<g stroke="#5a6e8b" stroke-width="3" fill="none" marker-end="url(#arrowSoc)"></g>
<defs>
<marker id="arrowSoc" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M 0 0 L 10 5 L 0 10 z" fill="#5a6e8b"/>
</marker>
</defs>
<line x1="210" y1="135" x2="250" y2="210" stroke="#5a6e8b" stroke-width="3" marker-end="url(#arrowSoc)"/>
<line x1="210" y1="265" x2="250" y2="265" stroke="#5a6e8b" stroke-width="3" marker-end="url(#arrowSoc)"/>
<line x1="210" y1="395" x2="250" y2="350" stroke="#5a6e8b" stroke-width="3" marker-end="url(#arrowSoc)"/>
<line x1="850" y1="210" x2="890" y2="140" stroke="#5a6e8b" stroke-width="3" marker-end="url(#arrowSoc)"/>
<line x1="850" y1="270" x2="890" y2="270" stroke="#5a6e8b" stroke-width="3" marker-end="url(#arrowSoc)"/>
<line x1="850" y1="330" x2="890" y2="420" stroke="#5a6e8b" stroke-width="3" marker-end="url(#arrowSoc)"/>
</svg>
</div>

The overall feel is this: the application processors do the heavy compute, the mesh keeps their memory view coherent, the SCP sets the rules of the house, and the rest of the platform makes sure data, power, interrupts, and security boundaries behave themselves.

## 3. The SCP (System Control Processor) — The Tiny Adult in the Room

The **SCP** in RD-N2 is a **Cortex-M7**-based management processor. And yes, it boots *before* the application cores. That’s not a quirky design choice; it’s the whole point. Before the big Neoverse cores can do anything useful, somebody needs to set up clocks, power rails, reset sequencing, and interconnect configuration. That “somebody” is the SCP.

If the N2 cores are the athletes, the SCP is the coach who unlocks the gym, turns on the lights, checks the schedule, and only then lets everybody in. Without it, the AP side is basically a very expensive sculpture.

### Why SCP goes first

- Initializes low-level system state.
- Configures parts of the **CMN-700** through control interfaces such as **APB**.
- Brings up power domains and clock sources.
- Releases application cores from reset in a controlled way.
- Continues to act as the management endpoint for SCMI once Linux is running.

### Modular firmware structure

One lovely thing about SCP firmware is that it is intentionally modular. The firmware is usually organized as a **framework plus modules**. The framework handles scheduling, services, and event flow; the modules implement platform-specific things like clocks, sensors, power domain drivers, SCMI services, MHU transport, and board/product wiring.

For RD-N2 specifically, the product-level code organization lives under:

```
product/neoverse-rd/rdn2/
```

That is where the product description, power domain layout, clock configuration, transport wiring, and platform-flavored SCP setup typically come together.

> 💡 **Note:** The elegance here is subtle: the SCP firmware is not a monolithic blob of “do everything.” It is
> more like a small operating environment for system management, with reusable modules plugged into a
> product-specific description. That keeps the design scalable across different reference platforms.

## 4. Boot Flow — The Relay Race from Reset to Linux

The RD-N2 boot path is a staged handoff. Picture a relay team where every runner has exactly one job and hopefully doesn’t drop the baton. The sequence is:

**SCP ROM → SCP RAM → power up AP → TF-A BL1 → BL2 → BL31 + StandaloneMM → UEFI → GRUB → Linux**

*Inline SVG: Boot Flow*
<div class="svg-diagram">
<svg viewBox="0 0 940 1080" role="img" aria-label="RD-N2 boot flow stages">
<defs>
<linearGradient id="bootGrad" x1="0" x2="1">
<stop offset="0%" stop-color="#5b7cfa" />
<stop offset="100%" stop-color="#34d399" />
</linearGradient>
<marker id="arrowBoot" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
<path d="M0 0 L10 5 L0 10 z" fill="#7287a4"/>
</marker>
</defs>
<rect x="310" y="30" width="320" height="88" rx="18" fill="#ffd166" stroke="#c98900" stroke-width="2"/>
<text x="470" y="67" text-anchor="middle" fill="#4a2c00" font-size="24" font-weight="800">SCP ROM</text>
<text x="470" y="95" text-anchor="middle" fill="#4a2c00" font-size="14">Reset vector, tiny immutable first step</text>
<rect x="320" y="175" width="300" height="96" rx="18" fill="#a7f3d0" stroke="#0f9f7c" stroke-width="2"/>
<text x="470" y="212" text-anchor="middle" fill="#0b2f24" font-size="24" font-weight="800">SCP RAM Firmware</text>
<text x="470" y="240" text-anchor="middle" fill="#0b2f24" font-size="16">Clock, power, interconnect, SCMI, policy</text>
<rect x="290" y="328" width="360" height="96" rx="18" fill="#c4b5fd" stroke="#7c3aed" stroke-width="2"/>
<text x="470" y="365" text-anchor="middle" fill="#2d1752" font-size="24" font-weight="800">Power Up AP Complex</text>
<text x="470" y="393" text-anchor="middle" fill="#2d1752" font-size="16">Release reset, set boot address, enable clocks</text>
<rect x="340" y="481" width="260" height="84" rx="18" fill="#93c5fd" stroke="#2563eb" stroke-width="2"/>
<text x="470" y="517" text-anchor="middle" fill="#10203b" font-size="24" font-weight="800">TF-A BL1</text>
<text x="470" y="542" text-anchor="middle" fill="#10203b" font-size="16">Trusted minimal first-stage loader</text>
<rect x="340" y="622" width="260" height="84" rx="18" fill="#93c5fd" stroke="#2563eb" stroke-width="2"/>
<text x="470" y="658" text-anchor="middle" fill="#10203b" font-size="24" font-weight="800">TF-A BL2</text>
<text x="470" y="683" text-anchor="middle" fill="#10203b" font-size="16">Loads later firmware images</text>
<rect x="230" y="763" width="480" height="96" rx="18" fill="#fdba74" stroke="#ea580c" stroke-width="2"/>
<text x="470" y="800" text-anchor="middle" fill="#4a2207" font-size="24" font-weight="800">BL31 + StandaloneMM</text>
<text x="470" y="828" text-anchor="middle" fill="#4a2207" font-size="16">EL3 runtime firmware + secure management mode</text>
<rect x="280" y="916" width="380" height="84" rx="18" fill="url(#bootGrad)" opacity="0.85" stroke="#0f766e" stroke-width="2"/>
<text x="470" y="952" text-anchor="middle" fill="#052420" font-size="24" font-weight="800">UEFI → GRUB → Linux</text>
<text x="470" y="977" text-anchor="middle" fill="#052420" font-size="14">OS handoff, kernel boot, normal life begins</text>
<line x1="470" y1="118" x2="470" y2="175" stroke="#7287a4" stroke-width="4" marker-end="url(#arrowBoot)"/>
<line x1="470" y1="271" x2="470" y2="328" stroke="#7287a4" stroke-width="4" marker-end="url(#arrowBoot)"/>
<line x1="470" y1="424" x2="470" y2="481" stroke="#7287a4" stroke-width="4" marker-end="url(#arrowBoot)"/>
<line x1="470" y1="565" x2="470" y2="622" stroke="#7287a4" stroke-width="4" marker-end="url(#arrowBoot)"/>
<line x1="470" y1="706" x2="470" y2="763" stroke="#7287a4" stroke-width="4" marker-end="url(#arrowBoot)"/>
<line x1="470" y1="859" x2="470" y2="916" stroke="#7287a4" stroke-width="4" marker-end="url(#arrowBoot)"/>
</svg>
</div>

### SCP ROM

This is the tiny immutable foothold after reset. It does just enough to begin executing SCP firmware from writable memory. No fireworks, no fancy policy, just the first trustworthy step.

### SCP RAM firmware

Now the real management code runs. The SCP initializes clocks, power domains, fabric controls, and whatever else the platform needs before the application world can boot. This is where the machine stops being a collection of powered-off blocks and starts becoming an actual system.

### Powering the AP side

The SCP powers up the application processor complex, deasserts resets, and ensures the boot path points to trusted firmware. This part matters enormously because boot bugs here are the kind that make you question your life choices at 2:17 AM.

### TF-A BL1

**Trusted Firmware-A BL1** is the first AP-side trusted stage. It is small, security-focused, and responsible for establishing the next steps in the trusted chain. Think of it as the doorman who checks the ID of every later boot stage.

### TF-A BL2

**BL2** is the image loader and setup stage. It loads later firmware payloads and prepares execution context. If BL1 is the doorman, BL2 is the logistics manager carrying boxes around backstage.

### BL31 + StandaloneMM

**BL31** runs at EL3 and stays resident as the secure runtime firmware. It handles secure monitor duties, power-state coordination hooks, and trusted runtime services. Alongside it, **StandaloneMM** provides a secure management-mode environment often used with UEFI secure services. In human terms: BL31 is the trusted supervisor, and StandaloneMM is the secure side office doing careful paperwork.

### UEFI

UEFI takes over for more general platform initialization, boot policy, device discovery, and preparing the OS launch. This is where the platform begins to look like a familiar modern server boot environment.

### GRUB

GRUB selects and launches the kernel. It is the last major “general-purpose bootloader” stop before the OS starts. We’ve now moved from hardware bring-up into software launch orchestration.

### Linux

Finally Linux boots, discovers the platform, sets up drivers, talks to the SCP via SCMI for management services, and gets on with the useful business of being a server OS.

## 5. CMN-700 Mesh Interconnect — The City Road Network of the SoC

If the CPUs are the office workers and memory is the filing system, then **CMN-700** is the city-wide transportation grid. It is a **mesh interconnect**, meaning traffic moves through a network of interconnected nodes rather than a single shared bus. That matters because 16 cores plus I/O plus memory controllers can get very busy, very quickly.

*Inline SVG: CMN-700 Mesh Topology*
<div class="svg-diagram">
<svg viewBox="0 0 1080 720" role="img" aria-label="CMN-700 mesh topology with node types">
<defs>
<marker id="arrowMesh" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto">
<path d="M0 0 L10 5 L0 10 z" fill="#70839f"/>
</marker>
</defs>
<rect x="40" y="40" width="1000" height="600" rx="26" fill="none" stroke="var(--svg-grid)" stroke-width="2"/>
<g stroke="#3c4f6d" stroke-width="4">
<line x1="190" y1="150" x2="890" y2="150"/>
<line x1="190" y1="300" x2="890" y2="300"/>
<line x1="190" y1="450" x2="890" y2="450"/>
<line x1="190" y1="600" x2="890" y2="600"/>
<line x1="190" y1="150" x2="190" y2="600"/>
<line x1="423" y1="150" x2="423" y2="600"/>
<line x1="656" y1="150" x2="656" y2="600"/>
<line x1="890" y1="150" x2="890" y2="600"/>
</g>
<g font-size="18" font-weight="800" text-anchor="middle">
<g>
<circle cx="190" cy="150" r="38" fill="#8ea5ff" stroke="#3457d5" stroke-width="3"/><text x="190" y="157" fill="#13233b">RN-F</text>
<circle cx="423" cy="150" r="38" fill="#8ea5ff" stroke="#3457d5" stroke-width="3"/><text x="423" y="157" fill="#13233b">RN-F</text>
<circle cx="656" cy="150" r="38" fill="#8ea5ff" stroke="#3457d5" stroke-width="3"/><text x="656" y="157" fill="#13233b">RN-F</text>
<circle cx="890" cy="150" r="38" fill="#8ea5ff" stroke="#3457d5" stroke-width="3"/><text x="890" y="157" fill="#13233b">RN-F</text>
</g>
<g>
<circle cx="190" cy="300" r="38" fill="#b2f5ea" stroke="#0f9f7c" stroke-width="3"/><text x="190" y="307" fill="#0b2f24">HN-F</text>
<circle cx="423" cy="300" r="38" fill="#b2f5ea" stroke="#0f9f7c" stroke-width="3"/><text x="423" y="307" fill="#0b2f24">HN-F</text>
<circle cx="656" cy="300" r="38" fill="#b2f5ea" stroke="#0f9f7c" stroke-width="3"/><text x="656" y="307" fill="#0b2f24">HN-F</text>
<circle cx="890" cy="300" r="38" fill="#b2f5ea" stroke="#0f9f7c" stroke-width="3"/><text x="890" y="307" fill="#0b2f24">HN-F</text>
</g>
<g>
<circle cx="190" cy="450" r="38" fill="#fcd34d" stroke="#d97706" stroke-width="3"/><text x="190" y="457" fill="#4b3200">SN-F</text>
<circle cx="423" cy="450" r="38" fill="#fcd34d" stroke="#d97706" stroke-width="3"/><text x="423" y="457" fill="#4b3200">SN-F</text>
<circle cx="656" cy="450" r="38" fill="#fcd34d" stroke="#d97706" stroke-width="3"/><text x="656" y="457" fill="#4b3200">RN-I</text>
<circle cx="890" cy="450" r="38" fill="#fda4af" stroke="#db2777" stroke-width="3"/><text x="890" y="457" fill="#4a1028">RN-D</text>
</g>
<g>
<circle cx="190" cy="600" r="38" fill="#fcd34d" stroke="#d97706" stroke-width="3"/><text x="190" y="607" fill="#4b3200">SN-F</text>
<circle cx="423" cy="600" r="38" fill="#fcd34d" stroke="#d97706" stroke-width="3"/><text x="423" y="607" fill="#4b3200">SN-F</text>
<circle cx="656" cy="600" r="38" fill="#93c5fd" stroke="#2563eb" stroke-width="3"/><text x="656" y="607" fill="#10203b">Cfg</text>
<circle cx="890" cy="600" r="38" fill="#c4b5fd" stroke="#7c3aed" stroke-width="3"/><text x="890" y="607" fill="#2d1752">Dbg</text>
</g>
</g>
<text x="540" y="680" text-anchor="middle" fill="var(--svg-text)" font-size="16" font-weight="700">RN-F = Fully Coherent Request Node • HN-F = Home Node + Snoop Filter • SN-F = Slave/Memory • RN-I = Coherent IO • RN-D = Non-coherent IO</text>
</svg>
</div>

### Key node types

- **RN-F** — Request nodes for fully coherent requesters, such as CPU clusters.
- **HN-F** — Home nodes that track memory home responsibilities and often include snoop filter behavior.
- **SN-F** — Slave nodes, usually associated with memory system endpoints/controllers.
- **RN-I** — I/O request nodes for coherent I/O agents.
- **RN-D** — Non-coherent I/O request nodes.

### How coherency works

Suppose Core 3 wants a cache line. The mesh routes the request to the relevant **home node**. That home node is responsible for determining the line’s state and whether another cache might already own or share it. If another core has a dirty copy, the fabric can trigger snoops to maintain a single coherent memory image. Nobody gets to quietly keep stale data in their pocket like a kid pretending they already handed in the homework.

### Snoop filter vs directory

These two ideas are related but not identical:

- A **directory** tracks ownership/sharing state for cache lines in a structured way.
- A **snoop filter** tries to reduce unnecessary snoops by remembering where lines are likely present.

In practice, a snoop filter helps avoid shouting “Who has this line?!” to every core on every miss. It narrows the blast radius. That reduces traffic, saves power, and improves scalability. On a big coherent system, that’s the difference between an orderly group chat and a family WhatsApp thread during a holiday argument.

## 6. Bus Hierarchy — APB, AHB, AXI, CHI and the “Use the Right Road” Rule

RD-N2 uses multiple interconnect protocols because not all traffic deserves a Formula 1 racetrack. Some accesses are tiny configuration pokes. Some are bulk DMA. Some need full hardware coherency. Putting all of that on one protocol would be like using a fire truck to deliver a cupcake.

*Inline SVG: Bus Hierarchy from Simple Control to Coherent Fabric*
<div class="svg-diagram">
<svg viewBox="0 0 1080 560" role="img" aria-label="Bus hierarchy diagram APB AHB AXI CHI">
<rect x="70" y="60" width="940" height="80" rx="18" fill="#fde68a" stroke="#ca8a04" stroke-width="2"/>
<text x="540" y="102" text-anchor="middle" fill="#4b3200" font-size="28" font-weight="800">APB</text>
<text x="540" y="128" text-anchor="middle" fill="#4b3200" font-size="15">Low-speed config • simple handshake • 32-bit • no burst</text>
<rect x="120" y="180" width="840" height="80" rx="18" fill="#bfdbfe" stroke="#2563eb" stroke-width="2"/>
<text x="540" y="222" text-anchor="middle" fill="#10203b" font-size="28" font-weight="800">AHB</text>
<text x="540" y="248" text-anchor="middle" fill="#10203b" font-size="15">SCP-local and medium-complexity paths • supports bursts • pipelined</text>
<rect x="170" y="300" width="740" height="88" rx="18" fill="#a7f3d0" stroke="#0f9f7c" stroke-width="2"/>
<text x="540" y="344" text-anchor="middle" fill="#0b2f24" font-size="28" font-weight="800">AXI</text>
<text x="540" y="370" text-anchor="middle" fill="#0b2f24" font-size="15">High-bandwidth data movement • DMA/peripherals • multiple outstanding txns</text>
<rect x="220" y="430" width="640" height="92" rx="18" fill="#ddd6fe" stroke="#7c3aed" stroke-width="2"/>
<text x="540" y="474" text-anchor="middle" fill="#2d1752" font-size="28" font-weight="800">CHI</text>
<text x="540" y="500" text-anchor="middle" fill="#2d1752" font-size="15">High-performance coherent interconnect • flit-based • cache-state aware</text>
<line x1="540" y1="140" x2="540" y2="180" stroke="#60748e" stroke-width="4"/>
<line x1="540" y1="260" x2="540" y2="300" stroke="#60748e" stroke-width="4"/>
<line x1="540" y1="388" x2="540" y2="430" stroke="#60748e" stroke-width="4"/>
</svg>
</div>

| Bus | Best used for | Key traits |
| --- | --- | --- |
| APB | Register configuration, control/status | Simple, low power, no burst, low throughput |
| AHB | SCP-local subsystem traffic | Pipelined, can burst, more capable than APB |
| AXI | DMA, memory-mapped high-bandwidth peripherals | Separate channels, IDs, outstanding transactions, high throughput |
| CHI | CPU-side coherent traffic and coherent I/O | Flit-based, coherent, scalable, cache-state aware |

### APB

**APB** is for low-speed configuration. Think register writes: enable this block, read that status bit, set this timeout. It is intentionally simple. Typical data widths are modest, burst transfers are not its thing, and that’s fine because it is not trying to be a data freeway.

### AHB

**AHB** is a step up. In many systems it is used in local subsystems where you need more capability than APB without dragging in the full complexity of AXI. For the SCP side, it often makes sense as a local fabric for memory and peripheral interactions that are more active than simple control traffic.

### AXI

**AXI** is where things get properly performance-oriented. You get separate read and write channels, bursts, IDs, and multiple outstanding transactions. That makes it ideal for DMA engines, high-throughput peripherals, and paths where data movement matters.

### CHI

**CHI** is for the grown-up coherent world. It is flit-based and designed for scalable coherent fabrics like CMN-700. This is where CPU clusters and coherent I/O live when memory ordering and cache coherency need to be preserved at scale. CHI is not just “faster AXI.” It is a higher-level protocol built for a different class of problem.

## 7. TZC-400 — The TrustZone Bouncer at the Memory Club Door

**TZC-400** stands for **TrustZone Address Space Controller**. Its job is to enforce which transactions may access which DRAM regions based on security attributes. In RD-N2 there are **8 TZC-400 instances**, typically aligning with **8 DRAM channels**. That one-per-channel arrangement keeps protection close to the actual memory paths.

Each controller supports **up to 9 programmable regions**. That lets platform firmware carve memory into secure and non-secure windows, with policy attached.

### Why this matters

ARM TrustZone divides the world broadly into **Secure** and **Non-secure**. The secure world hosts trusted firmware and potentially sensitive services. The non-secure world runs the normal OS and applications. If you don’t enforce boundaries at the memory controller side, then “security” is just an optimistic personality trait.

### NS/S access control

A transaction carries security attributes. TZC-400 checks whether that initiator and attribute set may access the target DRAM region. If not, the access is blocked or faulted. So if non-secure software tries to wander into a secure DRAM region, TZC says, “Absolutely not, my friend.”

> ✅ **Key insight:** A good mental model is a hotel with multiple wings. TrustZone says some rooms are VIP-only. TZC-400
> sits at each hallway entrance checking the wristbands. It doesn’t care how confidently you walk — if
> you don’t have the right tag, you are not getting in.

## 8. MHU — Tiny Doorbell Hardware with a Big Job

The **MHU**, or **Message Handling Unit**, is delightfully small in concept. It is often described as being on the order of **~500 gates of doorbell logic**. That number gives you the right vibe: this is not a rich mailbox processor or giant packet engine. It is more like a hardware nudge.

The MHU is used so one side of the system can say to the other side, “Hey, wake up, there is something for you.” Usually that means the AP side tells the SCP to look at a message in shared memory, or the SCP nudges the AP side when a response is ready.

*Inline SVG: MHU Doorbell + Shared Memory Mechanism*
<div class="svg-diagram">
<svg viewBox="0 0 1100 560" role="img" aria-label="MHU and shared memory signaling">
<defs>
<marker id="arrowMhu" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0 0 L10 5 L0 10 z" fill="#667b97"/>
</marker>
</defs>
<rect x="60" y="120" width="280" height="220" rx="20" fill="#bfdbfe" stroke="#2563eb" stroke-width="3"/>
<text x="200" y="170" text-anchor="middle" fill="#10203b" font-size="24" font-weight="800">Application Processors</text>
<text x="200" y="210" text-anchor="middle" fill="#10203b" font-size="17">Write SCMI payload</text>
<text x="200" y="236" text-anchor="middle" fill="#10203b" font-size="17">into shared SRAM</text>
<text x="200" y="286" text-anchor="middle" fill="#10203b" font-size="16">Then poke MHU SET register</text>
<rect x="380" y="90" width="340" height="300" rx="22" fill="#a7f3d0" stroke="#0f9f7c" stroke-width="3"/>
<text x="550" y="136" text-anchor="middle" fill="#0b2f24" font-size="22" font-weight="800">Shared Memory + MHU</text>
<rect x="420" y="170" width="260" height="82" rx="14" fill="#ecfeff" stroke="#0891b2" stroke-width="2"/>
<text x="550" y="205" text-anchor="middle" fill="#083344" font-size="22" font-weight="800">Dual-ported SRAM</text>
<text x="550" y="232" text-anchor="middle" fill="#083344" font-size="16">Payload lives here</text>
<rect x="420" y="280" width="260" height="72" rx="14" fill="#fde68a" stroke="#ca8a04" stroke-width="2"/>
<text x="550" y="312" text-anchor="middle" fill="#4b3200" font-size="20" font-weight="800">MHU doorbell bit</text>
<text x="550" y="336" text-anchor="middle" fill="#4b3200" font-size="16">SET → flip-flop → IRQ</text>
<rect x="810" y="120" width="230" height="220" rx="20" fill="#fbcfe8" stroke="#db2777" stroke-width="3"/>
<text x="925" y="170" text-anchor="middle" fill="#4a1028" font-size="28" font-weight="800">SCP</text>
<text x="925" y="210" text-anchor="middle" fill="#4a1028" font-size="17">Interrupt arrives</text>
<text x="925" y="236" text-anchor="middle" fill="#4a1028" font-size="17">Reads message from SRAM</text>
<text x="925" y="286" text-anchor="middle" fill="#4a1028" font-size="17">Processes SCMI command</text>
<line x1="290" y1="230" x2="420" y2="230" stroke="#667b97" stroke-width="4" marker-end="url(#arrowMhu)"/>
<line x1="680" y1="316" x2="810" y2="316" stroke="#667b97" stroke-width="4" marker-end="url(#arrowMhu)"/>
<line x1="810" y1="270" x2="680" y2="270" stroke="#667b97" stroke-width="4" marker-end="url(#arrowMhu)"/>
<text x="550" y="430" text-anchor="middle" fill="var(--svg-text)" font-size="18" font-weight="700">Important: MHU signals that data exists. The data itself is in shared memory.</text>
</svg>
</div>

### How the mechanism works

At its simplest, one side writes a bit to an MHU **SET register**. Internally that sets a flip-flop. That flip-flop causes an interrupt or alert on the other side. The receiver sees the doorbell, services it, and later clears or acknowledges it.

That’s it. No huge payload. No fancy packet routing. Just a digital ding-dong.

### Why shared memory is needed too

Because the MHU only tells you that there is a message — it does not *carry* a useful message body. If you need to send a command, parameters, status, or response payload, you place that in shared memory. Then you use MHU to notify the peer that the buffer is ready.

So the division of labor is:

- **MHU = notification**
- **Shared memory = actual data**

Think of MHU as ringing your friend’s doorbell, while shared memory is the note you slid under the mat. The bell gets attention; the note contains the actual information.

## 9. SCMI Protocol — The Polite Language AP and SCP Use to Talk

**SCMI** stands for **System Control and Management Interface**. This is the standardized protocol the application world uses to request system management services from the SCP. Instead of every SoC inventing a weird private command format like an overconfident startup naming its product “TaskLynxX,” SCMI gives a stable, vendor-neutral model.

Typical SCMI services include:

- **Clock control** — query rates, set rates, enable/disable clocks.
- **Power domain control** — on/off/state transitions.
- **Sensors** — temperature, voltage, telemetry.
- **Performance management** — performance levels related to DVFS.

### Message structure

An SCMI message usually contains a header plus payload. The header identifies protocol and message IDs, while the payload carries parameters. A response includes status and returned values.

```
struct scmi_msg {
    uint32_t header;      // protocol id, message id, token
    uint32_t payload[N];  // command-specific parameters
};

struct scmi_reply {
    int32_t status;       // success or error code
    uint32_t payload[M];
};
```

### Transport over MHU + shared memory

SCMI itself is protocol-level; it needs a **transport**. In RD-N2, that transport is commonly **shared memory plus MHU**:

1. AP writes SCMI request into shared memory.
2. AP rings SCP using MHU doorbell.
3. SCP interrupt handler notices the request.
4. SCP reads the buffer, processes it, writes back response.
5. SCP rings AP back.

That’s clean, low-overhead, and standardized enough that software stacks can rely on it without caring about endless product-specific weirdness.

## 10. Shared Memory — Dual-Ported SRAM, Not the SCP’s Private TCM

This topic is worth being extra explicit about, because it is easy to accidentally blur memories together.

The SCMI shared memory region is a **dual-ported SRAM** accessible to both the AP side and the SCP side. It is **not** the SCP’s private tightly coupled memory (TCM). The point is to provide a region that both sides can read and write without needing to share the SCP’s own internal execution memory.

In the RD-N2 setup described here, the same physical shared SRAM is visible at different addresses:

- **SCP view:** `0xA5000000`
- **AP view:** `0x45000000`

That is perfectly normal. Different bus fabrics or translation windows can map the same storage to different address ranges from different masters’ points of view.

### Why on-chip shared memory is fine

Sometimes people hear “shared memory” and imagine danger. But inside the SoC, this is a controlled, intentional design. The shared region is there *because* both endpoints are trusted system components participating in a defined protocol. The message buffers are small, well-known, and paired with signaling. This is not random wild-west RAM scribbling.

> ✅ **Key insight:** The key distinction is ownership. The SCP’s private TCM is like its locked office drawer. The dual-
> ported shared SRAM is like the small in-tray on the desk outside the office where both the assistant
> and manager can leave signed forms.

## 11. Power Domains — Turning Big Systems On and Off Without Drama

Power management on RD-N2 is hierarchical. At a high level you can think in terms of:

- **SYSTOP** — the top-level system domain.
- **CLUSTER** — groups of CPU logic beneath the system domain.
- **CORE** — individual CPU core domains.

But the tree also extends to non-CPU resources: **PCIe**, debug blocks, and other **I/O macros**. Because if you can’t power-gate peripherals independently, your power budget starts looking like an apology letter.

*Inline SVG: Power Domain Tree*
<div class="svg-diagram">
<svg viewBox="0 0 1100 700" role="img" aria-label="Power domain tree for RD-N2">
<defs>
<marker id="arrowPower" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
<path d="M0 0 L10 5 L0 10 z" fill="#667b97"/>
</marker>
</defs>
            <!-- SYSTOP -->
<rect x="420" y="30" width="260" height="64" rx="16" fill="#5b7cfa" opacity="0.9" stroke="#284bcb" stroke-width="3"/>
<text x="550" y="70" text-anchor="middle" fill="#ffffff" font-size="26" font-weight="800">SYSTOP</text>
            <!-- Arrows from SYSTOP to Clusters -->
<g stroke="#667b97" stroke-width="3" fill="none" marker-end="url(#arrowPower)">
<line x1="460" y1="94" x2="195" y2="155"/>
<line x1="500" y1="94" x2="425" y2="155"/>
<line x1="600" y1="94" x2="675" y2="155"/>
<line x1="640" y1="94" x2="905" y2="155"/>
</g>
            <!-- Clusters -->
<g fill="#a7f3d0" stroke="#0f9f7c" stroke-width="2.5">
<rect x="100" y="155" width="190" height="60" rx="14"/>
<rect x="330" y="155" width="190" height="60" rx="14"/>
<rect x="580" y="155" width="190" height="60" rx="14"/>
<rect x="810" y="155" width="190" height="60" rx="14"/>
</g>
<g fill="#0b2f24" font-size="20" font-weight="800" text-anchor="middle">
<text x="195" y="192">Cluster 0</text>
<text x="425" y="192">Cluster 1</text>
<text x="675" y="192">Cluster 2</text>
<text x="905" y="192">Cluster 3</text>
</g>
            <!-- Arrows from Clusters to Cores -->
<g stroke="#2563eb" stroke-width="2" fill="none" marker-end="url(#arrowPower)">
<line x1="195" y1="215" x2="195" y2="270"/>
<line x1="425" y1="215" x2="425" y2="270"/>
<line x1="675" y1="215" x2="675" y2="270"/>
<line x1="905" y1="215" x2="905" y2="270"/>
</g>
            <!-- Cores row -->
<g fill="#bfdbfe" stroke="#2563eb" stroke-width="2">
              <!-- Cluster 0 cores -->
<rect x="110" y="270" width="42" height="50" rx="8"/>
<rect x="157" y="270" width="42" height="50" rx="8"/>
<rect x="204" y="270" width="42" height="50" rx="8"/>
<rect x="251" y="270" width="42" height="50" rx="8"/>
              <!-- Cluster 1 cores -->
<rect x="340" y="270" width="42" height="50" rx="8"/>
<rect x="387" y="270" width="42" height="50" rx="8"/>
<rect x="434" y="270" width="42" height="50" rx="8"/>
<rect x="481" y="270" width="42" height="50" rx="8"/>
              <!-- Cluster 2 cores -->
<rect x="590" y="270" width="42" height="50" rx="8"/>
<rect x="637" y="270" width="42" height="50" rx="8"/>
<rect x="684" y="270" width="42" height="50" rx="8"/>
<rect x="731" y="270" width="42" height="50" rx="8"/>
              <!-- Cluster 3 cores -->
<rect x="820" y="270" width="42" height="50" rx="8"/>
<rect x="867" y="270" width="42" height="50" rx="8"/>
<rect x="914" y="270" width="42" height="50" rx="8"/>
<rect x="961" y="270" width="42" height="50" rx="8"/>
</g>
<g fill="#10203b" font-size="12" font-weight="700" text-anchor="middle">
<text x="131" y="300">C0</text><text x="178" y="300">C1</text><text x="225" y="300">C2</text><text x="272" y="300">C3</text>
<text x="361" y="300">C4</text><text x="408" y="300">C5</text><text x="455" y="300">C6</text><text x="502" y="300">C7</text>
<text x="611" y="300">C8</text><text x="658" y="300">C9</text><text x="705" y="300">C10</text><text x="752" y="300">C11</text>
<text x="841" y="300">C12</text><text x="888" y="300">C13</text><text x="935" y="300">C14</text><text x="982" y="300">C15</text>
</g>
            <!-- Separator label -->
<text x="550" y="380" text-anchor="middle" fill="var(--svg-text)" font-size="16" font-weight="700" opacity="0.6">── Peripheral Power Domains ──</text>
            <!-- Arrows from SYSTOP to Peripherals (routed around cores) -->
<g stroke="#d97706" stroke-width="3" fill="none" marker-end="url(#arrowPower)">
<line x1="420" y1="70" x2="60" y2="70"/>
<line x1="60" y1="70" x2="60" y2="430"/>
<line x1="60" y1="430" x2="155" y2="430"/>
<line x1="500" y1="94" x2="470" y2="420"/>
<line x1="680" y1="70" x2="1050" y2="70"/>
<line x1="1050" y1="70" x2="1050" y2="430"/>
<line x1="1050" y1="430" x2="935" y2="430"/>
<line x1="600" y1="94" x2="700" y2="420"/>
</g>
            <!-- Peripherals -->
<g fill="#fcd34d" stroke="#d97706" stroke-width="2.5">
<rect x="100" y="410" width="170" height="64" rx="14"/>
<rect x="380" y="410" width="170" height="64" rx="14"/>
<rect x="620" y="410" width="170" height="64" rx="14"/>
<rect x="850" y="410" width="170" height="64" rx="14"/>
</g>
<g fill="#4b3200" font-size="18" font-weight="800" text-anchor="middle">
<text x="185" y="448">PCIe</text>
<text x="465" y="448">Debug</text>
<text x="705" y="448">IO Macro A</text>
<text x="935" y="448">IO Macro B</text>
</g>
            <!-- Legend -->
<text x="550" y="540" text-anchor="middle" fill="var(--svg-text)" font-size="15" font-weight="700">PPUs drive transitions using head switches (PMOS), isolation cells, and reset sequencing.</text>
            <!-- Legend items -->
<g font-size="13" fill="var(--svg-text)">
<rect x="320" y="560" width="20" height="14" rx="4" fill="#a7f3d0" stroke="#0f9f7c"/>
<text x="346" y="572">= Cluster domain</text>
<rect x="480" y="560" width="20" height="14" rx="4" fill="#bfdbfe" stroke="#2563eb"/>
<text x="506" y="572">= Core domain</text>
<rect x="620" y="560" width="20" height="14" rx="4" fill="#fcd34d" stroke="#d97706"/>
<text x="646" y="572">= Peripheral domain</text>
</g>
</svg>
</div>

### PPU: the power transition orchestrator

The **PPU** (Power Policy Unit) manages power states for a domain. When a domain is turned off, this is not just “flip one switch and hope.” The PPU coordinates several things:

- **Head switches** — typically large **PMOS arrays** that connect or disconnect supply rails.
- **Isolation cells** — prevent powered-off logic from leaking garbage values into still-powered logic.
- **Reset control** — ensures logic comes back in a known state.
- **Handshake/status** — so software knows transitions completed cleanly.

Powering down without isolation is like letting a sleeping toddler control the stereo cables. Weird stuff leaks into places it absolutely shouldn’t.

## 12. PLL — The Frequency Factory

A **PLL** or **Phase-Locked Loop** is used to generate a stable higher-frequency clock from a reference clock. It is one of those blocks that sounds magical until you break it down — then it becomes elegant rather than mystical.

*Inline SVG: PLL Block Diagram*
<div class="svg-diagram">
<svg viewBox="0 0 1080 430" role="img" aria-label="PLL block diagram">
<defs>
<marker id="arrowPll" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0 0 L10 5 L0 10 z" fill="#667b97"/>
</marker>
</defs>
<rect x="30" y="170" width="120" height="70" rx="16" fill="#fde68a" stroke="#ca8a04" stroke-width="2"/>
<text x="90" y="213" text-anchor="middle" fill="#4b3200" font-size="20" font-weight="800">REFCLK</text>
<rect x="200" y="155" width="180" height="100" rx="18" fill="#bfdbfe" stroke="#2563eb" stroke-width="2"/>
<text x="290" y="198" text-anchor="middle" fill="#10203b" font-size="16" font-weight="800">PFD / Charge Pump</text>
<text x="290" y="220" text-anchor="middle" fill="#10203b" font-size="13">Compare phase &amp; freq</text>
<rect x="430" y="155" width="160" height="100" rx="18" fill="#a7f3d0" stroke="#0f9f7c" stroke-width="2"/>
<text x="510" y="198" text-anchor="middle" fill="#0b2f24" font-size="16" font-weight="800">Loop Filter</text>
<text x="510" y="220" text-anchor="middle" fill="#0b2f24" font-size="13">Smooth control signal</text>
<rect x="640" y="155" width="160" height="100" rx="18" fill="#fbcfe8" stroke="#db2777" stroke-width="2"/>
<text x="720" y="198" text-anchor="middle" fill="#4a1028" font-size="16" font-weight="800">VCO</text>
<text x="720" y="220" text-anchor="middle" fill="#4a1028" font-size="13">Voltage-controlled osc.</text>
<rect x="850" y="155" width="170" height="100" rx="18" fill="#ddd6fe" stroke="#7c3aed" stroke-width="2"/>
<text x="935" y="198" text-anchor="middle" fill="#2d1752" font-size="16" font-weight="800">POSTDIV</text>
<text x="935" y="220" text-anchor="middle" fill="#2d1752" font-size="13">Final output division</text>
            <!-- Forward arrows -->
<line x1="150" y1="205" x2="200" y2="205" stroke="#667b97" stroke-width="4" marker-end="url(#arrowPll)"/>
<line x1="380" y1="205" x2="430" y2="205" stroke="#667b97" stroke-width="4" marker-end="url(#arrowPll)"/>
<line x1="590" y1="205" x2="640" y2="205" stroke="#667b97" stroke-width="4" marker-end="url(#arrowPll)"/>
<line x1="800" y1="205" x2="850" y2="205" stroke="#667b97" stroke-width="4" marker-end="url(#arrowPll)"/>
            <!-- Output arrow -->
<line x1="1020" y1="205" x2="1060" y2="205" stroke="#667b97" stroke-width="4" marker-end="url(#arrowPll)"/>
<text x="1060" y="190" fill="var(--svg-text)" font-size="16" font-weight="800">OUT</text>
            <!-- Feedback path -->
<rect x="640" y="320" width="160" height="60" rx="16" fill="#fde68a" stroke="#ca8a04" stroke-width="2"/>
<text x="720" y="357" text-anchor="middle" fill="#4b3200" font-size="18" font-weight="800">FBDIV</text>
<line x1="935" y1="255" x2="935" y2="290" stroke="#667b97" stroke-width="3"/>
<line x1="935" y1="290" x2="720" y2="290" stroke="#667b97" stroke-width="3"/>
<line x1="720" y1="290" x2="720" y2="320" stroke="#667b97" stroke-width="3" marker-end="url(#arrowPll)"/>
<line x1="640" y1="350" x2="290" y2="350" stroke="#667b97" stroke-width="3"/>
<line x1="290" y1="350" x2="290" y2="255" stroke="#667b97" stroke-width="3" marker-end="url(#arrowPll)"/>
            <!-- Labels -->
<text x="540" y="50" text-anchor="middle" fill="var(--svg-text)" font-size="20" font-weight="800">PLL Block Diagram</text>
<text x="540" y="410" text-anchor="middle" fill="var(--svg-text)" font-size="15">Output = REFCLK × FBDIV / POSTDIV</text>
</svg>
</div>

### The main pieces

- **PFD** — phase-frequency detector compares the reference against the feedback signal.
- **Loop filter** — smooths the control signal so the system does not jitter like it had too much espresso.
- **VCO** — voltage-controlled oscillator generates the high-frequency oscillation.
- **Feedback divider** — scales the output back for comparison.
- **Post-divider** — derives the final useful output frequency.

### The classic frequency formula

```
output_freq = ref_clk × FBDIV / POSTDIV
```

For example, with a 100 MHz reference, FBDIV of 24, and POSTDIV of 2, the output would be 1.2 GHz.

### Lock time

The PLL needs time to **lock**, meaning the control loop has stabilized and the output is safely tracking the intended frequency. Firmware cannot treat a PLL like an instant vending machine. You request a change, wait for lock indication, and only then trust the clock.

Why it matters? Because the rest of the clock tree depends on this stability. If the PLL is unsettled and you rush ahead, downstream logic may see a bad clock. And bad clocks are one of those bugs that cause very philosophical conversations with oscilloscopes.

## 13. DVFS — Because Power Bills and Thermal Budgets Are Real

**DVFS** stands for **Dynamic Voltage and Frequency Scaling**. The key power relationship people remember is:

```
P ∝ f × V^2
```

That square on voltage is the big deal. Frequency matters, but voltage is the dramatic one. Drop voltage a bit and dynamic power can fall a lot. Combine voltage reduction with frequency reduction and you can get near-cubic feeling savings in practical operating regimes. This is why DVFS exists: to trade performance for efficiency in a controlled way rather than pretending every workload deserves maximum boost all the time.

*Inline SVG: Safe DVFS Transition Flow*
<div class="svg-diagram">
<svg viewBox="0 0 1100 520" role="img" aria-label="DVFS transition flow diagram">
<defs>
<marker id="arrowDvfs" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
<path d="M0 0 L10 5 L0 10 z" fill="#667b97"/>
</marker>
</defs>
<rect x="370" y="40" width="360" height="74" rx="16" fill="#5b7cfa" stroke="#284bcb" stroke-width="3"/>
<text x="550" y="85" text-anchor="middle" fill="#ffffff" font-size="24" font-weight="800">Need new performance point?</text>
<rect x="120" y="190" width="350" height="240" rx="20" fill="#a7f3d0" stroke="#0f9f7c" stroke-width="3"/>
<text x="295" y="234" text-anchor="middle" fill="#0b2f24" font-size="28" font-weight="800">Speeding Up</text>
<text x="295" y="280" text-anchor="middle" fill="#0b2f24" font-size="20">1. Raise voltage first</text>
<text x="295" y="316" text-anchor="middle" fill="#0b2f24" font-size="20">2. Wait until rail is stable</text>
<text x="295" y="352" text-anchor="middle" fill="#0b2f24" font-size="18">3. Increase PLL / clock frequency</text>
<text x="295" y="388" text-anchor="middle" fill="#0b2f24" font-size="18">4. Resume higher-performance operation</text>
<rect x="630" y="190" width="350" height="240" rx="20" fill="#fde68a" stroke="#ca8a04" stroke-width="3"/>
<text x="805" y="234" text-anchor="middle" fill="#4b3200" font-size="28" font-weight="800">Slowing Down</text>
<text x="805" y="280" text-anchor="middle" fill="#4b3200" font-size="20">1. Reduce frequency first</text>
<text x="805" y="316" text-anchor="middle" fill="#4b3200" font-size="20">2. Confirm timing margin is safe</text>
<text x="805" y="352" text-anchor="middle" fill="#4b3200" font-size="20">3. Lower voltage afterward</text>
<text x="805" y="388" text-anchor="middle" fill="#4b3200" font-size="18">4. Save power without violating timing</text>
<line x1="550" y1="114" x2="295" y2="190" stroke="#667b97" stroke-width="4" marker-end="url(#arrowDvfs)"/>
<line x1="550" y1="114" x2="805" y2="190" stroke="#667b97" stroke-width="4" marker-end="url(#arrowDvfs)"/>
<text x="550" y="482" text-anchor="middle" fill="var(--svg-text)" font-size="19" font-weight="700">Golden rule: upshift = voltage first; downshift = frequency first.</text>
</svg>
</div>

### Why order matters

When you **increase performance**, you must usually **increase voltage first** before increasing frequency. Otherwise the logic may be asked to switch faster than the current voltage safely supports, which risks timing violations.

When you **decrease performance**, you do the reverse: **decrease frequency first**, then lower voltage. That way the logic slows down before you reduce its timing margin.

> 💡 **Note:** In other words: when speeding up, feed the athlete before asking them to sprint. When slowing down,
> tell them to stop running before taking away the snacks.

## 14. Clock Tree — Who Gets the Beat, and at What Rate?

RD-N2 distributes clocks carefully because a big SoC is really a synchronized collection of smaller timing islands. The platform includes **16 CPU clock groups** plus an **interconnect clock**. In plain English: each core or core-group path can have controlled distribution, and the fabric itself also needs its own stable heartbeat.

The clock tree takes outputs from root clock sources and PLLs, then fans them through muxes, gates, and dividers to the destinations that need them. This arrangement allows:

- Per-domain or per-group enable/disable control.
- Frequency changes through programmable PLL/divider settings.
- Power savings by gating idle domains.
- Separation between CPU clocks and interconnect timing.

The interconnect clock matters just as much as the CPU clocks. You can have very fast cores, but if the fabric clocking is poorly chosen, congratulations, you have built a line of sports cars trying to exit through a grocery-store parking gate.

## 15. GIC-700 — The Platform’s Interrupt Air-Traffic Controller

**GIC-700** is the interrupt controller for the platform. Its job is to collect interrupt sources, prioritize them, route them to the right CPU interfaces, and support the interrupt model that modern operating systems expect on ARM systems.

Why it matters: in a multicore platform, interrupts are not just “something happened.” They are “something happened, it has this priority, it belongs to this security world, and it should be presented to this processing element under these rules.” That’s a lot more organized than one giant blinking red light.

On a platform like RD-N2, GIC-700 helps coordinate:

- Private interrupts local to cores.
- Shared peripheral interrupts from devices.
- Routing and target selection across many cores.
- Priority handling and masking.
- Security-aware interrupt separation where relevant.

For Linux and firmware, the GIC is foundational. Without it, devices can scream for attention all day and the CPUs would have no disciplined way to respond.

## 16. SMMUv3 — Translating and Policing I/O Access

**SMMUv3** handles I/O address translation. It sits between DMA-capable devices and memory, translating device-generated addresses into actual physical memory addresses under software control. If the CPU MMU is passport control for software memory accesses, SMMU is the customs officer for devices.

In the RD-N2 story here, SMMUv3 services **5 I/O macros plus PCIe**. That means external and internal I/O agents can be brought into a managed virtualized address model instead of getting raw access to system memory.

### Why this is useful

- **Isolation:** a buggy or malicious device should not DMA all over memory.
- **Virtualization:** different guests or functions can get different I/O address spaces.
- **Convenience:** software can hand devices I/O virtual addresses rather than needing everything physically contiguous and identity-mapped.

So when PCIe devices or other I/O macros issue memory transactions, SMMUv3 can translate, check permissions, and keep the I/O world properly fenced. It is one of those blocks that you miss only when it’s gone — usually right after a device DMA engine writes into somewhere terrifying.

## 17. Wrap-Up — Putting the Whole RD-N2 Story Together

So here’s the short version of the long story we just walked through together.

RD-N2 is a reference server-class SoC design built around **16 Neoverse N2 cores** and a **CMN-700 coherent mesh**. The **SCP** based on **Cortex-M7** wakes up first, sets up power, clocks, and fabric state, and then releases the AP side. The boot path runs through **SCP ROM**, **SCP RAM firmware**, then trusted firmware stages **BL1**, **BL2**, **BL31**, and **StandaloneMM**, before passing to **UEFI**, **GRUB**, and finally **Linux**.

The coherent side is handled by **CHI over CMN-700**, while simpler control and peripheral paths use **APB**, **AHB**, and **AXI** where appropriate. **TZC-400** guards secure DRAM access. **MHU + shared memory** carry the signaling and payload transport for **SCMI**. **PPUs**, head switches, and isolation cells manage power states. **PLLs** and the clock tree feed the system’s timing. **DVFS** balances performance against power. **GIC-700** routes interrupts, and **SMMUv3** keeps I/O translation sane.

And that, honestly, is why RD-N2 is such a nice platform to study. It is not just a pile of IP blocks. It is a coherent architecture story. Every piece has a role, and the whole thing feels like a system rather than a shopping list.

> ✅ **Key insight:** If you made it this far, congratulations: you and I have effectively walked through a server SoC
> from first reset all the way to Linux, while chatting like friends and only mildly threatening the
> hardware with sarcasm.
