---
layout: mypost
title: "DRAM Detective · Case File 1.10: A Well-Matched Pair (Comp Impedance Compensation)"
categories: [DDR5, Training]
lang: en
---

The transmit driver's output impedance must be a well-matched pair with the trace.

Comp, short for Compensation, calibrates the transmit driver's output impedance so it matches the trace's characteristic impedance Z0 — then the signal pushes out without reflecting at the interface.

<iframe src="/animations/comp-impedance-match.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Match the driver to the trace — no reflection"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/comp-impedance-match.html">open full screen ↗</a></p>

First, two words that are easy to mix up: **compensation is the goal, calibration is the means**. Compensation is what we want — cancel the process/temperature/voltage drift so the impedance returns to Z0. Calibration is how we get there — measure the error against the 240Ω standard, then adjust back to the standard. ZQ calibration (measure + adjust) is the means that achieves the impedance compensation (cancel the drift).

## One — Comp: transmit-side impedance compensation

A signal gets pushed onto the trace by an output driver circuit. When the driver pushes, it presents an output impedance. Ideally this impedance matches the trace's characteristic impedance Z0 exactly (say both 40Ω), so the signal does not reflect at the interface.

### Comp and ODT are two sides of one coin

Comp is the transmit side, ODT is the receive side of impedance matching — two faces of one thing:

| Direction | Transmit compensation Comp | Receive termination ODT |
|---|---|---|
| Read (DRAM→host) | DRAM Ron | host ODT |
| Write (host→DRAM) | host Write Drive Strength | DRAM Rtt |

Four points map to four training steps:

| Point | Training step |
|---|---|
| DRAM Ron | DIMM Ron Training |
| host Write DS | Write Drive Strength |
| host ODT | Read DQ/DQS ODT Training |
| DRAM Rtt | DIMM ODT Training |

ODT is the receive side (host ODT + DRAM Rtt) — letting the other side hear clearly; Comp is the transmit side (DRAM Ron + host Write DS) — letting yourself shout accurately. On the DRAM side it is called Ron, on the host side Drive Strength — both are the transmit driver's output impedance, just two names for the two chips on the two ends.

- Comp (compensation) is the transmit-side concept: DRAM Ron on read, host Write Drive Strength on write.
- ODT (termination) is the receive-side concept: host ODT on read, DRAM Rtt on write.

So Comp only covers the two transmit-side directions. Concept versus concept is Comp versus ODT; value versus value is Ron/DS versus Rtt.

## Two — why compensate

Comp and ODT are handled separately because they do opposite jobs. Comp makes the driver's output impedance equal the trace Z0, so the signal does not reflect on the way out; ODT makes the termination equal Z0, so the signal does not bounce on arrival. Both ends must match, but they are two different devices.

The signal travels down the trace; when it hits an impedance discontinuity it reflects back — like water splashing when a pipe suddenly changes diameter. If the driver's output impedance does not match Z0, the signal reflects right at the driver and disturbs the following signal, so the driver impedance must equal Z0.

The driver's impedance is built from transistors; process, temperature and voltage all make it drift. So every power-on cycle, a 240Ω precision reference resistor is used as the standard, and the driver impedance is calibrated to the target — that is ZQ Calibration.

Calibration is like your wall clock drifting — fast 2 minutes today, slow 3 minutes tomorrow, because temperature, humidity and battery voltage all change. Calibration is turning on the radio to hear the time signal, then reaching over to move the hands to the standard time. The DRAM is the same: its output impedance drifts with PVT, and ZQ calibration uses the 240Ω standard resistor to pull the impedance back.

Why is a full circle 360 degrees and not 100? Because 360 is a super-divisible number — divisible by 2, 3, 4, 5, 6, 8, 9, 10, 12 — so half a circle 180, a right angle 90, an equilateral triangle 60, a pentagon 72 are all whole degrees, super convenient for splitting angles. 240Ω is the same trick — it is divisible by 2, 3, 4, 5, 6, 8, so 40 (÷6), 48 (÷5), 60 (÷4), 80 (÷3), 120 (÷2), 240 (÷1) are all whole divisions of 240. 240 is a highly composite number that divides cleanly — not picked at random.

## Three — how to compensate

**DRAM-side ZQ calibration**: the host sends an MPC command to start calibration; the DRAM uses the 240Ω reference resistor to calibrate its output driver impedance; when done, the host sends ZQCal Latch to lock the result. After calibration the impedance matches Z0, reflection minimized.

MPC (Multi-Purpose Command) is the DRAM's special operation command with no dedicated pins — it reuses the CA bus carrying an 8-bit opcode to select the operation. ZQCal Start and ZQCal Latch are two of its functions.

**Host-side Comp Optimization**: when the host transmits on the write path, its output driver impedance must also match Z0. The host's Comp Optimization Training tunes the output impedance to match Z0.

**Training (Comp/Ron)**: the same recipe as before — sweep settings, measure the eye, pick the setting with the widest eye.

| Step | What it tunes |
|---|---|
| DIMM Ron Training | the DRAM die's output driver impedance Ron |
| Write Drive Strength / Write Equalization | write-direction drive strength / equalization |
| Comp Optimization Training | compensation optimization (host-side Rcomp target) |

## Four — how to diagnose

Margin check fails: after sweeping every setting, the eye is still too small.

- All ranks and lanes bad → impedance severely mismatched / trace impedance design problem / power anomaly.
- One lane bad → that driver's process deviation.
- One rank bad → die / socket / gold fingers.

**match the driver's impedance to the trace, and the signal pushes out without splashing.**
