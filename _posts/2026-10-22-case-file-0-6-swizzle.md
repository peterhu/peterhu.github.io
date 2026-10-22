---
layout: mypost
title: "DRAM Detective · Case File 0.6: Find Your Seat (DQ/DQS Swizzle Mapping)"
categories: [DDR5, Training]
lang: en
---

The motherboard, in its rush to keep every wire short and equal-length, has shuffled the seating chart. CPU's DQ0 does not sit with DRAM's DQ0 — it may be holding hands with DQ6. Before any training can begin, someone has to walk the room, tap each wire, and rebuild the seating chart. That is swizzle mapping — the very first act of training.

## The shuffled classroom

I once assumed CPU DQ0 connects to DRAM DQ0, DQ1 to DQ1… But the board designer, chasing the shortest, most equal-length routes with the fewest vias, crossed the wires. This crossing has a name: **swizzle**. The DRAM's own DQ0–DQ7 are fixed by JEDEC, but how the board wires them is a hardware-design choice — so swizzle is a *board-level* concept, not in the JEDEC spec.

> Eight students, eight seats. The school let them sit wherever was close. Student #1 sits in seat 6, student #2 in seat 4. The teacher needs the seating chart before calling on anyone — shout "seat 6" and student #1 stands up.

Before the seating chart, one fact of geometry: a single read wakes every die of a rank (x8 = 8 dies), each die giving 8 bits — **64 bits = 8 bytes per beat**. A DDR5 channel splits into two 32-bit sub-channels, and a read bursts BL16 = 16 beats, so one burst = 16 × 8 = **128 bytes = 2 cache lines**. Swizzle governs *space* (which of the 8 bytes is who); BL16 governs *time* (how many beats). The two are orthogonal.

## Two layers of shuffle

The crossing is actually two layers:

| Layer | What's shuffled | Picture |
|---|---|---|
| **Byte / DQS** (coarse) | a whole group of 8 DQ + 1 DQS swaps with another | conduit #1 is wired to conduit #3 |
| **Bit** (fine) | the 8 DQ inside a group are scrambled | the red wire is really the blue wire |

Re-mapping takes two steps.

### Step 1 — Byte/DQS: light a pattern to find the conduit (x16 only)

Give one byte a unique on/off pattern — written to **MR registers over the CA bus**, not the DQ pins. The DRAM spits the pattern onto DQ on read; back at the CPU, look down each conduit for the one showing the same pattern. That conduit is wired to this byte.

Why x16 only: an x8 die carries one byte-lane (8 DQ + 1 DQS), so byte ↔ die is one-to-one, nothing to swap. An x16 die carries two byte-lanes (upper/lower), and the routing may swap them. This step needs **PDA (Per-DRAM Addressability)**: many dies share one bus, so a single command wakes them all — you can't tell whose answer is whose. PDA silences every die but one.

> A row of conduits all carry current at once — you can't tell which one conducts. PDA is turning off every breaker but one.

### Step 2 — Bit: test each wire one at a time

Once the conduit is known, light up one wire inside it at a time (write MR28/MR29 to say "invert this one"; the DRAM returns that wire's pattern on read). Read back which wire lit up. Eight rounds fill in the table `DqMapCpu2Dram`.

> One conduit, eight colored wires. Light one, watch which one glows at the far end. Eight rounds, and you've got every wire.

<iframe src="/animations/swizzle-wire-map.html" width="100%" height="620" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Swizzle wire map animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/swizzle-wire-map.html">open full screen ↗</a></p>

## The 255 hole

`DqMapCpu2Dram[N]` = which DRAM DQ the CPU's DQ N is wired to (valid 0–31). **255 (0xFF) is a sentinel**: "this DQ's mapping was never found." A 255 in the table is a hole in the seating chart — one wire couldn't be matched.

## Reading the feedback

The feedback is an 8-bit readback, one bit per DQ (bit=1 = that pin lit up). Normally exactly one bit lights. Read `0xC0 = 1100 0000` and two pins lit at once — DQ7 and DQ6 — a short or a leak. Don't rush to "short": ask which pin is *always* there and which one *changes*. The one that's always lit is the culprit.

## Why swizzle failure is always physical

Swizzle needs no timing, no eye diagram, no signal quality — only that every DQ can flip independently. So a failure here is almost certainly physical: a short, an open, a bad contact.

## Why no skew?

The most counterintuitive question: swizzle runs first, before any DQ/DQS timing is trained — so why doesn't skew corrupt the readback?

Skew is a *timing* problem: DQS's edge arriving late relative to DQ. But swizzle's readback does not use DQS as a strobe at all. It runs a "who triggers, who spits, who samples, who reads" pipeline:

| Role | Does |
|---|---|
| **CPGC** | fires a string of read commands so the DRAM keeps spitting the pattern (*triggers*) |
| **DRAM** | outputs the fixed pattern onto DQ, per the MR invert settings (*spits*) |
| **PHY** | Lane Sensing captures each DQ's static level into `DataTrainFeedback` (*samples*) |
| **Firmware** | reads `DataTrainFeedback`, counts the 1-bits, fills `DqMapCpu2Dram` (*reads*) |

It only asks "is this wire high or low *right now*" — not "when did it flip". Skew is about arrival time; measuring a static level doesn't care about arrival time.

(And even on a normal read, DQS and DQ are spec'd edge-aligned at 0 UI skew — JESD79-5 §9.11 Figure 211.)

## Does CA need training first? Yes — and CS even before that

Issuing commands (including swizzle's MR writes) rides the CA bus, so CA must work first. The order: low frequency → Early Command Training (CS first, then CA) → back to target frequency → swizzle. CS_n is a single "doorbell" wire (*whose* command is this, *when* is CA valid); CA[13:0] is fourteen "words" (*what* command, *what* address).

## A better mapping

The board half is fixed per board — hardcode it into a table and pick it by **BoardID** (a few GPIO pins = a few bits). The die-internal half is fixed per DIMM — train it once, burn it into the DIMM's own **SPD or PMIC non-volatile storage**, and it travels with the DIMM. Training happens only once, or when the stored mapping is gone.

> The wires are crossed; training first untangles them. If it can't — the physical layer has gone wrong. "Find your seat" is the precondition for every read and write that follows.
