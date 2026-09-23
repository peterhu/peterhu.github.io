---
layout: mypost
title: "DRAM Detective · Case File 1.0: The Martial Art of Memory Training"
categories: [DDR5, Training]
lang: en
---

Memory training isn't a single sprint — it's a martial art, mastered stage by stage. Each stage tunes one kind of signal, and the order can't be shuffled: first right yourself, then right the other; first the mantra, then the moves; first observe, then strike; first coarse, then fine; and finish with grand mastery. Cultivate out of order and your qi deviates — all your work goes up in smoke.

## The training map

| Stage | Inner art | What it tunes |
|---|---|---|
| Pre-training prep | Right yourself | Calibrate your own analog circuits (Xover / SenseAmp / CTLE), exit DDRIO reset. |
| Command / Address | The mantra | CA/DCS/CS/QCS/QCA, through the RCD |
| Read direction | Open the third eye | Receive Enable → 2D Centering → DFE |
| Write direction | Strike | Write Leveling → 2D Centering → DFE |
| Per-bit de-skew | Unblock the meridians | Remove each bit's own skew |
| Finishing optimizations | Grand mastery | SenseAmp / RxJitter / RoundTrip / Turnaround / PPR |

Vref isn't a stage of its own — it's the Y axis of 2D Centering, and the Vref dimension of command training. More when we get to 2D.

<iframe src="/animations/kungfu-master.html" width="100%" height="620" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Kung fu master animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/kungfu-master.html">open full screen ↗</a></p>

## Why this order?

**① Right yourself before you right the other.**

Before training the DRAM, calibrate your own analog circuits. If your own ruler isn't calibrated, using it to calibrate someone else's ruler (the DRAM) is nonsense.

**② The mantra before the moves.**

Command/address is the mantra; data is the moves. Recite the mantra wrong and every move goes sideways. And the command lines have no doorbell riding along — they must be trained alone, first.

**③ Observe before you strike.**

Write training means "write it in, then read it back to check the answer" — so the read path must first become a sharp pair of eyes. Only a sharp eye can tell the true bead from a fake one.

**④ Coarse before fine.**

Every pass starts by sweeping for a rough position, then brings up the equalizer, then fine-tunes. DFE changes the eye's shape, so after fitting DFE the eye's center moves again and you must re-center — that's why Pre DFE 2D Centering and Post DFE 2D Centering always come in pairs.

**⑤ Grand mastery comes last.**

The finishing touches are the icing — there's only margin to squeeze once the earlier power is in place. Not just SenseAmp: also RxJitter cancellation, RoundTrip latency, Turnaround timing, and PPR repair — all squeezing more out of a system that already works.

## The real training trace

```
PreTrainingInit → HmrcScadExit           ← ① prep (exit reset)
RCD DCS → CS → RCD DCS Vref/DFE          ← ② mantra (command foundation)
RCD DCA TCO → Duty Cycle → DFE Tap1/DFE  ← ② mantra, advanced
Post Frontside → BCOM → QCS → QCA        ← ② command bus
MDQS RecEn → RecEn → Read DQ-DQS Coarse  ← ③ the eye (read direction)
DB Duty Cycle → Write Leveling → Write DQ-DQS  ← ④ strike (write, after read)
Rx Jitter → Sense Amp Optimization       ← ⑤ grand mastery
```

## The golden rule

> Training is a row of dominoes — each one has to fall right before the next can stand.
