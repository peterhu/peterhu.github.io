---
layout: mypost
title: "DRAM Detective · Case File 2.3: The Two Doors"
categories: [DDR5, Training]
lang: en
---

The gate Receive Enable talks about is actually two doors. The sample gate opens right on time; the analog gate opens a few beats early. Early, because it's preheating the sense amplifier and the termination — turn on the lights and tune the sound before the actor steps on stage, then open the curtain the instant they do. Receive Enable only said how to use this gate; here we open it up and see how it's actually built.

<iframe src="/animations/the-two-doors.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The two doors — the analog gate preheats, the sample gate samples"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/the-two-doors.html">open full screen ↗</a></p>

## One — the gate is actually two doors

Receive Enable training finds the moment the sample gate should open. But down at the circuit, there's another gate — the analog gate — that opens first. The earlier chapter was about how to use it; this one is about why it's built the way it is.

| Gate | What it drives | When it opens |
| --- | --- | --- |
| analog gate | the sense amplifier and the termination (analog circuits) | early (preheat) |
| sample gate | the actual sampling of DQS | when the DQS preamble arrives |

One gate owns analog, the other owns sampling. The analog gate is an analog enable — it turns on the sense amplifier (the read amplifier) and the termination. The sample gate is a receive enable — it's the digital act of "now sample DQS."

## Two — the master is precise, the slave follows

These two signals aren't programmed separately. The sample gate is the master; the analog gate is just an early copy derived from it.

```text
sample-gate delay ──► delay chain + DLL ──► sample gate (to a tiny fraction of a cycle)
                                        └─ shift earlier by the early-open amount (0~3 beats) ─► analog gate (whole-beat granularity)
```

The master is precise to a tiny fraction of a cycle; the slave is its early copy, coarse to a whole beat.

Why early at all? The sense amp and the termination are analog circuits — after powering on they need a moment to settle. Open them exactly on time and they haven't settled when the sampling happens; the read comes out wrong. On DDR5 the early-open amount is 0~3 beats, tied to the preamble length and the frequency — not filled in at random.

Back to the theater: the actor (DQS) — before they step on stage, turn on the lights and tune the sound (the analog gate powers the sense amp and termination); the instant they step on, open the curtain (the sample gate). The lights go on early, so the stage is already lit when the actor arrives.

## Three — a few unavoidable whys

Why not merge them into one gate? The two needs don't fit: the sample gate must be dead-on (its moment is fine-tuned by the phase interpolator, and the eye lives or dies on it), while the analog gate must open early (to give settling time). Analog circuits have inertia; digital sampling must be precise to the beat. Merge them into one, and either the analog circuit hasn't settled when you sample — a wrong read — or the analog circuit stays on the whole time, burning power and adding noise.

Why must the analog gate open early at all? The sense amp and the termination need settling time after power-on; open them in sync and they're not stable when sampling happens. Opening early lets them settle before DQS arrives. Same reason as the DLL's focus earlier: analog circuits all need to settle, and you can't rush them.

Why is the early-open amount 0~3 beats? It's tied to the preamble length and the frequency. A longer preamble and a faster clock spread the needed settling time over more beats — so the value is derived from those, not guessed.

Why does the master get the phase interpolator while the slave only gets whole beats? The sample gate is a digital sampling act — it must land on the exact DQS preamble moment, and the eye lives or dies on that one point, so it needs sub-cycle fineness. The analog gate just preheats — any of the 0~3 beats works — so whole-beat granularity is plenty, no need to spend the phase interpolator on it.

Why is the analog gate derived from the sample gate instead of programmed on its own? So the two doors always stay lined up. The analog gate is always exactly the early-open amount before the sample gate — never sampling before the analog circuit settles, never opening too early and wasting power. The slave hangs on the master; move the master one tick, the slave moves with it.

So when you read that pile of receive-enable settings, first recognize that the sample gate is the master and the analog gate is just its shadow a few beats early — and the two enable signals stop being confusing.

**The gate is two doors — lights first, curtain second. The one that opens early is there to let the analog circuit warm up.**
