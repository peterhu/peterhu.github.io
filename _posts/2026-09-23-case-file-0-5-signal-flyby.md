---
layout: mypost
title: "DRAM Detective · Case File 0.5: The Clock Rides the Bus, Data Takes the Taxi"
categories: [DDR5, Signal]
lang: en
---

Two signals born of the same mother walk different roads in life. The clock/command/address kid can only take the bus (fly-by, stop after stop). The data/strobe kid is the favorite — it takes a taxi (point-to-point, straight through). On the bus, time is never reliable, and every stop is late by a different amount. So the first job of training is to set the watch, stop by stop, for a bus that's never on time.

## Three signals: CK, DQS, DQ

Case 0.4 introduced the command (CA) and data (DQ) lines. Here we add the signals that bark the orders at them:

| Signal | Who | Job | Picture |
|---|---|---|---|
| CK (clock) | the conductor | the global reference | the wall clock, ticking away |
| DQS (strobe) | the whistle / flag | announces "cargo's here" | the courier's phone call — rings only when the cargo arrives |
| DQ (data) | the cargo | the real payload | the parcel itself |

CK is the wall clock that never stops ticking; DQS is the phone call that arrives with the goods.

CK: continuous, global, on its own wire — it answers "which beat are we on?"
DQS: momentary, local, bound to the data — it only rings when the cargo shows up.
DQS is a differential pair (`DQS_t - DQS_c`). The 64 data bits split into 8 byte-lanes, one DQS per lane. A DDR5 channel splits into two 32-bit sub-channels, with two CK pairs per sub-channel — so one channel has 4 CK pairs but 8 DQS pairs.
CK and DQS are both differential signals — noise-immune, reporting time at the crossing point. See the side story on why differential.

## The bus vs the taxi

Case 0.4 told us the RCD is the loudspeaker that fans the boss's orders out to every die. But from the loudspeaker to each die, the road forks into two completely different paths:

| Signal | Route | Topology | Picture |
|---|---|---|---|
| CA + CK | through the RCD, then along a chain, die by die | fly-by (daisy chain) | the bus, stop after stop |
| DQ + DQS | straight to the dies | point-to-point | the taxi, door to door |

<iframe src="/animations/bus-vs-taxi.html" width="100%" height="620" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Bus vs taxi animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/bus-vs-taxi.html">open full screen ↗</a></p>

The fly-by chain runs: controller → RCD → die #1 → die #2 → … → the last die. The clock rolls down the line like a wave, from the first die to the last.

So fly-by means every die hears CK at a different moment — the first die hears it first, the last die hears it last. The gap between them is the flight-time skew.

JESD79-5 §4.21.1: "The DDR5 memory module adopted fly-by topology for the commands, addresses, control signals, and clocks. … it also causes flight time skew between clock and strobe at every DRAM on the DIMM."

Why fly-by instead of a star branch? "benefits from reducing number of stubs and their length" — fly-by trades shorter stubs for that skew. Every gain costs something, and this cost is the hole training has to fill. Why a stub reflects and blurs the signal — that's a side story for the crosstalk-and-reflection case.

## Why DQ follows DQS, not CK

Because CK rides the bus. Its relative delay to DQ drifts with frequency and distance — the faster you go, the bigger the skew. The 64 bits each travel their own wire and arrive at different times; one edge of CK might land dead-center on bit3 and fall outside bit7's window. One beat can't shepherd 64 bits with 64 different delays.

DQS and DQ, though — those two brothers roll up in the same taxi, together. That's source-synchronous, and their relative timing is naturally stable.

Remote sync is never as good as walking together. CK walks its own road and doesn't know whether the cargo has arrived; the courier's phone call arrives with the cargo — that call is when you go pick it up.

## Filling fly-by's hole

Fly-by dug a hole that wiring can't fix. Only training can pay back each die's time difference — three directions of debt to settle:

| Direction | What to fix | How | One line |
|---|---|---|---|
| Write | DQS aligned to CK (fly-by gives each die its own CK arrival) | Write Leveling | actively align DQS, die by die, to its own write-latency moment |
| Read | which window the DQS gate opens | Read DQS Gate | passively open the window — work out the round-trip delay and open the gate where the data arrives |
| Command | CK/CA's own duty cycle and command alignment | CA training / duty cycle | fix the clock's own waveform and command alignment |

Write is active, read is passive:

When you write, the initiative is yours — you can actively decide when to send, and to which die, using leveling to set the watch. Writing is: I work out the time, and send on my own initiative.
When you read, the initiative is the DRAM's — it sends the data back and you don't know how long it flies, so all you can do is open a window and wait, using gate. Reading is: I work out the time, and open the window to wait.

## The golden rule

> The clock rides the bus, data takes the taxi. The bus never stops on time — so you set the watch, stop by stop.

The clock is the beat, DQS is the doorbell, DQ is the cargo. The clock travels the long daisy chain and runs late at every stop; the strobe rides with the data and arrives on time. That single difference — a shared road versus a private car — is the whole reason memory needs training.
