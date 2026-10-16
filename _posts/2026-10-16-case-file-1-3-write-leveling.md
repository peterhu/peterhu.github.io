---
layout: mypost
title: "DRAM Detective · Case File 1.3: Seize the Right Moment (Write Leveling)"
categories: [DDR5, Training]
lang: en
---

Writing is the host shipping data to a row of dies, but the "start" command travels by **relay** — passed from one die to the next, reaching the last one late — while the data travels by **dedicated line**, arriving at every die at once. The command staggers in late; the data lands simultaneously. Misaligned, the write goes crooked. **Write Leveling measures each die's time gap and staggers the data (DQS) so it matches each die's own clock.**

<iframe src="/animations/write-leveling-lamp.html" width="100%" height="640" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Write Leveling lamp animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/write-leveling-lamp.html">open full screen ↗</a></p>

## Why the write side needs leveling

DDR5 uses **fly-by** topology: command, address, and clock ride one daisy-chain line, stopping at every die like a bus. The payoff is fewer stubs and shorter traces; the cost is that every die sits at a different distance, so **its clock (CK) arrives at a different time**.

- Write data (DQS/DQ) takes **point-to-point** lines — direct, length-matched, arriving at all dies at once.
- The clock (CK) takes the **serpentine fly-by** line — stopping at every die, reaching the last one latest.

So the data arrives and the command hasn't — each die doesn't know which beat to write on. **The fix: measure each die's gap, stagger the DQS, and set each die's watch.**

### DQS is born earlier than CK

The write side can "add delay to align" only because **DQS arrives before CK**: CK takes the serpentine fly-by line (stops everywhere, slow); DQS takes the point-to-point short line (direct, length-matched, fast). So the write side only ever needs to **make the taxi wait for the bus** — never the dead end of "the bus arrived but the taxi hasn't".

### Why the command is broadcast, and the data isn't

| Signal | Topology | Nature |
|---|---|---|
| CK + CA (command/address) | fly-by (bus) | one-to-many broadcast |
| DQ + DQS (data/strobe) | point-to-point (taxi, length-matched) | one-to-one private |

**Why broadcast the command?** Because memory is accessed in **ranks**. A DDR5 ×8 die has only 8 DQ lines — one burst is 8 bits. To build a 64-bit bus, **8 ×8 dies are packed into one rank**. One READ goes out, and all 8 dies receive it at once, each spitting out its 8 bits to form 64. The access unit is the rank, not the die — and which rank is chosen by **CS_n**, not by point-to-point CA.

**Why not fly-by the data?** Fly-by's cost is **skew** — each die sees the signal at a different time. DQ is the fastest signal and must travel with DQS **matched, length-equal, and simultaneous** — skew kills it. And the data is already one-to-one, so point-to-point loses none of fly-by's benefits.

### The skew doesn't disappear — it just moves

DQS is point-to-point and arrives at every die simultaneously, so it has **no skew** — but it must align to CK, which **does** have skew.

- **CK's skew forces each die's DQS to wait for it.** If a far die's CK is 0.5 beats late, its DQS is delayed 0.5 beats. That's Write Leveling's job.
- **Adding delay ≠ slowing down**: the data rate is unchanged; only the arrival time shifts. And it's calibrated once at boot, fixed for every write after.
- **This echoes "DQS is born early"**: DQS early, CK late — that's why you can only *add* delay, never subtract.

## The lamp and the shutter (JESD79-5 §4.21.1)

The spec hands the host a clever trick to make the DRAM answer "am I aligned?":

> In Write Leveling mode, the DRAM samples its internal Write Leveling pulse with the controller's **last DQS rising edge**, and reports the result back asynchronously over DQ. The internal WL pulse is triggered by the WRITE command and stays low otherwise.

Two beats, in order:

1. **Beat one — the WRITE command lights the lamp** (lit by the DRAM's own timing).
2. **Beat two — the DQS last edge snaps the shutter**, and the DRAM looks at the lamp.

```
t0  controller sends WRITE (aligned to CK)
t1  DRAM receives WRITE (after fly-by), the WL pulse lights the lamp
     └─ the lamp stays lit for a fixed window, then goes out
t2  the lamp goes out
t3  controller sends DQS (with adjustable delay), the last edge snaps the shutter
     └─ at that instant the DRAM looks at its lamp: lit → 1, dark → 0
t4  DRAM reports the 1/0 back asynchronously over DQ
```

**The lamp is fixed; the shutter is adjustable.**

- The **lamp** (WL pulse) is triggered by the WRITE command — when it lights and how long, is the DRAM's own timing. **Fixed; the controller can't touch it.**
- The **shutter** (DQS last edge) is set by the controller's delay (CWL + PI phase). **Adjustable; this is Write Leveling's knob.**

So Write Leveling's essence: **a fixed lamp window, and a shutter you sweep across it**, watching whether the shutter falls inside or outside.

The controller doesn't know the lamp's exact position, but it knows the lamp lights once. So it sweeps the shutter from early to late, one click at a time:

| Shutter | Lamp | DQ |
|---|---|---|
| too early (t3 < t1, not yet lit) | dark | 0 |
| **just inside the lit window** (t1 ≤ t3 ≤ t2) | **lit** | **0 → 1 ← rising edge** |
| inside | lit | 1 |
| too late (t3 > t2, gone out) | dark | 1 → 0 (falling edge) |

**The 0→1 flip is the shutter just entering the lit window — the alignment point found.**

Sweep a string of shutter times and the feedback reads `0 0 0 **1 1 1 1** 0 0 0` — the first 1 is the left edge, the last 1 is the right edge, the run of 1s is how long the lamp stays lit. Take the center and the watch is set.

### Why the last edge, not just any edge?

A write burst toggles DQS many times (preamble + data), and **write latency (CWL) is aligned to the DQS's last rising edge** — so the last edge is the shutter that matches the write-timing definition.

### How does the DRAM know how long the preamble is?

The preamble length (**tWPRE**) is a **fixed, agreed value** — the DRAM counts, it doesn't guess. It's either set by the spec (DDR4) or programmed into the Mode Register (DDR5's tWPRE setting). The DRAM counts the DQS edges: after the fixed number of preamble beats, the next beat is the first data bit.

### Wait — CWL is fixed, so why train?

Because "beat 8" is not one global instant. CWL=8 is just the agreement — "count 8 beats after WRITE" — but the count starts from **each die's own CK edge**, and the CK is fly-by, arriving at a different time for every die.

So **each die's "beat 8" is a different absolute instant**: near die early, far die late. And DQS is point-to-point, arriving at **one** instant — it can only match one die's "beat 8"; the rest land off.

| | CK (bus, fly-by) | DQS (taxi, length-matched) |
|---|---|---|
| to the near die | early | instant T |
| to the far die | **0.5 late** | **still instant T** (length-matched, not late) |

Each die's "beat 8" counts 8 from its own CK: near die → instant T, far die → instant T + 0.5. DQS arrives at T, so it hits the near die's beat 8 but the far die's beat 7.5 — 0.5 early.

**Fix 0.5 in the host PHY and the far die's DQS lands at T+0.5 = its own beat 8.** CK late by how much, DQS delayed by how much. The one that takes longer to the far die is CK (the bus), not DQS (the length-matched taxi).

**What does Write Leveling actually change? Not CWL, but the DQS delay.** CWL stays 8; what changes is the per-die DQS delay. The result is a per-die list of "how late should this DQS be", measured once at boot and written into the host PHY's delay line.

### Why not give each die its own CWL?

Because CWL is already counted from each die's own CK — **the skew is already baked in**. Adding per-die CWL double-counts it.

With a uniform CWL=8 (4 dies, fly-by making CK 0.5 later each):

| die | CK | CWL | expects data at (absolute) |
|---|---|---|---|
| die1 | 0 | 8 | 0+8 = **8** |
| die2 | 0.5 | 8 | 0.5+8 = **8.5** |
| die3 | 1.0 | 8 | 1.0+8 = **9** |
| die4 | 1.5 | 8 | 1.5+8 = **9.5** |

"Farther = later" (8-8.5-9-9.5) is **already there** — a single CWL=8 gives it for free, because each die counts from its own late CK.

If instead you give farther dies a bigger CWL (8, 8.5, 9, 9.5):

| die | CK | CWL | expects data at (absolute) |
|---|---|---|---|
| die1 | 0 | 8 | 0+8 = **8** |
| die2 | 0.5 | 8.5 | 0.5+8.5 = **9.0** |
| die3 | 1.0 | 9 | 1.0+9 = **10** |
| die4 | 1.5 | 9.5 | 1.5+9.5 = **11** |

The spacing doubles from 0.5 to 1.0 — "farther" got counted **twice**: once into "CK late", once into "bigger CWL". Double-counted, and worse the farther you go.

And a hard constraint: **CWL is an integer in the Mode Register** — you can set 8 or 9, never 8.5. Fly-by skew is sub-cycle; CWL is a whole-cycle knob that can't hit sub-cycle granularity — that's what the PI (phase interpolator) does, which is Write Leveling's Fine stage.

Why "delay DQS" beats "per-die CWL", beyond double-counting and the integer constraint:
1. **Granularity** — skew is sub-cycle, CWL is whole-cycle; only the PI hits sub-cycle.
2. **Blast radius** — CWL drags a pile of write timings (tWR, burst length, pre/postamble); changing it moves everything. DQS delay touches one line's phase, precisely and locally.
3. **Unified agreement** — a whole rank shares one CWL; per-die CWL breaks the "one rank, one agreement" design.

### Why only one 0→1, not the whole waveform?

1. **DQ has a 1-bit mouth** — it can only answer "0" or "1", not a whole waveform.
2. **The controller only wants that edge** — it doesn't care what the lamp looks like, only where DQS should stop. So it sweeps, one 0/1 at a time, and probes out the pulse's left and right edges.

## Fine and Coarse

| Stage | Adjusts | Granularity |
|---|---|---|
| **Coarse** (first) | `CwlAdj` | whole-cycle (which beat) |
| **Fine** (second) | `PiDelay` (PI) | sub-cycle (which tick) |

> Wait — CWL=8 stays fixed, so why does Coarse "adjust CWL"? Naming gotcha: `CwlAdj` is **not the DRAM's CWL**; it's a whole-cycle delay the host adds on the DQS send path. CWL (the DRAM's MR value) stays 8; CwlAdj and PI are two layers of compensation added on top: **DQS arrival = CWL (baseline) + CwlAdj (whole-cycle) + PI (sub-cycle)**.

Fine sweeps **-64 to +96, 161 ticks** (1 PI tick each), snapping the shutter one tick at a time and recording a bitmap:

- **RisingEdge** = first 1 = shutter just entered the lit window;
- **FallingEdge** = last 1 = shutter just left it;
- **PulseWidth** = run of 1s = how long the lamp stays lit.

**Final alignment = RisingEdge + PulseWidth/2** (the center of the pulse).

## Write levels vs read gates

| | Write Leveling | Receive Enable |
|---|---|---|
| Essence | set the watch (find the 0→1 edge) | guard the door (find when DQS arrives) |
| Sweeps | one edge | one edge + one cycle count |
| Two-stage order | **Coarse first**, Fine second | **Fine first**, Coarse second |
| Cost | 78 ms | 23 ms |
| Feedback | DQ async 1 bit (0/1) | DQ counter 9-bit |

**The same fly-by skew: write compensates, read absorbs.**

| Direction | How fly-by skew is handled |
|---|---|
| Write | **compensate actively** — stagger each die's DQS to match its own CK |
| Read | **absorb passively** — each die's gate opens on its own |

## Why the order reverses

Write goes **Coarse then Fine**; read goes **Fine then Coarse**:

- The write-side DQS is **one-shot** — the WL pulse lights once after WRITE, so you must first align DQS to the roughly-right whole cycle (Coarse) before asking which tick within it (Fine).
- The read-side DQS is **periodic** — in rehearsal mode the lamp lights every cycle, so you can find the tick within a cycle first (Fine), then count the cycle (Coarse).

## The golden rule

> The clock staggers in late, the data arrives at once — stagger the DQS, set every die's watch. Seize the right moment.
