---
layout: mypost
title: "DRAM Detective · Case File 1.6: Silence the Lingering Echo (DFE Training)"
categories: [DDR5, Training]
lang: en
---

A singer hits a note — and its echo lingers in the room. The next note arrives while that echo is still ringing, and the two smear into each other.

**DFE (Decision Feedback Equalizer) is the sound engineer who knows exactly what was sung** — so it can cancel the lingering echo and let the next note through clean. DFE training is calibrating the tap coefficients (TapBias), so that echo gets cancelled just right.

<iframe src="/animations/dfe-echo-cancellation.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="DFE cancels the previous bit's lingering echo"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/dfe-echo-cancellation.html">open full screen ↗</a></p>

## One — what makes DFE clever

- The previous bit's echo hasn't decayed yet, so it muddies the current bit — that's **ISI, inter-symbol interference**.
- DFE's clever trick is that **it knows what the previous bit was decided to be**, so it can predict how big this echo is, and therefore cancel it precisely from the current signal.
- Formula: `y[n] = x[n] − (TapBias1·d[n−1] + TapBias2·d[n−2] + …)`

> The echo is DFE's job; reflections/echo-of-the-channel belong to ODT.

| Concept | Echo metaphor | Meaning |
|---|---|---|
| **tap** | remembers what was sung before | cancels the earlier bits' echo (tap1 cancels 1 bit back, tap2 cancels 2 bits back) |
| **TapBias** | how loud to cancel each echo | each tap's cancellation strength (coefficient) |

**tap and TapBias aren't two things — they're the position and strength of one thing: tap is the slot, TapBias is the number in the slot.**

Looking at the formula above: `d[n−1]` is what the previous bit was decided to be; **tap** decides which earlier bit to cancel (tap1 = 1 back, tap2 = 2 back); **TapBias** decides how much of that bit's echo to cancel. Together they form the `TapBias_k · d[n−k]` term.

- tap alone isn't enough: each earlier bit leaves a different-sized echo (nearer bits leave bigger echoes, farther bits smaller), so each tap needs its own coefficient deciding how much to cancel.
- The number of taps is fixed at 4; what training sweeps is the TapBias number.
- Correction: tap doesn't map to *every* bit — it only maps to the previous 4 bits (the echo decays fast, earlier ones are negligible).

In one line: **tap decides which earlier bit to cancel, TapBias decides how much — position and strength, two different things.**

How much DFE cancels matters a lot: **cancel too little** and the ISI stays, the eye is still narrow. **Cancel too much** and you over-compensate, squashing the eye the other way. So **DFE itself needs training** — training DFE is sweeping the tap coefficients and picking the setting with the widest eye.

## Two — both read and write need DFE

DFE training isn't a standalone step — it's **sandwiched between the read and write 2D Centering passes**:

```
Read  DQ-DQS Pre  DFE 2D Centering  ← coarsely set read-eye center first
Read  DQ-DQS DFE                    ← read DFE training ★
Read  DQ-DQS Post DFE 2D Centering  ← fine-tune read-eye center

Write DQ-DQS Pre  DFE 2D Centering  ← coarsely set write-eye center first
Write DQ-DQS DFE                    ← write DFE training ★
Buffer Write DFE Training           ← DB write DFE
Write DQ-DQS Post DFE 2D Centering  ← fine-tune write-eye center

Read  DQ-DQS DFE Late               ← read DFE late fine-sweep ★
Read  DQ-DQS Post DFE 2D Centering Late ← read-eye center late fine-tune
```

In **Pre 2D** (the 2D Centering before DFE), leave DFE alone and coarsely set the eye center first, because DFE reshapes the eye — if you tuned DFE first, you couldn't tell whether a crooked eye is a center problem or an echo problem. Once the center is set, sweep the taps to clean the echo, and the eye becomes wide and straight. In **Post 2D** (the 2D Centering after DFE), after DFE has reshaped the eye, the center may have shifted a bit, so fine-tune it once more.

## Three — where the echo is, there the knob is

Read and write are **two completely independent trainings**:

| | Read DFE (Rx) | Write DFE (Tx) |
|---|---|---|
| Where the echo is | host PHY receiving read data | DRAM receiving write data |
| Where the knob is | host PHY internal registers | DRAM's MR (written back to the die) |
| Training method | DRAM sends pattern, host tunes its own DFE | host writes pattern, DRAM tunes its own DFE |

1. Write DFE's tap coefficients are in the DRAM's MR, so every swept value must be written back via MRS; read DFE tunes locally in the host PHY.
2. Write DFE has a bigger, signed search space: write tap1 sweeps `-40 ~ +12` (signed, both directions to try), read tap1 sweeps `1 ~ 33` (positive, one direction).
3. Write DFE runs multiple rounds / Tabu search, read DFE is simpler.

> **Read DFE cancels the echo in the host, write DFE cancels it in the DRAM; one tunes in place, the other writes back over the bus — so write DFE is much slower.**

## Four — sweep taps, find the widest eye

- **First round**: full-range sweep, run each tap value once, measure the eye, record the widest setting.
- **Later rounds shrink the range** toward the optimum (like binary search / coarse-to-fine).

> Training is trying every cancellation possibility, finding the one that cancels cleanest and opens the eye widest.

The sweep range is **an empirical constant hard-coded by the SI engineer**.

| DFE type | tap1 sweep range | Note |
|---|---|---|
| Write DFE (Tx) | `-40` to `+12` | signed — echo can point up or down |
| Read DFE (Rx) | `1` to `33` | positive — echo direction fixed |
| DB DFE | `-50` to `+15` | signed |

The unit is a **code value**, a step on the DFE bias DAC — bigger code = stronger cancellation.

This thing isn't computed, it's swept. The ISI echo depends on traces, crosstalk, the die, temperature and voltage — computable but too complex and it drifts; in engineering, sweeping once and measuring the eye beats computing.

What's a code value?

- A code value is a dimensionless step number, not volts or amps.
- Physically the code drives a DAC (digital-to-analog converter), which turns it into a real analog offset (voltage or current) that subtracts that much echo from the received signal. One code = one DAC step.
- How much voltage/current one step equals depends on the DRAM or PHY datasheet (the DAC's LSB); the training code doesn't care — it only sweeps codes, measures the eye, finds the max, without converting to voltage.
- Write DFE's code is signed (`-40` to `+12`), because the echo can point up or down: the MR's sign bit sets the direction, the bias bits set the magnitude — together, which direction to cancel and how much.

Every DQ has its own echo, so every DQ needs its own set of tap coefficients.

**How many sets of taps? First separate tap from pin**

- **tap** is the "which earlier bit" slot (tap1 = 1 back, tap2 = 2 back, usually 4 taps); **pin** is a physical DQ/DM pin. It's not one tap per bit — it's one set of taps per pin (a set is 4 taps).
- So for 64 DQ + 16 ECC = 80 pins, do we need 80 sets? It splits by read/write:
  - **Write DFE (DRAM side)**: not 80. Equalization happens at the DRAM receiver, and each die only sees its own pins. An x8 die is 8 DQ + 1 DM = 9 pins, an x16 die is 16 DQ + 2 DM = 18 pins. The channel's 80 bits are sliced across many dies, each tuning its own: an ECC DIMM with x8 dies is 10 dies (8 for 64 DQ + 2 for 16 ECC), each programming its own 9 sets — 90 sets total, not one 80-entry table.
  - **Read DFE (host side)**: the host receives all 64 DQ + 16 ECC (plus DQS) at once, so read DFE covers the full bus width; but whether the host PHY does it per-pin or per-byte-lane is a PHY implementation detail.

> **Write DFE counts by the DRAM die's own pins (x8 = 9, x16 = 18), read DFE counts by the full bus width the host receives (about 80 plus DQS) — neither is a simple "80 taps".**

## Five — how to diagnose when cancellation fails

Three failure points are three different diseases, not a severity ladder:

| Failure | Warning (log keyword) | What dimension | Criterion |
|---|---|---|---|
| eye width too small | `"eye width is too small, current margin: %d minimum: %d"` | absolute margin enough | worst bit's eye < absolute danger line |
| Eye Width Delta | `"Eye Width Delta = %d, Minimum criteria %d"` | uniform across bits | some bit far below average (outlier) |
| TapBias all zero | ("zero margin" string TBD) | can DFE cancel ISI | swept every tap, eye still 0 |

**Medical metaphor**: myopia (eye narrow), strabismus (one bit off-axis), cataract (echo can't be cancelled) — three different medical reports, not three stages of one disease.

**Diagnostic guide (what to do when you see a warning)**:
- Bisect "whole eye narrow (board SI)" vs "one bit narrow (that bit)"
- Inspect that bit's trace directly (bit-level)
- Swap the part (DIMM) / fix SI (tune CTLE/ODT)
