---
layout: mypost
title: "DRAM Detective · Case File 1.11: Not Too Fast, Not Too Slow (Slew + Tx EQ)"
categories: [DDR5, Training]
lang: en
---

The transmit edge must be neither too fast nor too slow; and on the write side, shout first.

Slew controls the edge slope — too steep means overshoot, ringing, EMI; too shallow means the eye closes. Tx EQ is pre-emphasis — shout the high frequencies out first, so the trace eats some and what lands at the DRAM is flat.

<iframe src="/animations/slew-txeq-shout.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Slew tunes the edge, Tx EQ shouts first"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/slew-txeq-shout.html">open full screen ↗</a></p>

## One — the two transmit-side knobs

How fast the signal flips from low to high is the slew rate. Too slow and it drags, the eye closes; too fast and it overshoots, rings, and bothers the neighboring lines. So there is no "faster is better" — only the setting that is just right.

The write side has one more knob: Tx EQ. The trace is a low-pass filter — the higher the frequency, the harder it eats, and the sharpness of the edge is the first thing that goes dull. So the host shouts a bit harder on the way out, and after the trace eats its share, what reaches the DRAM is flat. It is betting ahead of time on how much the line will steal.

Two knobs, two different things. Slew controls how hard you push — dV/dt = I/C — it changes the current charging the load. Tx EQ controls how much extra swing you add at the transition, to make up for the channel loss. One manages the slope, one manages the amplitude — don't mix them up.

### How these knobs map to 1.8 / 1.10

These knobs split into two camps by who sends and who receives:

| What | Transmit | Receive |
| --- | --- | --- |
| Impedance | Comp (Ron / Drive Strength) | ODT / Rtt |
| Edge / high frequency | Slew + Tx EQ | CTLE / DFE |

On read, the DRAM spits out data, it travels the whole line to the host, and by then the treble is mostly eaten. The host is the receiver, so it compensates at the ear: CTLE turns the treble back up, DFE clips the ISI tail.

On write it is reversed — the host is the sender, so it compensates at the mouth first, with Tx EQ pre-emphasis. The DRAM side has no CTLE: that is an analog circuit that takes area and power, and the die is a small cost-sensitive chip that cannot afford it, so it only keeps a cheap DFE. DFE only cancels ISI, it cannot make up for channel loss. Reworking the board or raising the voltage is hardware money; the cheapest move is to have the host shout louder at the transmit end.

So no matter how you slice it, the treble-boosting job always lands on the host: on read it compensates on arrival, on write it compensates before leaving. The DRAM only contributes that cheap DFE on both ends. Case 1.8 told the same story from the read direction; this one tells it from the write direction.

## Two — how they are trained

First, an easy thing to get wrong: Slew cannot be hardcoded at the factory. Trace length, layer count, which batch of dies, how hot it is today, what the voltage is — all of it moves, so the optimal slope is not a constant. The spec backs this up: JESD79-5 §8.5.3 (input), §9.6 / §9.8 (output) only define how to measure slew rate and its limits, and the bins above 5200 are literally marked TBD. The spec stops there — how the settings are divided and swept is the PHY's own job.

The sweep follows the same recipe as ODT, CTLE, DFE: sweep the setting → measure the eye → pick one → re-center.

But "pick one" does not mean pick the widest eye. Margin has to be enough, the power budget cannot break, and you still need to leave aging guardband; hit a parameter boundary and this sweep is wasted. So it is a trade-off, not a maximum.

And one more easy-to-miss step: after turning the transmit knob, the receive side and the timing both need re-centering. Otherwise they drift and the earlier work is undone. In the log, every one of these tasks ends with a re-center of write Vref and write timing — that is exactly this.

The three tasks show up back to back in a real log — don't mix them up:

- `CMD CTL CLK Slew Rate`: the edge on the command/control/clock path.
- `CMD/CTL Tx Equalization`: the transmit equalization on the same path.
- `Write Equalization`: the transmit equalization on the data path.

The first two handle the command side, the last one handles data — all host-to-DRAM.

## Three — before you blame the board

Sweep every setting and the eye is still small — that is a margin check failure. Split it three ways first:

- All lanes bad → trace loss, crosstalk, or power — the knob cannot save it.
- One lane bad → that one line's loss or crosstalk.
- One rank bad → the die, socket, or gold fingers.

Also, glance at the log for these three tasks' status and how many sweeps they ran — is it "no good setting found" or "no eye at all" — before you start suspecting the board.

**Golden rule: not too fast, not too slow — and on the write side, remember to shout first. That's the whole trick.**
