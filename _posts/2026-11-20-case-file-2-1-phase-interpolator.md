---
layout: mypost
title: "DRAM Detective · Case File 2.1: The Seesaw"
categories: [DDR5, Training]
lang: en
---

The phase interpolator (PI) makes the sub-cycle phase — the finest tick training uses to move timing.

I and Q, two quadrature clocks, sit on the two ends of a seesaw: current is the weight, and the balance point is the new phase. One clock is split into a power-of-two number of fine PI ticks, and training nudges DQS left and right across those ticks.

<iframe src="/animations/the-seesaw.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The seesaw — two arrows, one balance point"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/the-seesaw.html">open full screen ↗</a></p>

## One — the body of the tick

Every training step, in the end, moves some delay by N ticks. What circuit is this tick, and how fine is it?

Total delay splits into two parts, handled by two different circuits:

| Circuit | Handles | Unit | Metaphor |
| --- | --- | --- | --- |
| Delay line / cycle counter | whole-cycle delay (which cycle) | one cycle | the centimeter (which notch) |
| PI (phase interpolator) | sub-cycle phase (where inside the cycle) | PI tick (a tiny fraction of a cycle) | the millimeter (inside a notch) |

Don't mistake the PI for a delay circuit. A delay makes the signal arrive later — a shift along the time axis, still the same signal. The PI takes two clocks at 0° and 90° and mixes them by weight, making a new clock whose phase sits in between — a shift along the angle axis. Both move the timing a bit, but the mechanism is entirely different — that's why there are two circuits.

One centimeter is many millimeters. Training says "move DQS right by 3 ticks", and in hardware that means changing the PI's interpolation weights so the phase moves exactly that many fine ticks of a cycle.

## Two — two arrows added together

How does the PI squeeze out intermediate phases from two clocks? The standard story is vector addition:

- **I clock**: an arrow pointing right (0°).
- **Q clock**: an arrow pointing up (90°).
- Weight each one (the current), add the two vectors, and the resulting arrow points at the output phase.

<iframe src="/animations/phase-vector-sum.html" width="100%" height="560" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Two arrows, one phase — the vector sum"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/phase-vector-sum.html">open full screen ↗</a></p>

Core formula: `output = cos(θ)·I + sin(θ)·Q`, with θ the target phase.

- All weight to I, none to Q → the arrow points right (0°).
- All weight to Q, none to I → the arrow points up (90°).
- Half each → 45°.
- Change the weights and the sum arrow rotates between I and Q. One tick is a small rotation.

One detail that bites: for evenly-spaced phases, the weights must follow sine/cosine, not a linear code. With sinusoidal weights the vector traces a circle — even phase steps. With a linear `α·I + (1−α)·Q` the trace is a diamond — sparse at the edges, dense in the middle, an uneven ruler.

The seesaw is this same picture drawn differently: I/Q sit on the two ends, current is the weight, and the balance point is the phase. The vector picture is the 2-D view (arrow direction); the seesaw is the 1-D view (balance position). Same math.

## Three — the queue at the gate: the whole-cycle part

The delay line is a chain of buffers, like a row of gates. The signal slows a little at each gate; tapping out at the Nth gate is a delay of N notches. It handles the coarse part (centimeters), the PI handles the fine part (millimeters); the two together cover both a wide range and fine precision.

Total delay = whole cycles × (ticks per cycle) + sub-cycle ticks. The "round-trip delay + receive-enable delay" that keeps showing up in training are exactly these two numbers: round-trip is the whole cycle, receive-enable is the sub-cycle tick.

## Four — a few unavoidable whys

Why not just build many real clocks? That needs one clock generator per phase — too expensive. Interpolation needs only two quadrature clocks plus one current DAC to produce the intermediate phases. Trading "two clocks plus weights" for "many phases" is pure circuit savings.

Why double the resolution from the older generation? DDR5 runs faster, so one UI is a fraction of a nanosecond. The same absolute error takes a bigger share of a narrower UI, so the ruler must be finer to land precisely and keep the eye margin. The tick count is a power of two — the binary-friendly choice — and the newer generation simply moves to the next power.

Why two separate circuits at all? One tracks "which cycle" — wide but coarse; the other tracks "where inside the cycle" — narrow but fine. Each does one job, and together they're both long and fine — one ruler can't do both.

## Five — training turns exactly this

| Training | What it moves |
| --- | --- |
| 1.2 Receive Enable | the receive-enable delay (sub-cycle tick) |
| 1.3 Write Leveling | the transmit DQS delay (sub-cycle tick) |
| 1.4 2D Centering | the read DQS delay (sub-cycle tick) |
| 1.5 per-bit de-skew | the per-bit read DQS delay (sub-cycle tick) |

One tick is one small nudge of the phase. All that training is doing, knob by knob, is moving that balance point on the seesaw.

**I and Q sit on the two ends of a seesaw, current as the weight, the balance point as the phase — training moves that balance point.**
