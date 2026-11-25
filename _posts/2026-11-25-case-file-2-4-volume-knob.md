---
layout: mypost
title: "DRAM Detective · Case File 2.4: The Volume Knob"
categories: [DDR5, Training]
lang: en
---

The line that decides 0 or 1 in Case 1.12 isn't handed down from heaven — it's a voltage twisted out by a DAC in the PHY. Just like a volume knob: 8 bits, 256 steps. Set the dial where you like and the decision line sits there, each step one 256th of VDDQ, about 4.3 mV on DDR5. And one gotcha: the top bit is hardwired to 1, so the volume is never below half — the default lands at 75%.

<iframe src="/animations/the-volume-knob.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The volume knob — a DAC twists out the decision line"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/the-volume-knob.html">open full screen ↗</a></p>

## One — the line is twisted out by a DAC

Case 1.12 was about how to use this line (sweep it and find the eye). This one is about what it's made of.

Write a number into a register and the DAC turns it into a voltage; that number is the Vref setting. So tuning Vref is nothing more than changing the DAC's input code.

8 bits, 256 steps, step size VDDQ/256 — about 4.3 mV a step on DDR5. Eight bits is a power of two again, same trick as the PI and the DLL: an N-bit register gives 2^N steps, not an arbitrary number.

The top bit isn't written by software — hardware ties it to 1, so only the lower 7 bits are yours. The default 75% falls out of exactly this: the hardwired MSB contributes half (50%), and the default control value adds the rest on top.

## Two — two layers: each DQ gets a base, plus one shared knob

Case 1.12 talked about one overall Vref plus a per-bit offset. Two roles — don't mix them up:

- Per-DQ control: each DQ has its own 7-bit value, its own Vref base.
- Shared adjustment: one common 7-bit signed value (-64 to +63), default 0, added to all DQs during a margin sweep.

So each DQ's final Vref is its own control plus the shared adjustment, added in hardware into the same DAC. That way you don't build a full 8-bit DAC per DQ — one base plus a small knob per lane, and the circuit shrinks.

## Three — inside it's a capacitive divider, not a resistor ladder

Not an R-2R resistor ladder — a capacitive divider. Metal-fringe capacitors in one string toward VDDQ/VTT and one toward VSS, and the node tapped in the middle is Vref:

```text
VDDQ ──┐
        ├── C_top (capacitors, the code decides how many) ──┐
VTT  ──┘                                                    ├── Vref ──► comparator (0/1)
                                                             │
VSS ──────── C_bottom (capacitors, the code decides) ────────┘
```

Capacitors for three reasons: area (metal-fringe caps beat precision polysilicon resistors on area and matching); supply tracking (a capacitive divider tracks the rail naturally — VDDQ drifts, Vref follows proportionally, so the decision line keeps its place relative to the swing); and power (a capacitive divider leaks no static current, while a resistor divider leaks forever).

## Four — a few more whys

Why 8 bits? 256 steps is fine enough — 4.3 mV a step, smaller than a single-ended signal's noise margin — and it's a power of two. Going to 10 bits and 1024 steps buys little, while the DAC's area and power double first.

Why is the top bit hardwired to 1? The inference: pin the 50% reference point straight into hardware, leave the lower 7 bits to add as an offset, and skip the circuit that would otherwise compute the 50% base. The source doesn't say so — this one is marked inference.

Why two layers at all? Manufacturing spread makes each DQ's comparator threshold naturally off by a few mV. The per-DQ layer sets each lane's own base; the shared layer is for the margin sweep to move all of them together. Each does its own job.

Why is the shared adjustment signed? A margin sweep has to go both up and down — probe the 0 side and the 1 side. ±64 steps, about ±25% of VDDQ, is more than enough to cancel that little offset.

**The line that decides 0 or 1 is twisted out by a volume knob — 256 steps, each one 1/256 of VDDQ.**
