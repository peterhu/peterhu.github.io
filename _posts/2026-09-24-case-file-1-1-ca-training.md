---
layout: mypost
title: "DRAM Detective · Case File 1.1: A Miss Is As Good As a Mile (CA Training)"
categories: [DDR5, Training]
lang: en
---

CA training is tuning the **RCD messenger's loudspeaker** so the general's order lands exactly when every soldier's attention is sharpest. The order has to reach everyone at once, so the usable eye width is the **intersection** of every soldier's hearing window.

## Why CA goes first, and alone

We've covered the ground before; the short version:

1. **CA has no doorbell.** It rides the global CK alone, while DQ has DQS riding along. No doorbell means nothing backs up its timing — it must be trained on its own.
2. **CA goes through the RCD.** The order passes through the messenger before reaching the soldiers; one more relay is one more delay and one more distortion.

> If the order gets garbled, the whole army mishears: a CA failure means **no boot / POST hang**.

## Two gates: front gate trains the messenger, back gate trains the soldiers

The order travels two hops: **general → messenger → soldier**. So training splits into two stages, one per hop.

**Front gate (hop 1: general→messenger)** — the messenger is the one under test.

The general shouts an order; the messenger **repeats it back** (over the ALERT_n line, a loopback) → the general compares what he said with what the messenger repeated, sees whether the messenger misheard, and sweeps the messenger's "receiving" delay to find the point where the messenger hears best. (DCS/CS/DCA do this job.)

> The general tests the messenger's ears by making him repeat the order.

**Back gate (hop 2: messenger→soldier)** — the soldiers are under test, the messenger steps aside.

The messenger **opens the door and steps aside** (Pass-Through) → the general's order reaches the soldiers **directly** → each soldier reports back in training mode whether it heard, and the general sweeps the messenger's "shouting" delay (QCA output delay) to find the point where every soldier hears best. (QCS/QCA do this job.)

> The feedback comes from the soldiers, but the knob is on the messenger: the messenger→soldier hop's delay is the messenger's output delay (the knob the general can turn) plus the soldiers' receive (fixed, out of the general's reach). So the general turns the messenger's voice, asks the soldiers "heard it?", and finds the best voice from their feedback — like turning a radio's volume knob and asking the listener if it's clear.

DCA vs QCA bit-width (JESD82-513 §3.4.1): the front side is **7-bit double-data-rate + parity**, the back side is **14-bit single-data-rate without parity** — the messenger turns 7 double-shift turnstiles into 14 single-shift assembly lines.

**Why the bit-width differs** — information is conserved, but pin cost trades off against timing stability:

```
host side (DCA):  7 lines × double-data-rate (2 per line per cycle) = 14 bit/cycle
DRAM side (QCA): 14 lines × single-data-rate (1 per line per cycle) = 14 bit/cycle
```

- **Host side, 7 double-rate lines**: controller pins are scarce (many channels, many DIMMs); double-rate packs 14 bits into 7 lines, saving half the pins and traces.
- **DRAM side, 14 single-rate lines**: the RCD fans the command out to a pile of dies (heavy load); single-rate timing is looser, easier to align, steadier — and a die has pins to spare for 14 lines.
- **The RCD does a serial-to-parallel conversion**: it expands the host's 7 double-rate lines (fast, pin-frugal) into the DRAM's 14 single-rate lines (slow, stable) — fewer turnstiles but double-shift, more assembly lines but single-shift.
- **Parity** (spec doesn't spell out the why; inferred): host→RCD is a long trace, error-prone, so parity is added to catch errors; RCD→DRAM is short and re-driven clean, so parity is dropped.

> That's why the log shows a dozen front-gate DCA phases (Simple/Complex/Vref/DFE/Recenter) but only one or two back-gate QCA phases — the front gate is fine-tuning the messenger's ears, the back gate just opens the door and trims the voice.

## Eye width: the intersection of every soldier's window

<iframe src="/animations/eye-intersection.html" width="100%" height="620" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Eye intersection animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/eye-intersection.html">open full screen ↗</a></p>

CA training sweeps a string of delays, asking each soldier "can you hear the order at this delay?" The delays it can hear form a window. The order has to reach everyone at once, so all the windows are intersected. Three soldiers on a number line:

```
Soldier A: delay 10 ────────────── 30  hears (wide window)
Soldier B: delay       15 ────── 25    hears (hard of hearing, narrow)
Soldier C: delay 5 ────────── 20       hears
```
For all three to hear at once, the delay must land in the **overlap** (15~20):

- **Left wall MaxLeft = the largest of all left edges** = max(10, 15, 5) = 15. The intersection only starts once everyone has started hearing, so it waits for the latest starter (B enters at 15).
- **Right wall MinRight = the smallest of all right edges** = min(30, 25, 20) = 20. The intersection ends the moment anyone drops out, so the earliest dropout (C stops at 20) sets the wall.
- **Eye width = MinRight − MaxLeft = 20 − 15 = 5** (exactly B's window, the dullest ear).

> Take the **largest** left edge and the **smallest** right edge. One hard-of-hearing soldier narrows the whole army's intersection — the shortest stave of the barrel sets the eye width.

The eye center `(MinRight + MaxLeft) / 2` is written back to the RCD — that's the final delay.

## The pass line is graded by difficulty (normalized)

The eye width is checked against a margin table:

| Pattern | Low (warn) | Critical (disable) |
|---|---|---|
| SIMPLE | 20 | 12 |
| COMPLEX | 12 | **6** |

Counter-intuitive: the harder pattern gets the **lower** bar. Because the margin is the *deviation from this order's expected eye width*, not an absolute width. A SIMPLE order is easy, so soldiers hear it wide (~20+) and dropping to 12 is abnormal; a COMPLEX order is hard, so soldiers barely keep up (~8-12) and only dropping to 6 is abnormal. Hold COMPLEX to 12 and every normal COMPLEX eye (8) gets condemned — training never converges.

## The full flow (one skeleton for both gates)

```
Sweep delay (front: DCA delay / back: QCA delay), collect each hear-window (left/right edge)
  → intersect to get eye width = MinRight - MaxLeft
    → look up the margin table (TestType + Pattern + frequency override) for Low/Critical
      → eye width < Critical → DisableChannelSw (disable)
      → eye width < Low      → warn only (marginal, allow)
      → wide enough          → write eye center back to RCD (front: DCA delay / back: QCA delay)
```

> One line: **sweep delay → intersect eye width → look up table to judge life or death → write back the center.** Both gates walk this same skeleton; they differ only in which delay to sweep, which path the feedback rides, and which delay to write back — front sweeps DCA timing with a loopback to the host, back sweeps QCA timing with DRAM reporting over MPC.

## Three ways CA training fails

| Warning | Condition | Action | Nature |
|---|---|---|---|
| `Best Eye Width ... smaller than minimum critical margin` | eye < Critical | **DisableChannelSw** | SI fatal |
| `Best Eye Width ... small but not critical` | eye < Low | warn only | SI marginal |
| `cannot find the minimum CA margin ... not found in CaMinimumMargin table` | step missing from table | warn only | **table/config missing** |

1. **CA goes through the RCD, not through DFE** — DFE is the echo canceller on the read-DQ path, it can't save CA. Treating CA with DFE's medicine is the wrong prescription.
2. **The same stage can fail for totally different reasons** — "eye < Critical" is a signal-integrity problem, "table missing" is a config problem; one is fixed by tuning SI, the other by filling in the table.
3. **"Eye too small" must be read normalized** — COMPLEX fails only at 6, SIMPLE fails at 12; don't judge by the absolute number.

## Duty cycle

**Duty cycle is the ratio of a clock signal's high time to its low time** — ideally half and half.

```
Perfect (50/50):  beat—rest—  beat—rest—  ← beat and rest equal
Distorted (60/40): beat—rest—  beat—rest—   beat longer, rest shorter
```

- "Beat" = high (HIGH), "rest" = low (LOW); **duty cycle = beat / (beat+rest)**.
- Not the frequency: frequency asks "how fast is the beat", duty cycle asks "what's the beat/rest ratio".

**First, what "high/low" means (a differential signal has two layers)**:

The clock CK is a differential pair (CK_t + CK_c). The "high/low" in my diagrams is the **single-ended voltage of the CK_t line** (high=1.2V, low=0V), **not the logical 1/0 of the difference**:

| Layer | What it says | Who |
|---|---|---|
| Single-ended | CK_t's own voltage high/low | CK_t alone |
| Differential | sign of CK_t−CK_c (positive=1, negative=0) | both together |

CK_c is CK_t's **inverse**: when CK_t is high 60%/low 40%, **CK_c is low 60%/high 40%**, exactly reversed. So a duty-cycle distortion propagates to the differential layer — the difference's 1/0 also becomes 60/40.

**Why 50/50 matters (double-data-rate sampling + crossing-point shift)**:

DDR samples **twice per cycle** — at the differential clock's two **crossing points** (rising crossing = rising edge, falling crossing = falling edge). And the duty cycle decides where those two crossing points sit:

```
Cycle T fixed (frequency unchanged), two sample points (crossings):
  ↑rising crossing      ↑falling crossing      ↑rising crossing

50/50: |---- 50% ----|---- 50% ----|
       data bit 0 window  data bit 1 window   ← crossings even, half a cycle each

60/40: |------ 60% ------|-- 40% --|
       data bit 0 (wider)  data bit 1 (narrower)  ← falling crossing shifts from "midpoint" to "60%"
```

The data bits are laid out for an ideal 50/50 (each bit gets half a cycle around its ideal sample point). With a 60/40 duty cycle, the falling crossing shifts from T/2 to 0.6T, so:

- Data bit 0 (rising-edge sampled): window widens to 60% (even steadier).
- **Data bit 1 (falling-edge sampled): window squeezed to 40%** — the data has less time to finish toggling and settle, and may get sampled at a half-high, half-low blur.

And the eye width is set by the narrowest direction, so at 60/40 the whole CA eye is choked by that 40% direction.

> The clock is like breathing: inhale = high, exhale = low. 50/50 is even breathing; 60/40 is too short an exhale, forced to inhale before you finish — you choke, and sampling gets shaky.

**Why the RCD distorts it**: when the messenger re-drives the clock, the rising and falling edges have **different propagation delays**, so the re-driven clock comes out deformed, no longer 1:1.

**What DCA does**: tune the messenger by **sweeping a string of duty-cycle values and picking the one with the widest eye** — not force it back to 50/50. Because the RCD's rising/falling propagation delays differ (the signal is asymmetric), the best duty cycle is "50/50 + a correction for that asymmetry", usually **floating a few points around 50/50** (say 52/48, 48/52), never drifting to 60/40 or 70/30 — the further it drifts, the more one direction's eye is squeezed, and since the narrowest direction sets the eye, drifting further only makes it narrower. The center of the widest eye is the optimal duty cycle.

That's what `RCD DCA/DCK Duty Cycle Training` does. Twiddle the duty cycle by hand and watch the eye width change — the more it drifts, the narrower the eye. It's the most direct way to watch signal quality bite the eye width.

## The golden rule

> One order rings out, the whole army must hear it — the soldier with the dullest ear decides the whole army's fate.
