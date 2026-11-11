---
layout: mypost
title: "DRAM Detective · Case File 1.8: Boost the Treble, Silence the Crosstalk (CTLE / DXC)"
categories: [DDR5, Training]
lang: en
---

A signal that travels a long line turns dull — the high frequencies get eaten away. And neighboring signals leak into each other.

CTLE/DXC training tunes two signal-restorers on the host PHY's **read** path: **CTLE is the treble knob** that restores the high frequencies the line ate, and **DXC is the anti-noise wave** that cancels the crosstalk leaking in from the neighbors.

<iframe src="/animations/ctle-dxc-restorers.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="CTLE restores the treble, DXC cancels the crosstalk"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/ctle-dxc-restorers.html">open full screen ↗</a></p>

## One — a long line turns the signal dull

A trace is a low-pass filter: the higher the frequency, the larger the loss — officially, **high-frequency loss**. When the signal's high-frequency content (the sharp edges, the fast transitions) is eaten away, the eye rounds off and shrinks.

Data leaves the DRAM, travels the whole trace to the host, and arrives with its treble gone.

**CTLE (Continuous-Time Linear Equalizer)** is the host PHY's read-path treble knob. It amplifies the high frequencies, restores the eaten treble, and sharpens the eye again. Training sweeps the CTLE compensation to find the setting with the widest eye — the same recipe as DFE sweeping TapBias and ODT sweeping Rtt.

The training process (turning the treble knob):

1. Have the DRAM send back a pattern the host already knows.
2. The host turns the knob to setting 1, receives once, measures the eye.
3. Sweep setting by setting, measuring each time.
4. Pick the setting with the widest eye.

Neighboring DQ lines sit close together, so signals leak into each other (capacitive and inductive coupling) — a signal on one line bleeds into its neighbor. That's **crosstalk**. When all DQ lines transmit at once, the neighbors interfere the most, so the read direction suffers the most.

**DXC (Cross-Talk Cancellation)** is the host PHY's anti-noise wave: it subtracts the crosstalk component out of the neighbor's signal, leaving only the line's own signal. Training sweeps the DXC cancellation to find the setting with the least crosstalk.

The training process (turning the anti-noise wave):

1. Have all DQ lines send data at once, deliberately maximizing the crosstalk.
2. The host turns the anti-noise to setting 1, receives once, measures the noise.
3. Sweep setting by setting, measuring each time.
4. Pick the setting with the least noise (the widest eye).

**CTLE/DXC have no JEDEC clause** — they are PHY implementations, not DRAM-spec definitions.

## Two — why read-direction only

The loss is bidirectional: the trace eats treble in both the read and write directions. But where the treble is restored differs:

| Direction | Who restores treble | Where | Who cancels ISI |
|---|---|---|---|
| Read (DRAM → host) | host's CTLE | receiver (restore after arrival) | host's DFE |
| Write (host → DRAM) | host's Tx EQ (pre-emphasis) | transmitter (shout louder up front, so the line's eating leaves it flat) | DRAM's DFE |

Why doesn't the write direction use a DRAM-side CTLE? CTLE is an analog circuit — it takes area and power; the DRAM die is a small, cost-sensitive chip that can't afford CTLE (it only carries DFE). The host is a big ASIC that can afford the full CTLE + DFE + Tx EQ set. So the write direction's high-frequency loss is moved to the host's transmit pre-emphasis, and the DRAM only uses a cheap DFE to hold down ISI.

CTLE and Tx EQ restore the high frequencies (fix the dullness); DFE cancels ISI (fixes the lingering tail) — one handles loss, the other handles the tail, two different jobs.

The trace eats treble in both directions, but on read the host restores it at the ear (CTLE), and on write the host shouts louder up front (Tx EQ pre-emphasis) — restoring the treble is always the big ASIC's job.

## Three — how to diagnose when it fails

CTLE/DXC training fails the same way ODT does: after sweeping every setting of treble boost and crosstalk cancellation, the best setting's read eye is still too small.

After training, a margin check-up measures the read direction's voltage margin, timing margin, and gating margin; if the margin drops to the retrain threshold, it errors out.

- If the whole eye is small (all ranks, all lanes): the loss or crosstalk is too severe — CTLE's treble boost and DXC's cancellation can't save it, and the eye has already closed. No fix but lower the frequency or rework the board.
- If one lane's eye is small: that DQ trace's problem (a crosstalk source or especially large loss) — chase that bit.
- If one rank's eye is small: that rank's problem (gold fingers, socket, die).

**Golden rule: the trace eats the treble, CTLE restores it; neighbors talk through the wall, DXC cancels it — the read path's two restorers bring the eye back big and sharp.**
