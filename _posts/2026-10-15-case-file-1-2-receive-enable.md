---
layout: mypost
title: "DRAM Detective · Case File 1.2: One Man Guards the Pass (Receive Enable)"
categories: [DDR5, Training]
lang: en
---

Reading memory is **a late reply**. The host sends a READ, and the DRAM doesn't answer right away — it dawdles, and the dawdling differs from die to die and from run to run. The host has no idea of the exact minute the reply will land. So it installs **a gate** (the DQS Gate) and trains when to open it. **Receive Enable is the training of when the gate opens.** Open too early and noise walks in; open too late and the data leaks away. One man guards the pass — **ten thousand cannot break through.**

<iframe src="/animations/receive-enable-gate.html" width="100%" height="660" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Receive Enable gate animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/receive-enable-gate.html">open full screen ↗</a></p>

## Why the read side needs a gate, and the write side doesn't

Writing is the host **shipping its own parcel** — it sets the rhythm, so it knows when the parcel leaves and when it lands. Reading is the DRAM's **reply** — the DRAM sets the rhythm, and the host just waits and guesses.

### What the read latency is made of

```
host sends READ
   ↓ ① CK rides the bus (fly-by, stops at every die)
DRAM gets the command (CK edge)
   ↓ ② DRAM works internally (access time, folded into CL): access, sense, drive DQS/DQ
DRAM spits out DQS + DQ
   ↓ ③ DQS/DQ take the taxi, point-to-point, back to the host
host catches the data
```

### Why not adjust each leg separately?

The read side does **not** tease apart CL / tDQSCK / fly-by / flight time. It wraps them in a black box and asks only one thing: *how late is the reply, in total?*

The black box isn't one exact number — every die, every run differs — so the host lets the DRAM drill on a fixed cadence and **sweeps** to find where the gate must open to catch the reply. Simple, crude, effective.

| Component | Nature | Can it be trained? |
|---|---|---|
| CL (read latency, in cycles) | **set by the host** into the DRAM's MR | No — you chose it |
| tDQSCK | DQS rising edge's **position** vs CK (a fixed offset, looked up) | No — looked up |
| fly-by skew | topology physics | read side **absorbs** it |
| flight time | DQS/DQ flying back to the host (trace length) | **measure only**, can't change |

> **Wait — the host writes CL itself, so why can't it adjust it?** Writing ≠ tuning. CL = CAS Latency, the number of clock cycles from READ to the first data bit, programmed in MR0. It *looks* free, but it's locked by the DRAM's physics: **CL = tAA / tCK** (rounded up). tAA is the DRAM's real internal access time in nanoseconds, set by silicon; tCK is the length of one clock cycle. The host just **converts physical nanoseconds into a clock-count and writes it down** — it can't turn three minutes into two.
>
> A chef takes three minutes per dish (tAA). At one beat per second, the host says "serve at beat 180"; if the beat halves (frequency doubles), it says "serve at beat 360". The host only converts and announces.

So CL is **chosen, not trained**. Training = measuring the unknown and canceling its bias; CL is a converted constant with no bias left to cancel. The real unknown is the **drift** — tDQSCKi, the DQS jitter window (0.41~0.60 tCK) — not tDQSCK itself (a fixed ~0.348 tCK position, `MrcDdr5.c` looks it up from a table).

## Two coordinates: sub-cycle first, then whole-cycle

| Coordinate | Asks | Who |
|---|---|---|
| **within a cycle** — which slot | where in one clock cycle the reply lands | **Fine** (first) |
| **which cycle** — which beat | which beat after the command the reply arrives | **Coarse** (second) |

> CL is only the **timetable** — in theory, beat CL. The real arrival carries offsets (tDQSCK position + fly-by + flight time) and jitter (tDQSCKi), different for every die and every run. The timetable says 9:00; the train is never on time. So Fine and Coarse measure *when this train actually arrives*: Coarse counts the beat, Fine finds the slot within it.
>
> And the answer is **not** written into the DRAM's MR. The gate lives in the host's PHY: Coarse writes `RxIoTclDelay` (the whole-cycle delay), Fine writes `RecEnDelay` (the per-strobe PI phase). The MRs are only the exam-room setup — MR2 enters training mode, MR8 sets the preamble length — not the result.

## Let the DRAM show its hand first — read preamble training mode

Normally the DRAM ships only when an order comes, and closes shop after. The host can't catch that rhythm. So in training the host tells the DRAM: **"enter rehearsal mode — I'll keep ordering, you keep shipping on a fixed beat."**

That's **JESD79-5 §4.18 Read Preamble Training Mode**: set `read_preamble_training` in MR2, and the DRAM sends its DQS preamble on a known fixed rhythm, so the host can probe when the first data and strobe come back.

> Normally the courier ships only when you order; in training mode he ships on a timetable, non-stop — so you learn his rhythm and catch the parcel.

## Fine first: find the lamp's position in the cycle

In rehearsal mode, DQS is like a lamp that **flashes once every cycle**. The host sweeps a shutter across one cycle to find the lit stretch: rising edge, falling edge, pulse width, center. That's the lamp's position **within** a cycle — found with the phase interpolator, a millimeter ruler that slices one cycle into fine ticks.

## Coarse second: count which trip the lamp is on

Fine only says **which tick within a cycle** the lamp lights — not **which cycle**. Coarse fills that in: the gatekeeper slides the gate back, trip by trip, from "lit" to "not yet lit". **The 1→0 flip is the preamble just peeking out — the round-trip delay found.**

> Fine finds the lamp's slot within a cycle; Coarse counts which trip the lamp is on. Know the platform first, then count the train.

## The preamble offset (the closing step)

The final gate position is shifted once more by the read preamble length (MR8 Read Preamble Settings: 1/2/3/4 tCK). The catch: **training mode forces a 1 tCK preamble** (spec §4.18.3), but real operation uses the MR8-programmed length — so the phase measured at 1 tCK must be converted back to the real length.

Final gate = round-trip delay (which beat) + in-cycle phase (which tick) + preamble offset.

## Write levels vs read gates (the money table)

| | Write Leveling | Receive Enable |
|---|---|---|
| Direction | host sends DQS, DRAM samples & reports | DRAM sends DQS, host opens the gate |
| Essence | DQS aligned to CK — **set the watch** | when the gate opens — **guard the door** |
| Spec mode | write leveling mode | read preamble training mode |
| Two-stage order | **Coarse first**, Fine second | **Fine first**, Coarse second |
| Cost | slower | **faster** — one edge + one cycle count, not a whole eye |

**The same fly-by skew: write compensates, read absorbs.**

| | How fly-by skew is handled |
|---|---|
| **Write** | **compensate actively** — the data (taxi) reaches all dies at once, but each die's CK (bus) arrives at a different time, so each die's DQS is **offset** to align with its own CK |
| **Read** | **absorb passively** — each die's DQS returns at a different time (its CK arrived late), so the host's gate opens **per die** |

> Writing is delivering meals to different floors — stagger the times so each floor eats at its own mealtime (compensate the bus skew). Reading is each floor coming down to collect — you set each floor's pickup window at its own arrival time (each gate guards its own).

## Why the order reverses

Write goes **Coarse then Fine**; read goes **Fine then Coarse**. The root cause: the read-side DQS **loops every cycle** (the lamp flashes every cycle — find the slot first, then count the trip; the two are orthogonal), while the write-side DQS is **one-shot** (you must align the beat before you can talk about the tick within it).

## Why DQS is always earlier than CK

The hidden premise of fly-by: the write side can "add delay to align" only because **DQS is born earlier than CK**.

- **CK takes the serpentine fly-by line** (loops around, stops at every die); **DQS takes the point-to-point short line** (direct, length-matched).
- Same speed, longer route arrives later — so for the far die, CK loops around (late) and DQS shoots straight (early). **DQS always wins**, and the gap grows the farther the die.

So the write side always just needs to **make the taxi wait for the bus** (add delay) — never the dead end of "the bus arrived but the taxi hasn't" (you can't subtract delay).

> The design motive of fly-by: **sacrifice CK's route (longer, per-stop skew) to buy DQ's length-matching (point-to-point, fast and even)** — which is what makes "add delay to align" possible.

## Why the host does all the training, and the DRAM stays dumb

Why not put a gate on the DRAM die too, and save the host the trouble? Three reasons:

1. **The dumber the DRAM, the better.** It's a JEDEC-standard mass-produced die — cheap, abundant. The industry rule: all training smarts go into the host; the DRAM is a well-behaved dumb slave. Add training logic to every die and every die costs more and breaks the standard.
2. **The knobs already live in the host.** The delay line (phase interpolator) used to offset DQS sits in the host's PHY. Putting another in every DRAM is duplicate construction.
3. **The DRAM is a slave, in no position to train.** Training = measure + adjust + feedback, a closed loop only the host can see in full.

> **The measurer and the adjuster must be on the same side.** The host sees the skew (measure) and holds the delay line (adjust), so training is the host's job.

## The golden rule

> Write sets the watch once; read guards the door once. Open too early and noise walks in; open too late and data leaks away — one man guards the pass.
