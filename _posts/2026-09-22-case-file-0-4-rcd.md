---
layout: mypost
title: "DRAM Detective · Case File 0.4: RCD — the Conductor's Amplifier"
categories: [DDR5, RCD]
lang: en
---

We've checked the family tree (Case 0.1), the ID card (Case 0.2), and the power substation (Case 0.3). Now the finer question: how do commands and data travel across this DIMM?

Before memory does anything, the controller has to give it orders — read, write, activate, refresh, which row, which column, which bank. Those orders ride the **CA (Command/Address) bus**, a bundle of parallel wires. The actual payload rides the **DQ (data) lines**. And the two take different roads: commands get shouted through an amplifier; data gets carried straight from the warehouse.

## Two paths

Two kinds of signal, two different roads:

| Signal | What | Route |
|---|---|---|
| CA + CK | command/address + clock | **through the RCD** (to the RCD first, then fanned out to every die) |
| DQ + DQS | data + data strobe | **straight to the dies** (LRDIMM adds a DB to re-drive) |

Commands go through the amplifier; data goes straight.

<iframe src="/animations/rcd-amplifier.html" width="100%" height="620" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="RCD amplifier animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/rcd-amplifier.html">open full screen ↗</a></p>

## RCD: the amplifier

The **RCD (Registering Clock Driver)** (JESD82-513) is the chip on the command/address path. Its one job: take the host's orders, amplify them, and broadcast to every die.

Picture a full orchestra. The conductor shouts, and fifty musicians all need to hear it. In a big hall you can't do that with a bare throat — you need a PA system. The RCD is the PA.

## Why RCD

Because the load got too big. An RDIMM carries multiple ranks and dozens of dies, every die's command/clock pins wired up. The host's drive strength can't pull that many loads and keep the signal clean.

The RCD is the middle manager that takes the load off the boss: the host drives one chip (the RCD), the RCD re-drives every die, and all is well. But there's a price — the RCD brings two things:

1. **Delay**: every order takes a beat to enter and leave the RCD.
2. **Skew**: the traces from RCD to each die aren't the same length, so each die hears the order at a slightly different moment.

## Finding the amplifier

Like the PMIC, the RCD hangs on the local bus **behind the SPD Hub**. To talk to it, the host goes through the Hub (same drill as Case 0.3).

Its address (JESD300-5 Table 5): LID (device type) = `1011`, so `1011 + HID(slot)` = **0x58 ~ 0x5F**:

| DIMM | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| RCD addr | 0x58 | 0x59 | 0x5A | 0x5B | 0x5C | 0x5D | 0x5E | 0x5F |

And the whole bus side-by-side:

| Device | LID | Address |
|---|---|---|
| PMIC (substation) | `1001` | 0x48~0x4F |
| SPD Hub (front desk) | `1010` | 0x50~0x57 |
| **RCD (messenger)** | **`1011`** | **0x58~0x5F** |
| Temp sensor TS0/TS1 | `0010`/`0110` | 0x10/0x30 |

Access is the same as the PMIC: the host sends `1011 + slot`, each Hub compares the slot against its own, the matching one rewrites the local HID to `111`, and its RCD raises its hand (JESD300-5 §2.6.6).

## The soul: CA has no doorbell

The deepest difference between CA and DQ — and the reason CA needs its own training:

| | clocked by | doorbell? | analogy |
|---|---|---|---|
| **DQ** | DQS (one per lane, rides with the data) | yes | the courier follows the cargo; the bell rings when it arrives |
| **CA** | CK (a global clock, not its own) | no | the musician only follows the conductor's baton |

DQ has DQS as a doorbell riding along, so its timing is self-aligning — DQS absorbs part of the timing problem.

**CA has no bell; it only has the global CK.** If the RCD mangles the CA↔CK timing (delay + skew), the dies hear the beat wrong — listening when they shouldn't, deaf when they should.

That's why DQ's "follow the bell" trick can't save CA. CA needs its own dedicated training.

## CA training: de-skew for the command lines

One line: **adjust each CA line's delay, one by one, so every command/address line lands on CK's beat at the die.**

Same "sweep the delay, find the window" idea as de-skew:

1. The controller sends a known set of CA orders.
2. The dies (in training mode) report back over DQ whether they heard.
3. The controller sweeps the CA delay and finds the window that "hears".
4. It locks each line to the center of the window.

The eye width is the **intersection of every die's "I can hear" window** (the shortest stave of the barrel).

One more gotcha: **the pass line is graded by pattern difficulty (normalized)** — the pass line is the *deviation from this pattern's expected eye width*, not an absolute width. A hard tune gets a lower bar; not everyone is a star vocalist, and if you hold a hard tune to the easy bar, the whole class fails.

## CA vs DQ: how to tell them apart

| | CA failed | DQ failed |
|---|---|---|
| what happened | the order wasn't heard | the order was heard, the cargo was dropped |
| symptom | **won't boot / POST hangs** | **boots, but data corrupts / blue screen / ECC errors** |
| analogy | the conductor said "go" and nobody heard; everyone froze | they heard "go", but someone played the wrong note |

> Won't boot, stuck early in training → suspect CA. Boots but data is wrong → look at DQ.

## A case: CA training fails

The log prints "Best Eye Width ... smaller than minimum critical margin" and the channel is disabled.

The CA eye width is the intersection of all CA signals' "hear" windows; too small (below critical) and the channel is disabled.

One thing to remember: **CA goes through the RCD and does not pass through DFE.** CA (command direction) and DQ (data direction) are two separate paths; DFE is the echo canceller on the read-DQ path, and it can't save CA. Using the wrong medicine is a classic blunder.

## DIMM types

| Type | RCD | DB | CA/DQ route |
|---|---|---|---|
| UDIMM / SODIMM | no | no | CA straight to DRAM, no RCD |
| RDIMM | yes | no | CA through RCD, DQ direct |
| LRDIMM | yes | yes | CA through RCD + DQ through DB ("load-reduced") |
| MRDIMM | yes | yes(+mux) | adds rank muxing |

**DB (Data Buffer)** — LRDIMM only — re-drives the DQ data.

## DB: LRDIMM's load reduction

The RCD took the CA/clock load off the host, but the DQ load is still on it. RDIMM stops there, so the rank count it can support is limited.

**LRDIMM (Load-Reduced DIMM)** goes one step further and adds another middle manager — the **DB** — to take the DQ/DQS load off too:

| | CA/CK | DQ/DQS | ranks it can hold |
|---|---|---|---|
| RDIMM | through RCD | direct | few |
| LRDIMM | through RCD | **through DB** | many (8~16) |

Now the boss only drives the RCD and the DB, and one channel can hold more ranks. The DB re-driving DQ/DQS brings its own delay and skew, so LRDIMM's DQ training has to absorb the DB's skew too.

## Want to poke it yourself?

I wrote a UEFI Shell tool, **RCDTest**, that goes through the Hub and reads/writes the RCD's control words (RW00~RWxx) — straight from JESD82-513. One more sibling in the repo: SpdTest reads the ID card, PmicTest reads/burns the substation, RCDTest reads/writes the messenger.

```
RCDTest.efi scan                                       # probe 0x58~0x5F, list present RCDs
RCDTest.efi read -c <ctrl> -ch <ch> -d <dimm>          # read all control words RW00~RW5F
RCDTest.efi read -c <ctrl> -ch <ch> -d <dimm> -r <reg> # read one control word
RCDTest.efi write -c <ctrl> -ch <ch> -d <dimm> -r <reg> -data <8hex>  # write one control word
```

Expected output (I have no RDIMM to hand and no GNR server, so this is the shape, not a real dump):

```
===== RCD Scan (addresses 0x58..0x5F) =====
  DIMM 0 (0x58): RCD present (RW00=0xXXXXXXXX)
  Total: 1 RCD(s) present
=====================================

===== RCD Control Words (RW00~RW5F) =====
  DIMM: Controller=0 Channel=0 Dimm=0  (SMBus 0xB0, 7-bit 0x58)
  RW00 = 0xXXXXXXXX  (Global Features)
  RW05 = 0xXXXXXXXX  (DIMM Operating Speed)
  ...
==========================================
```

The RCD isn't a PMIC-style "write register address, read/write data" device — it uses a *sideband control word* protocol (JESD82-513 §7.5.7/7.5.8): block-write the command and setup, then block-read back Status + a DWord. And the command byte comes in two flavors — I2C (`0xC2`) and I3C (`0xC0`); RCDTest drives I2C, so it uses `0xC2`.

The RCD's revision is readable too — it lives in the paged control word **PG[3]RW6E (Vendor Revision ID)** (write RW5F to pick the page, then read RW6E). Real values from a server log: **IDT = Rev 1.0x33, Montage = Rev 2.0x11**.

Source: [github.com/peterhu/ddr5-shellkit](https://github.com/peterhu/ddr5-shellkit).

## The golden rule

> **Commands go through the amplifier, data goes straight. Miss the command and nothing boots; mangle the data and it blue-screens.**

CA is the order layer, DQ is the data layer. If CA dies, every die goes deaf and the machine won't boot; if DQ dies, the orders were heard but the cargo was dropped — it boots, but the data is wrong. And CA is the hardest to train because a whole bundle of lines has to hit the beat together, it has no doorbell, and the RCD adds skew on top.
