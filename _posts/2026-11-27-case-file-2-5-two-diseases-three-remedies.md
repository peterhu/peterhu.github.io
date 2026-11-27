---
layout: mypost
title: "DRAM Detective · Case File 2.5: Two Diseases, Three Remedies"
categories: [DDR5, Training]
lang: en
---

A signal running down a trace catches two diseases: reflection and inter-symbol interference (ISI). The three circuits that treat them all live on the host's receive side (RX): ODT swallows the reflection, CTLE and DFE team up to cancel the ISI. Reads go DRAM → host, and whether the eye stays open comes down to whether these three can rescue the signal that arrives.

<iframe src="/animations/the-three-remedies.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The three remedies — ODT, CTLE, DFE"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/the-three-remedies.html">open full screen ↗</a></p>

## One — the two diseases

| Disease | Symptom | Treatment |
| --- | --- | --- |
| reflection | the signal bounces off the far end and lands on top of later signals | ODT |
| inter-symbol interference (ISI) | the highs are eaten, the previous bit's tail drags onto the current bit | CTLE + DFE |

One thing that trips people up: high-frequency loss and ISI are not two things — they're the same thing from two angles. Look at frequency, and the trace is a low-pass filter that eats the highs first. Look at the waveform, and with the highs gone the edges round off, the pulse drags a tail, and that tail smears onto the next bit. So there are only two diseases, not three.

This case is about the receive side only. The transmit direction (host writing data) uses a different tool — it lifts the highs at the source, called FFE pre-emphasis, deferred to Case 2.6. On the receive side there are three:

| Circuit | Job |
| --- | --- |
| ODT (RX ODT) | terminates the trace on a read, swallowing the reflection |
| CTLE | boosts the highs on a read, blind |
| DFE | subtracts the previous bit's tail on a read, precise |

## Two — ODT swallows the echo

### Why a well-behaved signal bounces back

A trace has its own temperament, called characteristic impedance (commonly around 40 Ω on a board). A signal runs to the far end, and if it meets a hard wall there — the impedance differs a lot from the trace, or it's simply open — the energy has nowhere to go, so it bounces back the way it came and stacks on top of later signals, leaving ringing and stair-steps on the waveform.

ODT (on-die termination) puts a resistor at the receive end, about as big as the trace's impedance, so the energy arriving at the far end has somewhere to be absorbed. Like shouting into an empty hallway and hearing the echo off the wall — line the wall with sound-absorbing foam and the sound hits it and dies instead of bouncing back. The closer the resistor matches the trace, the smaller the echo.

### The resistor isn't a single fixed one

The ODT is a bank of resistor legs in parallel; how many legs are switched on sets the equivalent resistance — the same idea as how many bulbs you switch on sets the brightness. There's also rotation, so the legs take turns and no single leg ages faster (they rotate when the termination turns off). How big each leg is comes from the ZQ/COMP calibration — the same parallel-leg scheme as Case 1.10.

### Not always on — it follows the sample gate

It opens a little after and closes a little after the sample gate, wrapping the actual sampling window. This is the exact opposite of Case 2.3's analog gate: the analog gate opens early to preheat the sense amp and termination, while the termination itself opens late — only when there's a reflection to swallow, to save power. A resistor always hanging on the line both burns power and disturbs the next signal's swing.

### Don't mix up the two sides' ODT

ODT isn't unique to the host — the DRAM side has it too. Lay out the two directions:

| Direction | Transmitter (drives) | Receiver (terminates) |
| --- | --- | --- |
| read (DRAM → host) | DRAM uses Ron | host uses RX ODT |
| write (host → DRAM) | host uses DS | DRAM uses Rtt |

The host side is called RX ODT, used on reads, a bank of resistor legs coordinated with the sample gate. The DRAM side is called Rtt, used on writes, defined by JEDEC. The DRAM side comes in three:

| ODT | JEDEC name | Scenario | Who terminates |
| --- | --- | --- | --- |
| write | Rtt_WR | write: the target rank receives write data | the target rank |
| nominal | Rtt_Nom | read or write: non-target ranks hang on and absorb reflection | non-target ranks |
| park | Rtt_Park | idle / power-down: keep the bus at a known level | all ranks |

In plain words: during a write the target rank terminates with Rtt_WR; during a read or write the ranks not being addressed help absorb reflection with Rtt_Nom; and when idle everyone hangs Rtt_Park to keep the bus quiet.

## Three — CTLE turns the eaten highs back up

### The disease and the cure

A signal over a long trace loses its highs fast, and the waveform rounds off, the edges dull. Like listening to music through a thick blanket — the kick drum is still there, the cymbals' highs are gone. CTLE (continuous-time linear equalizer) is what turns the treble knob up: it amplifies the highs, re-sharpens the edges, shortens the tail. This is the frequency-domain half of fighting ISI.

It has a built-in shortcoming — it's **blind**. CTLE doesn't know whether the bit just sent was a 0 or a 1, so it can only lift the whole high-frequency band, noise included. That's why DFE is still needed for the time-domain touch-up.

### There's an RC tucked under the amplifier

CTLE isn't a simple RC filter — it's a differential amplifier with an RC network in parallel at the source-degeneration point. The differential amplifier is like a little speaker that only amplifies the difference between two wires; source degeneration is like a block placed under that speaker to keep it from blasting too hard. The clever part is that this block changes with frequency:

- **Low frequency**: the capacitor is an open circuit, current only flows through the resistor R, the block is still there, gain is held low.
- **High frequency**: the capacitor is a short, bypassing R, the block effectively disappears, and the gain jumps up.

So the curve is born low at low frequencies and high at high frequencies — the exact inverse of the trace's low-pass. Two knobs do the job: bigger R means stronger degeneration, the low-frequency baseline sits lower, and the highs poke out more; bigger C bypasses R more completely at high frequency, and the highs lift higher.

### It's receive-side only

- The host's CTLE is for reads: data arrives from the DRAM, and the highs are boosted on arrival.
- Sending commands and writing data is the TX direction, which uses FFE pre-emphasis (de-emphasis, Case 1.11's TxEQ), lifting the highs at the source — a different tool from CTLE.
- The DRAM data side has no CTLE and no DFE — only ODT (Rtt) and Ron. Equalizers are a luxury only a big controller chip can afford.
- One exception: the RCD (the command repeater on an RDIMM) has CTLE, but it receives commands (CA), still on the receive side.

## Four — DFE subtracts the previous echo

At high speed, the previous bit's tail hasn't fully dissipated before it lands on the current bit, and the decision easily goes wrong. Like a video call where the previous sentence's echo hasn't faded and bleeds into this one. DFE (decision feedback equalizer) does this: first hear the previous sentence clearly (decide the 0/1), use that to work out how much tail it will bring, and subtract it from the current sentence. This is the time-domain half of fighting ISI, pairing with CTLE's frequency-domain half.

<iframe src="/animations/dfe-decision-feedback.html" width="100%" height="560" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="DFE subtracts the previous echo"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;"><a href="/animations/dfe-decision-feedback.html">open full screen ↗</a></p>

Each bit gets one subtraction and one decision; the decision goes into a shift register (remembering the last 1–6 bits), and the next bit uses it to cancel the tail — nothing is subtracted twice. Three bits make it clear:

| Time | Received | Subtractor removes | Decision |
| --- | --- | --- | --- |
| t1 | bit1 raw signal | 0 (no previous bit yet) | bit1 = 1 |
| t2 | bit2 raw signal | bit1's tail (1 × tap1) | bit2 = 0 |
| t3 | bit3 raw signal | bit2's tail (0 × tap1) + bit1's tail (1 × tap2) | bit3 = 1 |

Six taps in total (post-cursor), with shrinking widths:

| tap | width | cancels whose tail |
| --- | --- | --- |
| 1 | 6 bits | the previous bit (closest, strongest tail) |
| 2 | 6 bits | two bits ago |
| 3 | 5 bits | three bits ago |
| 4–6 | 4 bits | four to six bits ago (farthest, weakest tail) |

The 6/6/5/4/4/4 order has a reason: the farther the tail, the weaker it is, the less precision it needs, so the width can shrink and save circuit.

Why not use the raw signal to work out the tail? Because the decision is clean — just a 0 or a 1, no noise — while using the raw waveform would subtract the noise in as well, making it dirtier the more you subtract. **It cancels ISI without amplifying noise** — that's exactly where DFE beats CTLE, and why both remedies are kept.

The coefficients are also two layers: a base per rank, plus a small per-DQ offset, same pattern as Vref and CTLE.

DFE likewise lives only on the host's receive side, not the DRAM data side — it has to use the previous bit's decision to compute the tail, which needs decision logic and a fast feedback loop that only a big controller chip can fit.

## Five — a few small questions

Why treat reflection and ISI separately? Different roots. Reflection is an impedance mismatch — the signal bounces where the impedance jumps, a transmission-line matter. ISI is insufficient bandwidth — the highs are eaten and the pulse drags a tail, a channel matter. One eats the energy with a termination resistor, the other repairs the waveform with an equalizer; neither can do the other's job.

Why does ODT follow the sample gate instead of staying on? As above, a resistor always hanging on the line both burns power and disturbs the swing, so it only opens inside the sampling window. This connects naturally to Case 2.3's two gates: the sample gate opens, ODT follows a moment later; the sample gate closes, ODT holds a moment more.

Why does fighting ISI need both CTLE and DFE? CTLE is blind — an analog high-pass that doesn't know the previous bit, so it lifts the whole high band, noise and all. DFE is precise — it knows what the previous bit was decided to be, and subtracts only that tail, leaving the noise alone. One patches the frequency domain coarsely, the other subtracts the time domain precisely; they work in tandem, not either-or.

**Two diseases, three remedies — ODT swallows the reflection, CTLE turns up the highs, DFE cancels the echo.**
