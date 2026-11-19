---
layout: mypost
title: "DRAM Detective · Case File 0.8: The Label, the Ruler, and the Subtraction"
categories: [DDR5, Training]
lang: en
---

Timing numbers don't all come from one place: some are read from the SPD and fixed, some are measured by training after power-up, and some are computed from a spec baseline plus training results.

Know which hand manages each number, and you'll know which knob training is turning — and why.

<iframe src="/animations/timing-sources.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The label, the ruler, and the subtraction"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/timing-sources.html">open full screen ↗</a></p>

## One — three kinds of numbers, three origins

| Origin | When it's settled | Typical | Written by |
| --- | --- | --- | --- |
| configured (static) | written into the SPD at the factory / looked up from spec, read at power-up | tCL / tRCD / tRP / tWR / tRFC / tREFI / tCCD_L | the DIMM vendor |
| measured (dynamic) | measured by training after power-up | DQS/DQ delays, sample phase, gate timing, Vref, RTT, DFE taps | the MRC training |
| computed (mixed) | after training, from a baseline plus results | turnaround, rank-to-rank delta | the MRC's arithmetic |

The label, the ruler, the subtraction. Configured numbers are the spec stamped on the nameplate — copy them. Measured numbers come from a tape measure on-site. Computed numbers are two measurements subtracted — no single ruler can read that gap directly.

## Two — the full map

### Configured: fixed at the factory, never trained

These record the die's own physics. The vendor measures them, writes them into the SPD (or they're looked up from the spec), and they're read into the controller at power-up and never touched. The reason they aren't trained is simple: they don't depend on how the board is routed, only on the die's speed and internal circuitry. Later chapters barely mention them again — training turns the "measured" category — so here's the full story in one place.

The four waits in the read/write flow (the four steps from Case 0.0):

- **tRCD** (RAS to CAS): after activate, you wait for the sense amplifier to read out, amplify, and settle the whole row before a read or write can go out. So activate and read/write are separated by one tRCD.
- **tCL** (CAS Latency): after the read command, the die is still busy a while before data reaches the bus; this column-to-data delay is CL.
- **tWR** (Write Recovery): after the last write datum, the data still has to drain from the bus into the capacitor and settle before the row can be precharged and closed.
- **tRP** (Precharge): precharge writes the sense amplifier back, resets it, and clears it, and only then can the same row be activated again.

The two refresh ones:

- **tRFC** (Refresh Cycle): one refresh equals activating a row, reading it, writing it back, and precharging again — the whole flow's time.
- **tREFI** (Refresh Interval): the capacitor leaks, and room temperature requires all rows refreshed within 64 ms, so on average one refresh every tREFI (about 7.8 µs).

The rest, all about commands fired back to back:

- **tCCD_L / tCCD_S** (CAS to CAS): the minimum gap between two read or write commands. tCCD_S is across different bank groups, tCCD_L within the same bank group. Case 1.13's turnaround uses tCCD as its baseline, then adds the rank delta and a guardband.
- **tCWL** (CAS Write Latency): after a write command, data reaches the bus a few beats later — the write-direction CL.
- **tWTR** (Write to Read): from the last write datum to being able to issue a read — the bus has to about-face.
- **tRTP** (Read to Precharge): from the last read datum to being able to precharge — the data must first be drained clean off the bus.
- **tRRD** (Row to Row): the minimum gap between two activate commands (different banks). tRRD_S across different bank groups, tRRD_L within the same bank group.
- **tFAW** (Four Activate Window): at most four activate commands in one window, capping the peak current.
- **tRAS** (Row Active): a row must stay active at least this long for the data to settle before precharge is allowed.
- **tRC** (Row Cycle): activate to the next activate of the same row, equal to tRAS plus tRP.

And one that's neither in the SPD nor given by training, only looked up from a table:

- **tDQSCK / tDQSCKi** (DQS to CK): the read-direction DQS-to-CK inherent position (tDQSCK) and jitter window (tDQSCKi), a die-inherent property. Case 1.4 uses it when it breaks down read latency.

### Measured: measured after power-up

This is where training actually works — numbers nobody could know before the board was assembled.

| Knob | What it trains |
| --- | --- |
| CA de-skew | command/address alignment |
| gate timing (RcvEn) | when the DQS gate opens |
| tDQSS | write DQS aligned to CK |
| DQS delay + sample phase | read DQS arrival + sample point |
| per-bit DQ delay | per-bit de-skew |
| DFE taps | the ISI-cancelling tap coefficients |
| RTT / ODT setting | termination impedance |
| CTLE / DXC | treble boost + crosstalk cancellation |
| duty-cycle DCC | fixing DQS/CLK duty cycle |
| Ron / Comp | transmit-side drive impedance |
| slew + Tx EQ | edge slope + pre-emphasis |
| Vref | the decision line |
| rank-to-rank switch offset | the cross-rank switching offset |

### Computed: baseline and training results put together

This kind has no single true value to measure — only arithmetic.

| Quantity | How it's computed |
| --- | --- |
| Turnaround (Wr2Wr / Rd2Rd switch gap) | JEDEC tCCD baseline + first-to-last rank timing delta + guardband |
| delta (first-to-last rank timing gap) | max minus min of the timing settings across all ranks |

## Three — why three kinds at all

The configured kind needs no training. tRCD, tCL, tRP are set by the die's speed and internal circuitry; the board's routing, temperature, and voltage can't change them. The vendor measured them and wrote them into the SPD — reading them at power-up is enough, no point measuring again.

The measured kind can only be measured on-site. Trace length, temperature, voltage, and die batch all vary, and — worse — the combination of "this die on this board" is something nobody knows at the factory. So after power-up, training drives traffic, watches the feedback, and measures each alignment value one by one.

The computed kind has no choice. It wants a relationship between two quantities — turnaround is the difference between the first and last rank's timing, and there's no register holding a "true value" waiting to be read. You subtract the two trained numbers, then stack the spec baseline tCCD and a guardband on top.

So there's a crude but useful trick for reading a training log: first recognize which knob in the "measured" category this step is turning, and you know which line it's serving. Training spends all its effort on that category — the configured and computed kinds are the two ends, nailed down.

**Copy the label, measure with the ruler, subtract the gap — training always turns the ruler.**
