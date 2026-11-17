---
layout: mypost
title: "DRAM Detective · Case File 1.12: Cut It Down the Middle (Vref)"
categories: [DDR5, Training]
lang: en
---

The decision line must cut right down the middle of the eye.

Vref is the voltage line the receiver uses to decide 0 or 1. Lean it one way and that side's data hangs by a thread; put it in the middle and both 1 and 0 keep their full margin.

<iframe src="/animations/vref-cut-middle.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Cut the decision line right down the middle"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/vref-cut-middle.html">open full screen ↗</a></p>

## One — one line decides 0 or 1

When the receiver gets a bit, how does it know it's a 0 or a 1? A voltage threshold. Above it is a 1, below it is a 0 — that line is Vref (reference voltage).

Look at the eye vertically: Vref is the boundary between the 1-eye and the 0-eye. Cut it in the middle and both halves get equal margin. Lean it up and the 1-side narrows first — a 1 gets misread as a 0. Lean it down and the 0-side suffers. In the end, Vref is just where you put that boundary.

## Two — one ruler isn't enough

First, centering. The eye is often asymmetric — the 1-eye and the 0-eye aren't the same size. So the line goes where the top and bottom margins are equal, not the geometric center. Otherwise the narrow side gives out first.

Then, per-bit. Each DQ line has its own comparator, and manufacturing variation makes these comparators' thresholds a bit crooked — some a few millivolts high, some a few low. A single Vref is just an average ruler that covers most bits; the most-offset bits still lose margin. So each DQ gets its own correction to cancel its comparator's tilt.

The method is two steps: first sweep the global Vref to set the baseline; then, near that baseline, sweep a small per-bit offset and keep the widest eye. So per-bit isn't a fully independent Vref per line — it's one baseline plus a small offset per bit.

Like a stack of kitchen scales: each scale's zero drifts, and empty they read +2g or −1g. The global Vref calibrates them all to the average zero; per-bit is pressing each scale's tare button to zero out its own drift, so the same item reads the same on every scale.

The spec has a home for all of this: MR10 is the global VrefDQ baseline, and MR118/126/134 and so on are the VrefDQ Offset registers — one per DQ, holding that small offset. In the log, RXVREFPERBIT toggles the read-side per-bit Vref, and it's usually off — an advanced option.

### And three thresholds

DQ, CA, CS are three different single-ended paths — different traces, rates, loads — so their optimal thresholds differ. One Vref over all three squeezes some path.

| Signal | Carries | How it runs | Why the threshold differs |
| --- | --- | --- | --- |
| DQ | data | high-speed, bidirectional, point-to-point | fastest and most noise-sensitive, needs fine tuning |
| CA | command/address | low-speed, unidirectional, fly-by broadcast | must let the worst die still hear it, cares about the whole |
| CS | chip select | one wire, gates a whole rank | wrong means the whole rank dies, kept separate |

The trace length, load capacitance, noise, and rate all differ, so the three eyes don't align and their centers don't land on one point. Each cuts its own. Like three rivers — a rapid, a main channel, a tributary — each with its own flood line; one line for all three means trouble.

The spec puts the three Vrefs in three registers: MR10 is VrefDQ (data), MR11 is VrefCA (command/address), MR12 is VrefCS (chip select). MR9 is VREF Configuration, which picks which set and on/off — a master switch, not a fourth threshold.

While we're here: only single-ended signals need this line. CK/DQS are differential, so they don't.

Differential is two wires — true (CK_t / DQS_t) and complement (CK_c / DQS_c) — always opposite, mirror images:

```text
voltage
 ↑
 │  ╱╲    ╱╲    ╱╲          ← true CK_t (swings up)
 │ ╱  ╲  ╱  ╲  ╱  ╲
 │╱    ╲╱    ╲╱    ╲
 ├──────────────────────     ← VIX (crossing: the two lines are equal)
 │╲    ╱╲    ╱╲    ╱
 │ ╲  ╱  ╲  ╱  ╲  ╱
 │  ╲╱    ╲╱    ╲╱          ← complement CK_c (mirror image)
 │
 └──────────────────────→ time
```

The moment the two lines cross (t equals c) is the flip. Since the two are symmetric, the crossing naturally lands in the middle of the swing — the complement is the true's own reference, so no one needs to draw a ruler. DQ, by contrast, has only one wire and no reference to lean on, so we have to draw a Vref for it.

One more easily-missed layer: CK is a clock, DQS is a strobe — whether they're 0 or 1 doesn't matter; what matters is the edge, when the transition happens. The crossing gives that edge, and the receiver uses it to time the sampling. The one that must be judged 0 or 1 is DQ, which is single-ended, hence VrefDQ.

So one manages the moment and one manages the value: CK/DQS say when to sample, DQ plus VrefDQ say whether you got a 0 or a 1. The differential side needs no Vref because it only manages the moment, never the value.

## Three — how it's swept

Same recipe as the other margin optimizations: walk through the Vref settings, measure the eye each time, keep the one with the most even top/bottom margin.

- `Read Voltage Centering`: read data's VrefDQ
- `Write Voltage Centering`: write data's VrefDQ
- `Command Voltage Centering`: command's VrefCA
- `Read/Write Voltage Centering 2D`: timing × voltage together

The first two are one read and one write; the third handles the command; the last turns both the time ruler and the voltage ruler at once.

## Four — the eye is still small

Sweep every Vref setting and the eye is still too small — that's a failed margin check.

- All lanes bad: the eye is nearly closed (loss, crosstalk, power), and no line can save it.
- One lane bad: that line's receiver offset.
- One rank bad: the die, socket, or gold fingers.

**Golden rule: cut the line right down the middle, and both sides keep their margin. Lean one way, and that side gives out first.**
