---
layout: mypost
title: "DRAM Detective · Case File 1.9: Level the Day and Night (Duty Cycle)"
categories: [DDR5, Training]
lang: en
---

A clock's day and night must be the same length.

Duty Cycle training levels the clock CK and the strobe DQS to 50/50 high/low. DDR is double data rate — it samples on both the rising and falling edges — so if one half is short, that half's data eye shrinks and the timing margin is eaten away.

<iframe src="/animations/duty-cycle-day-night.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Level the day and night — duty cycle must be 50/50"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/duty-cycle-day-night.html">open full screen ↗</a></p>

## One — what is duty cycle

A clock is a square wave: high, low, high, low, repeating. Duty cycle is the share of the period spent high — 50/50 is ideal.

DDR5 is double data rate: it samples twice per clock period. The **rising edge** (low→high) captures one bit; the **falling edge** (high→low) captures another. One clock period carries two bits — same clock frequency, doubled bandwidth. That is DDR's edge over SDR, but the price is that both edges must be steady, so the duty cycle must be even.

A period is cut in half: **the high half serves the rising-edge bit, the low half serves the falling-edge bit**. If the duty cycle skews to 60/40, the 40% half (the low side) gets a smaller window than it had at 50% — a smaller window means less timing margin and more errors.

## Two — why it drifts, and how DQS relates to CK

The clock travels from the host through several stations before reaching the DRAM die:

1. **The host PHY's clock tree** generates CK and sends it out.
2. **The RCD (Registering Clock Driver)** receives CK, regenerates it, and fans it out to every DRAM.
3. **Each DRAM's internal clock tree** divides it further to the subchannels.

Process variation, trace length, temperature, voltage — every step makes the high/low pulses unequal. And every die, every board, every power-on is different, so it must be measured and tuned on the spot, never written once at the factory.

On read, **DQS is the strobe the DRAM die itself sends back**; the die's DQS generation circuit has its own duty-cycle distortion, tied to the manufacturing process. So read DQS needs its own duty-cycle trim.

**DQS vs CK** — not locked together, but two cooperating roles:

- **CK is the system-synchronous global clock**: every die, every command, every internal operation uses CK as the timebase. Each die DLL-locks to CK to make its internal clock.
- **DQS is the source-synchronous companion strobe**: it travels together with its 8 DQ bits, so the receiver samples DQ with DQS, not CK.

Why not sample data with CK: CK is a fly-by trace (a bus stopping at every stop), so it reaches each die at a different time; DQS rides the same point-to-point trace as DQ, so it is naturally delayed the same as the data. At speed, sampling with CK misaligns the edges; sampling with DQS keeps the edge glued to the data.

CK (global) → each DRAM's DLL → multi-phase internal clocks (0°/90°/180°/270°) → DQS clock tree → read DQS out, sent back with the DQ.

So DQS is derived from CK through the DLL, but it does not need to align to CK's edges — only to its own DQ. They just need a trained, known relationship:

| Direction | Who sends DQS | How it relates to CK |
|---|---|---|
| Write | host | Write Leveling aligns the write DQS to each DRAM's CK (compensating fly-by arrival skew) |
| Read | DRAM | host samples directly with DQS (source-synchronous); it only trains tDQSCK to know where DQS sits relative to CK |

Write must align, because the DRAM ultimately samples write data on its internal clock (CK-derived); the write DQS has to land inside CK's correct window. Read need not align — the host samples with DQS, and CK only gives a coarse reference (tDQSCK: DQS arrives earlier than CK).

The DCA sits before the DQS clock tree, adjusting the internal-clock phases derived from CK, so it can change DQS's duty cycle — but it only changes the high/low ratio, never DQS's position relative to CK (tDQSCK stays put).

## Three — four correction points

| Correction point | What it levels | Who tunes it |
|---|---|---|
| RCD DCA/DCK Duty Cycle Training | CK's duty cycle | RCD (regenerates and fans out CK) |
| DB Duty Cycle Training | DQS duty cycle (LRDIMM only) | Data Buffer |
| Read DQ-DQS Duty Cycle Adjuster | read DQS duty cycle | the DRAM die's own DCA |
| Write DQS duty-cycle correction | write DQS duty cycle | host TX (no separate log step — folded into Write DQ-DQS training) |

Four points along the signal chain, each one re-generating a clock or a strobe:

- **RCD** takes CK in and sends it out again — that round trip adds duty-cycle distortion, so correct the RCD itself.
- **DB (Data Buffer)** exists on LRDIMM; DQS passes through it and gets distorted.
- **The DRAM die** sends read DQS; the die has its own distortion, corrected by its own DCA.
- **Write DQS comes from the host**, so the write direction's duty cycle is corrected by the host TX.

Duty-cycle training is not one thing — **whoever re-generates a clock or a strobe gets corrected**.

## Four — how to tune

DDR5 dies carry a built-in **DCA (Duty Cycle Adjuster)** that levels DQS's high/low halves. JEDEC defines the Mode Registers for it. MR42 reports three DCA types:

- `DcaNotSupported` — not supported
- `DcaSingleOrTwoPhase` — single/two-phase correction
- `DcaFourPhase` — four-phase correction (finer, splits the period into four segments)

The higher the type, the finer the granularity, the closer to 50/50.

Training follows the same recipe as DFE and ODT: have the DRAM send a known pattern → turn the DCA knob to step 1 → measure the eye → sweep step by step → pick the step with the widest eye, closest to 50/50.

The spec also defines a die-built ruler: the **DQS Interval Oscillator**. The die's DQS oscillator counts once during high and once during low; the difference between the two counts is the duty-cycle error — no external instrument, the die measures itself.

The write direction is another story: write DQS comes from the host, so the host corrects itself. Training sweeps the Tx DQ delay — write a known pattern, read it back, and find where capture starts working (one edge) and starts failing again (the other); the window in between is the write data eye. `p` is the positive/late edge, `n` the negative/early edge, eye width = p − n.

`TxDqDlyTemp` is the current DQ bit transmit delay, the reference point. LeftEdge = p − TxDqDlyTemp (how far the late side is), RightEdge = TxDqDlyTemp − n (how far the early side is). Equal on both sides means centered; unequal means off, and half the difference is the correction.

Even/odd UI are measured separately because DDR is double data rate: even bits are sampled on the DQS rising edge, odd bits on the falling edge. Duty-cycle distortion is precisely a wrong relative position of these two edges, so the even and odd eyes shift independently. Only by measuring them separately can you see the fingerprint of "even-eye center ≠ odd-eye center"; merged, the offsets cancel and the distortion hides.

Read direction uses the DRAM's DCA register; write direction uses the host's TX phase offset — whoever sends the strobe corrects it. Write has no separate step because it is folded into Write DQ-DQS Coarse/TCO training.

### Why the DCA can change the duty cycle

DQS is not an independently generated line — its rising and falling edges come from two different phases of the DRAM's internal clock tree:

| Internal clock scheme | Phases | What each serves |
|---|---|---|
| single/two-phase | 0° (rising, fixed) + 180° (falling, adjustable) | 0° serves even bits, 180° serves odd bits |
| four-phase | ICLK(0°) / QCLK(90°) / IBCLK(180°) / QBCLK(270°) | even/odd duty cycle adjustable separately |

The 0° clock's rising edge is the reference edge; the 180° clock is adjusted relative to 0°. The 0° rising edge serves even bits, the 180° serves odd bits. So the DCA's action is: move the 180° clock (the DQS falling edge) earlier or later, while the 0° clock (the rising edge) stays put.

- 180° later → falling edge later → high time longer → duty cycle larger
- 180° earlier → falling edge earlier → high time shorter → duty cycle smaller
- positive DCA = larger duty cycle, negative DCA = smaller duty cycle

Global −7 to +7 steps (MR43/44), roughly 28–56 ps per step, and non-linear.

It is like clocking in at a fixed 9 a.m. every day; clocking out is the DCA knob. Move clock-out earlier or later and the workday lengthens or shortens — but the clock-in time (tDQSCK) stays 9 a.m.

The DCA's internal clocks come from a DLL and are multi-phase (single/two-phase 0°/180°, four-phase 0°/90°/180°/270°); the DCA moves these phases. The standard circuit for moving a clock phase is a **phase interpolator**: two adjacent phases each drive a current into one output node; the two currents' weights are set by the DCA code (step −7 to +7, the sign bit picks the direction); whichever current is stronger pulls the output edge toward that phase. Change the weight and the edge slides between the two phases — the falling edge moves, the high time changes, the duty cycle changes.

The PI's current-weight-to-phase relationship is not perfectly linear (transistors saturating, parasitic capacitance), so each step's actual shift varies. Training does not care — it sweeps one step, measures the eye, picks the best, no exact math.

## Five — how to diagnose

Sweep every DCA step and some edge's eye is still too small — that is a failed duty-cycle correction.

- **All ranks and all lanes bad** → the clock path's duty-cycle distortion is too severe (RCD / trace / power), or temperature anomalies amplify the drift.
- **One lane bad** → that die's DQS generation circuit.
- **One rank bad** → that rank's die / socket / gold-finger contact.

**Golden rule: a clock that samples on both edges must level its day and night — shorten one half and that half's eye shrinks.**
