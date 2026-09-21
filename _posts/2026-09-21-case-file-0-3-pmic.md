---
layout: mypost
title: "DRAM Detective · Case File 0.3: PMIC — the DIMM's Power Substation"
categories: [DDR5, PMIC]
lang: en
---

We've checked the family tree (Case 0.1) and the ID card (Case 0.2). Now the deeper question: who feeds this DIMM?

Every DIMM runs on power. In the DDR4 era the motherboard delivered it — every rail regulated on the board, and if the voltage was dirty, that was the board's problem. DDR5 changed the deal: the board sends one 5V feed and washes its hands of the rest. Whatever else the module wants, it makes itself. For that, every DDR5 DIMM carries its own live-in electrician — the **PMIC** (Power Management IC). A tiny chip soldered to the stick, turning one feed into three rails.

## The three rails

The PMIC takes the 5V in (plus a 3.3V line for its own management) and produces three core voltages:

| Rail | Voltage | Feeds | Think of it as |
|---|---|---|---|
| **VDD** | 1.1V | memory array + core logic | the room lights |
| **VDDQ** | 1.1V | the data interface (DQ/DQS) | the front door + elevator |
| **VPP** | 1.8V | the row-activation pump | the basement water pump |

Notice VDD and VDDQ are both 1.1V — but they are *not* the same thing. Lights and elevators are both 220V too, yet the elevator dying doesn't turn off your lights. This matters in a moment.

VPP is new in DDR5 (no external 1.8V rail in DDR4): before reading, the DRAM activates a row by pumping the word-line above VDD, and that pump gets its own 1.8V supply.

### The gotcha that trips 90% of people: it's VDDQ that squeezes the eye, not VDD

The eye diagram pictures DQ (data) being sampled by DQS (strobe). Both are IO pins, and both are fed by **VDDQ**. So:

- **VDDQ ripple** → squeezes the eye vertically, makes it bounce (directly crushes the eye)
- **VDD instability** → the core logic gets woozy, reads and writes both fail (but doesn't directly crush the eye)

Both can fail training, but the rail that crushes the data eye is **VDDQ — not VDD**. I got this one wrong the first time; don't.

## Why move power onto the DIMM?

DDR5 runs faster and packs more dies, so it wants more current, more rails, and tighter tolerance — VDD 1.1V core, VDDQ 1.1V IO, VPP 1.8V pump. The board can't deliver all that precisely, at that scale, across that many rails.

The bigger win is proximity. Picture an old apartment building: the water company pipes straight to every unit, and by the time the water arrives the pressure's long gone. A new building puts a booster pump on the roof — centimeters from every tap. That's the PMIC: centimeters from the dies, so less voltage drop and less ripple.

## Wait — the SPD is a front desk, not just an ID card

Back in Case 0.2 I showed you the SPD as a 1024-byte ID card. I left out half the story: DDR5's SPD device is not just an EEPROM — it's a **Hub** (SPD5108 / SPD5118, JESD300-5).

Think of a DIMM as a building. The DRAM dies are tenants; the PMIC is the substation, the RCD is the PA system, the temperature sensor is the thermometer — all "staff." The host (building management) can't run a wire to every one of them. So there's a front desk: the host talks to one device, and it routes to the rest. The SPD Hub is that front desk.

The Hub has two jobs:

1. **Main job**: hold the ID card (1024-byte SPD) + a built-in thermometer, plus a "neighbor registry" of who's on the module (PMIC and thermometer vendors — SPD bytes 194~221).
2. **Side job (today's subject)**: be the front desk, bridging the host's one bus onto a local bus. PMIC, RCD, and temperature sensor all hang off this local bus; the host reaches them through the Hub.

Why the detour? JESD300-5 §1: the Hub "allows isolation of a local bus from a Controller host bus".

- Isolation: a fault on the local bus doesn't drag down the host bus, and the host doesn't care about each local device's details.
- Bridging: one host bus can drive 8 DIMMs (8 Hubs at 0x50~0x57), each with its own local bus, independent of the rest.

## Finding the substation: the address

The PMIC has rails and registers, so the host needs its address first (JESD300-5 Table 5). A 7-bit address = high 4 bits "who are you" + low 3 bits "which DIMM":

- High 4 bits, **LID** (Local Device ID) = device type: PMIC = `1001`, RCD = `1011`, temp sensor = `0010` / `0110`.
- Low 3 bits, **HID** (Host ID) = slot number: `000`~`111` = DIMM 0~7.

Careful — there are two HIDs. One is the Hub's own (set by the HSA resistor, deciding its address 0x50~0x57); the other is the local device's (default `111`, changeable).

So the PMIC's address isn't fixed — one per DIMM:

| DIMM | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| PMIC addr | 0x48 | 0x49 | 0x4A | 0x4B | 0x4C | 0x4D | 0x4E | **0x4F** |

`0x4F` = `1001 111` = PMIC + DIMM 7. If someone tells you "the PMIC address is 0x4F," they mean the default HID=`111` — or their stick really is in DIMM 7.

Every DIMM's PMIC answers to 0x4F *locally*; the low 3 bits are the Hub's doing. When the host asks for `1001 + slot`, each Hub compares that slot against its own: the matching one rewrites the local HID to `111`, so only its PMIC answers; the others stay quiet (JESD300-5 §2.6.6).

One more wrinkle: 0x48~0x4F is the **7-bit** address; on the wire it's shifted left one bit and the read/write bit goes in the bottom, making **8 bits** — 0x48 becomes 0x90, SPD's 0x50 becomes 0xA0. Same address, two spellings: 7-bit is the ID, 8-bit is the wire byte.

The PMIC may talk I2C or I3C. I3C has dynamic address assignment (ENTDAA), but DDR5 deliberately skips it and uses SETAASA ("keep your static address") instead — so in either mode the substation's address is the static 0x48~0x4F.

And SMBus and I2C are basically one family (SMBus is a tightened I2C), while I3C is the backward-compatible successor — which is why a plain SMBus line reads the PMIC while it sits in its default I2C mode, but loses it once it's switched to I3C push-pull.

## Want to poke it yourself?

I wrote a UEFI Shell tool, **PmicTest**, that goes through the Hub and reads (or burns) the substation's NVM. Sibling of Case 0.2's SpdTest — SpdTest reads the ID card, PmicTest works the substation.

Three commands:

```
PmicTest.efi scan                                       # probe 0x48~0x4F, list present PMICs
PmicTest.efi read -c <ctrl> -ch <ch> -d <dimm>          # read R40~R6F + decode voltages
PmicTest.efi burn -c <ctrl> -ch <ch> -d <dimm> -b <blk> -data <32hex>  # burn one block
```

Real `scan` and `read` on an ARL-S board:

```
===== PMIC Scan (addresses 0x48..0x4F) =====
  DIMM 0 (0x48): PMIC present, hub mode=I2C
  DIMM 2 (0x4A): PMIC present, hub mode=I2C
  Total: 2 PMIC(s) present
=====================================

===== PMIC Vendor Region (R40~R6F, 48 bytes) =====
  DIMM: Controller=0 Channel=0 Dimm=0  (SMBus 0x90, 7-bit 0x48)
  Hub mode       : I2C
89 D9 00 00 00 78 63 00 00 78 63 78 63 80 88 00
CF 42 00 00 00 00 00 00 D2 DA 00 00 00 20 22 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  Voltage (decoded):
    VDD  (R45, SWA) : 1100 mV
    VDDQ (R49, SWC) : 1100 mV
    VPP  (R4B, SWD) : 1800 mV
===================================================
```

"hub mode=I2C" is the proof: this PMIC was never switched to I3C, so a plain SMBus line reads it straight.

Source: [github.com/peterhu/ddr5-shellkit](https://github.com/peterhu/ddr5-shellkit).

## What's inside the substation?

The PMIC has registers and a brain. The MRC's first job on boot isn't training — it's teaching the substation how to work.

**① Power-on sequencing** — the three rails can't come up together; there's an order and spacing. But that order isn't hardcoded by JEDEC — it's burned into NVM by the vendor (R40~R43), and vendors differ. The stick I measured brings VPP up first (R40 enables only VPP, waits 2ms), then VDD+VDDQ together (R41). Wrong order or spacing, and the dies won't start — or get hurt.

**② The CAMP signal** — all the PMICs on a platform share one CAMP wire. If any of them fails to power up, it pulls that wire low; the host sees it low and knows at least one substation never came up. Then it interrogates each PMIC's status registers, finds the dead one, and maps it out (JESD301 §2.13.1).

**③ Factory memory: three NVM blocks**

Everything so far is taught fresh at every power-up — written to volatile registers, forgotten at power-off. But the PMIC also has non-volatile memory: three blocks, 16 bytes each, 48 bytes total:

| block | registers | what's in it |
|---|---|---|
| **Block 40** | R40~R4F | power-on sequence, voltage/threshold, mode/switching frequency |
| **Block 50** | R50~R5F | current-limit threshold, power-off sequence, soft-start |
| **Block 60** | R60~R6F | reserved |

In plain words: Block 40 is "how to wake up" (order, voltages), Block 50 is "how to protect itself and shut down," Block 60 is empty. Three functional partitions, not three redundant copies — and they're programmable block-by-block (JESD301 §3.3.3).

**Block 40's voltages are the VDD/VDDQ/VPP defaults.**

The PMIC keeps each voltage in two copies: the **NVM copy** (R45/R47/R49/R4B) survives power-off — the burned-in default; the **runtime copy** (R21/R23/R25/R27) is volatile — what the substation is actually using right now. Change the runtime copy and it's gone at reboot; burn the NVM copy and it sticks.

| NVM (Block 40) | regulator | rail | runtime copy |
|---|---|---|---|
| R45 | SWA | **VDD** | R21 |
| R47 | SWB | VXX (4th rail) | R23 |
| R49 | SWC | **VDDQ** | R25 |
| R4B | SWD | **VPP** | R27 |

(SWB's VXX is the fourth switching regulator — idle on normal sticks; on high-density ones it pairs with SWA as dual-phase SWAB for more VDD current.)

One more encoding gotcha: R45/R49/R4B all read `0x78`, but VDD/VDDQ use an 800mV base while VPP uses a 1500mV base — so the same `0x78` decodes to 1.1V, 1.1V, 1.8V. Decode VPP with the wrong base and you'll read it as 1.1V.

And the power-on truth: the PMIC copies its NVM defaults into the runtime registers at first power-on ("at first power on, this register is automatically configured identically" — JESD301 Table 123 NOTE 1). MRC can then tweak the runtime copy for overclocking, but that tweak dies at power-off. The DIMM vendor region (R40~R6F) is locked by default; to write or burn it you first present the password — factory default `0x9473`. A burn goes: password → unlock → write 16 bytes → burn command (`0x81`/`0x82`/`0x85` per block) → poll `0x5A` → lock (JESD301 §3.3.3, Table 146).

## The soul of this case: floor vs. line

Training is about dialing "time" and "voltage" to find the most stable sampling point — and all of it rests on one assumption: the power "floor" is flat and steady.

> If the floor shakes, no line you draw stays straight.

<iframe src="/animations/pmic-substation.html" width="100%" height="640" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="PMIC power substation animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/pmic-substation.html">open full screen ↗</a></p>

## A hypothetical: three phases all fail

Suppose one DIMM fails three phases at once:

```
[Write Leveling]  TxDQS Delay ... more than ... failed
[Receive Enable]  Unable to find round trip latency
[2D Centering]    RxPerBitDeskew eye width is too small
```

You won't see this in the wild, because the first hard failure gets the DIMM mapped out — the other two phases never even run. So "three phases fail together" exists only in the hypothetical. Which is exactly what makes it worth asking: if one stick could fail three unrelated phases at once, which phase is really to blame?

None of them. It's the layer they share — power, clock, temperature, or the stick itself. Top suspect: the PMIC under-supplying, slowing every signal edge at once.

## Power vs. trace: how to smell the difference

Power or trace? Look at the scope and consistency of the failure:

| problem | signature |
|---|---|
| **Power** | global + random: every lane a bit thin, and the thinnest lane moves between retrains |
| **Trace/die** | local + fixed: one bit/lane is thin, always the same one |

One line: a trace problem lives on one wire; a power problem lives on the whole voltage plane.

## The golden rule

> **Power is the floor, training is the line. A shaking floor ruins even the straightest line.**

When training fails, don't reach for the parameters first — check whether the floor is flat. A power problem lives across the whole voltage plane — global and random; a trace problem lives on one wire — local and fixed.
