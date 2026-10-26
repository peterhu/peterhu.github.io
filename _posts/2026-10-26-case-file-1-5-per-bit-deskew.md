---
layout: mypost
title: "DRAM Detective · Case File 1.5: On Your Marks (Per-Bit De-Skew)"
categories: [DDR5, Training]
lang: en
---

A 400 m race staggers its starting blocks — the outer lanes are longer, so the outer runners start ahead, and every lane still runs a true 400 m.

Memory runs the same race. Eight DQ lines run side by side, but **each trace is a hair shorter or longer**, so each signal reaches the host a hair earlier or later. 2D Centering drew all eight DQ **on one starting line**; the long-trace DQ finishes late, the short-trace DQ finishes early, and each DQ's own eye center is still off.

Per-bit de-skew is the referee who **staggers the starting lines** — it gives each DQ its own delay, so each DQ is sampled at **its own eye center**.

<iframe src="/animations/per-bit-deskew.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Per-bit de-skew staggered starting lines animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/per-bit-deskew.html">open full screen ↗</a></p>

## One — why one line isn't enough

A DIMM has dozens of DQ lines running side by side, but **each trace differs a little** (trace length + on-die variation), so each signal arrives a hair earlier or later. That spread is **skew**.

The spec calls it **tRx_DQS2DQ_Skew (receive-side DQS-to-DQ skew)**, in JESD79-5 §4.8.7.2 (Data to Strobe Eye Height or Width Violations) — with timing margin and voltage margin, it fences the eye.

**2D Centering's limit**: it tunes **one** DQS sampling point (`RxDqsDelay`), one cut across the whole byte of 8 DQ. But the 8 DQ each skew differently, so the whole byte's eye is centered while **each DQ's own eye center is still off** — some early, some late.

> One starting line, eight runners, eight different track lengths — the runners don't cross together. 2D Centering is "one line for all"; per-bit de-skew is "a line for each."

### tDQSQ is absorbed in two steps

DQS and DQ carry a small skew **tDQSQ** (DQS-to-DQ, an intrinsic spec parameter the host can't change). It's **different per bit**, so it must be absorbed in two steps:

| Who | Absorbs | Granularity |
|---|---|---|
| 2D Centering | tDQSQ's **common** offset | whole byte (one DQS point for the group) |
| **per-bit de-skew** | tDQSQ's **per-bit** differences | per bit (each DQ alone) |

2D Centering's knob is **shared** — one `RxDqsDelay` for the whole group, physically unable to give each DQ a different value — so it only absorbs the offset all bits share; each bit's residual is left to per-bit de-skew.

## Two — two edges, two lanes per bit

DDR is **double data rate**: the clock (DQS) is a square wave that flips twice per cycle — up (rising edge) and down (falling edge). DDR samples on both, so one cycle carries 2 bits.

> A swing passes two turn-around points per round trip — DDR snaps the shutter at both. SDR snaps once, DDR snaps twice, doubling the data at the same frequency.

| Edge | Name | Delay register |
|---|---|---|
| rising (phase 0) | **P bit** | `RxDqsPBitDelay` |
| falling (phase 2) | **N bit** | `RxDqsNBitDelay` |

So each bit has **two** independent starting lines, P and N, each set on its own.

> A naming trap: here **P/N are the sampling edges** (rise/fall). The source's margin fields `.p`/`.n` (positive/negative = right/left boundary) are a different thing — don't mix them.

### Why P and N skew differently — two causes

P and N are sampled at two different instants, so their eye widths and centers differ. The root is that **the two edges are born asymmetric**:

1. **Drive asymmetry** — the rising edge is PMOS pulling up, the falling edge NMOS pulling down; the two transistors have different strength, so the two edges have different steepness. (Which edge is faster is *not* a law — in plain CMOS the electron-based NMOS is faster, but real DDR drivers compensate; the direction must be measured.)
2. **Duty-cycle distortion** (more important, more intuitive) — DDR samples on both edges, which presumes the two edges split the cycle evenly. In reality the high time and low time are often unequal, so the two bits get different-sized time windows.

> Day and night: one cycle is one day, high is day, low is night. It should be 12/12 — but winter is short-day, long-night (duty-cycle distortion). The day's job (P bit) is tight, the night's job (N bit) is loose — the two edges' windows differ, their centers shift apart, so P and N each need their own delay.

## Three — the 1-D centering

Per-bit de-skew's core is **1-D centering** — sweep one dimension (phase), find the middle:

1. For each bit, sweep the phase (delay), test whether each phase samples right (margin);
2. find the range that samples right (**passing range** = eye width);
3. take the **midpoint**, write it into `RxDqsPBitDelay` / `RxDqsNBitDelay`.

The source comment gives the clearest example (MemChipDdrio.c:8199-8200):

```
Example: N passing range [-32, 0], RxDqsNBitDelay will be -16
         P passing range [-34, -30], RxDqsPBitDelay will be -32
```

> The N bit samples right from [-32, 0], midpoint **-16**; the P bit from [-34, -30], midpoint **-32**. Each DQ finds its own eye width and takes its own center — that's one-dimensional centering.

### What are -32, -16?

These are **sampling-delay offsets, not absolute times**. `RxDqsNBitDelay` / `RxDqsPBitDelay` are an offset from the byte (strobe) delay — the `RxDqsDelay` 2D Centering set:

- **0** is the reference — the byte's delay, no extra offset;
- **positive** samples *later* than the base;
- **negative** samples *earlier* than the base.

So `-32` isn't "negative time" — it's "**sample 32 ticks earlier than the byte's base delay**."

**Why negative?** Because this bit's signal arrives **earlier** than the byte's average — its trace is a hair shorter (or its die a hair faster), so the sampling point must move **earlier** to catch it. Earlier = negative.

The unit is **PI ticks** (the phase interpolator's smallest step); 1 tCK ≈ 128 PI ticks, so -32 ≈ 0.25 tCK earlier, -16 ≈ 0.125 tCK.

### The base is DQS, the fine-tune is per bit

A byte's 8 DQ **share one DQS**; data is latched by the DQS edge. But the delay has **two levels**:

| Level | Register | Controls | Set by |
|---|---|---|---|
| **strobe** | `RxDqsDelay` | the whole byte's DQS moves together | 2D Centering |
| **per bit** | `RxDqsPBitDelay` / `RxDqsNBitDelay` | each DQ fine-tuned alone | per-bit de-skew |

So one DQ's **actual sampling instant** = `RxDqsDelay` (the shared base) + `RxDqsPBitDelay[this bit]` (the per-bit offset):

```
 sampling point (DQ0) = RxDqsDelay + RxDqsPBitDelay[0]
 sampling point (DQ1) = RxDqsDelay + RxDqsPBitDelay[1]
   ⋮
 sampling point (DQ7) = RxDqsDelay + RxDqsPBitDelay[7]
```

**Each bit has its own sampling point, but the base is the same DQS.** 2D Centering moves the base (everyone together); de-skew moves each bit's own offset (each alone).

"Move it earlier to catch it" = set this bit's `RxDqsPBitDelay` negative, so "base DQS edge + negative offset" lands on the instant its signal arrives. **You move not the DQS itself (that would drag the other 7), but this one bit's own offset.**

> One starting gun (DQS) fires once; eight runners go on it. The fast one is already out, the slow one still planted. De-skew doesn't change the gun — it measures each runner's stance, so each is sampled at *their own* eye center. Same gun, different stances.

## Four — the DLL: delay a full loop

The external CK enters the DRAM and fans out to every internal corner (the data-output latches, the read/write circuits). That clock-tree path takes time — CK reaches the internal circuits a hair late. Use that late clock raw, and the data output comes out late overall, misaligned to CK. So the DRAM embeds a **DLL (Delay Locked Loop)** to make the data output align to CK.

### Why does adding delay align it?

The DLL aligns by adding delay — but the clock tree is already late by Δt, so isn't adding more even later? No: the DLL adds **almost a full cycle** (T − Δt):

- the clock tree delays Δt (late);
- the DLL adds (T − Δt);
- total = T — a whole cycle — aligned to the *next* CK edge.

The clock repeats every T, so **late by a whole cycle = back to the same phase = aligned.** The DLL can't subtract delay (time doesn't run backward), so it adds up to the next cycle.

> A second hand reacting 5 s slow: the DLL makes it wait another 55 s, 60 s total — one full turn, back on the mark.

De-skew's result must stay steady under DLL wobble, via two mechanisms:

1. **DLL reset off/on, measured each** — the DRAM's DLL re-locks with timing drift.
2. **multiple iterations, take the tightest** — measure repeatedly, `AggregateResultsMin` keeps the worst margin.

> Guarantee the de-skew result passes in the *worst* case, not just a lucky one-time catch.

## Five — why the center, not the edge?

**First, fix the direction: left = early, right = late.** The passing range is a delay axis — the left end is the earliest passing point, the right end the latest:

```
  left edge (earliest)   center        right edge (latest)
        │                  │                 │
 early ───────────────────────────────────→ late (delay grows)
       -32                -16                0      ← N's example [-32, 0]
        └────────── eye width = 32 ──────────┘
```

So **the left edge is not "late" — it's the earliest end.**

**Why does the left edge only buffer "late"?** Because "buffer" means "which way can you still drift, and how far, before failing" — not where the edge sits:

- **drift earlier** (further left): already outside the range → immediate error → **zero buffer**;
- **drift later** (right): still inside → **buffer** (a full 32 wide).

> The left edge is already the earliest — one step earlier and you're out; but a whole stretch of safe room lies to the right. It can only protect "late," never "early."

**The center** is the midpoint of the two edges = `(left + right) / 2`. For N's [-32, 0]: center = (-32 + 0)/2 = **-16**. Standing at the center, 16 ticks early or 16 ticks late both still pass — equal, maximal buffer.

| Stand at | drift early | drift late | verdict |
|---|---|---|---|
| **left edge** | ✗ zero | ✓ (whole eye) | only protects late |
| **center** | ✓ (half eye) | ✓ (half eye) | protects both, max |
| **right edge** | ✓ (whole eye) | ✗ zero | only protects early |

> Stand at the bullseye and however the arrow (timing jitter) wobbles, you're still on target; stand at the edge and one slip is a miss.

### The edges are swept out, not specified

The left/right edges are **measured**, not decreed:

1. For each bit, try the delay value by value, early to late;
2. each value: CPGC writes a known pattern, reads back, compares — did this delay sample right (pass/fail);
3. **the earliest still-passing delay = the left edge**, the latest = the right edge.

> Groping the walls in a dark room: feel left until you touch a wall (left edge), feel right until a wall (right edge), and between them is where you can stand. The walls are felt, not drawn.

**Why is a narrow eye fragile?** Because the center's buffer is half the eye width: eye 32 → 16 of buffer each side (jitter up to 16 survives); eye 8 → only 4 each side (jitter past 4 flips a bit). The narrower the eye, the more fragile even the center.

**The edges themselves are also measured — they carry noise.** Left/right aren't exact constants; each measurement drifts a little with noise/temperature/voltage. That's why training iterates and takes the tightest — to survive the edges themselves wobbling.

## Six — the narrowest eye is the weakest bit

The source computes each DQ's eye width (right edge − left edge), finds the **narrowest** — the most fragile bit in the group:

| Threshold | Meaning | Result |
|---|---|---|
| **Critical** (`RxPerBitDeskewCriticalEyeWidth`) | eye below this = guaranteed misread | uncorrectable (MemDfe.c:1586) |
| **Small** (`RxPerBitDeskewSmallEyeWidth`) | eye passes but is narrow | excluded from outlier check (MemDfe.c:1593) |

margin ≈ 0 is **not "training must fail"** but **"barely passes, one touch and it blows"** — training "passes" but the margin is paper-thin, and any noise/temperature/voltage wobble flips a bit. "Trained" ≠ "ample margin."

Five root causes, in debug order:

1. **Trace/impedance** — too long, mismatched → reflection, loss, the eye nearly shut
2. **Crosstalk** — neighbors interfering
3. **Supply noise** — VDD rippling (power-integrity)
4. **Timing too tight** — frequency too high / timings squeezed, no settle time
5. **Hardware defect** — a bad die or trace on that bit

> The whole channel's ceiling is set by the narrowest-eye DQ, not the average.

## The golden rule

> 2D Centering draws all eight lanes on one starting line; per-bit de-skew staggers them, outer lanes ahead — each DQ has its own skew, so each deserves its own center.
