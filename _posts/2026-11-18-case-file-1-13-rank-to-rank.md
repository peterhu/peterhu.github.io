---
layout: mypost
title: "DRAM Detective · Case File 1.13: Nearest First, Farthest Last (Rank-to-Rank)"
categories: [DDR5, Training]
lang: en
---

Several ranks share one channel, strung along the same fly-by line at different distances — the signal reaches the near one first and the far one last.

Rank-to-rank training measures this gap, so switching between ranks stays aligned and collision-free.

<iframe src="/animations/rank-to-rank-street.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Ranks arrive at different times — measure the gap"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/rank-to-rank-street.html">open full screen ↗</a></p>

## One — CS picks the rank

A channel's ranks don't each pull their own bus — three signals work together:

| Signal | Role | What it does |
| --- | --- | --- |
| CA / CK | the shout | command and address broadcast down the fly-by line to every rank; everyone hears it, just at different times |
| CS_n | the name call | each rank has its own CS_n (chip select, active low); only the rank whose CS_n is pulled low responds |
| DQ / DQS | the shared mic | all ranks share one data bus; only the named rank uses it at a time |

Switching rank is simply: raise rank A's CS_n, lower rank B's. The command doesn't change a bit — only who is named changes. That's broadcast plus chip-select: CA is the shout, CS is the name call. Case 0.6 established CS_n as the doorbell; Case 1.3 established broadcast-plus-select to save pins.

### The shared mic — one mouth at a time

Each byte-lane (8 DQ plus a DQS pair) is one shared wire: rank 0's die and rank 1's die for the same byte-lane are wired onto the same net, to the same controller pins. It isn't separate wires — it's several mouths on one wire.

Keeping one mouth at a time takes two moves. On a read, the selected rank turns on its output and drives the bus; the others go high-impedance (tri-state), neither driving nor fighting. On a write, the selected rank latches the data; the others ignore it.

The crux is high-impedance: the unnamed rank isn't whispering, it's holding its mouth shut. So the wire is time-shared — one rank uses it for a stretch, then the next.

As an aside, the unnamed rank isn't fully dead: its DQ pins still carry a termination (ODT/Rtt) that soaks up reflections on the shared wire. That's Case 1.7, not here.

This brings two headaches.

First, arrival order. CA, CK, and CS all fly-by — they reach rank 0 near-and-first, rank N far-and-last. Switching to a different rank changes when the command takes effect. That gap is the delta (max minus min) computed later.

Second, the switching gap (turnaround). DQ/DQS is shared by several ranks; switch CS too fast and rank A's data hasn't finished when rank B grabs the bus — a collision. Two CS assertions must be separated by a minimum: JEDEC's tCCD_L as the baseline, plus the first-to-last rank's timing delta, plus a guardband.

### Several ranks strung on one line

A channel holds up to several ranks (DDR5 max 2DPC × 2R = 4). These ranks don't stand side by side — they're strung on one fly-by line: rank 0 near the controller, rank N far.

The same beat of command or data arrives near-first and far-last — that's flight time. CA/CK broadcast down the fly-by line, each rank hearing at a different moment; DQ/DQS is point-to-point, but the dies sit at different physical spots, so the read DQS arrives at a different time too.

The earlier trainings (2D Centering, Receive Enable, Write Leveling, per-bit de-skew) align each rank's timing individually. But each rank's flight time differs, so the timing they need is slightly different. Rank-to-rank fills this gap.

## Two — four knobs move when you switch rank

Reading rank A then rank B isn't turning one knob. Four timing domains on the read path must move:

| Knob | What it does | Why switching moves it |
| --- | --- | --- |
| DQS offset | when the DQS strobe arrives | DQS flight time differs per rank |
| DQ offset | when the DQ data arrives | same |
| sampling phase | where the sampler sits | the sample point follows arrival time |
| gate timing | when the DQS gate opens | the gate must cover each rank's DQS |

Why exactly four? Switching rank changes flight time, and flight time feeds all four at once — DQS arrival, DQ arrival, sample phase, gate timing. All four move together, so all four get their own switch offset.

### Don't misread it: earlier steps already train per-rank

The easiest trap here: rank-to-rank sits so late — did the earlier steps only train rank 0, and the other ranks get their first touch here? No.

The earlier steps (Receive Enable, Write Leveling, 2D Centering, per-bit de-skew) already train per-rank; each rank has its own timing. Those are the per-rank values trained earlier — this step only stores them to do the arithmetic.

So this step trains not each rank's static timing, but the act of switching between ranks. Two orthogonal things:

| | per-rank timing (static) | rank-to-rank switch (dynamic) |
| --- | --- | --- |
| what | each rank's DQS delay, gate timing, phase delta | the four rank-to-rank switch offsets |
| trained by | the earlier steps, per-rank | this step |
| solves | each rank's eye centered on its own | back-to-back cross-rank access (minimum turnaround) passes |
| dimension | per-rank, one set per rank | not per-rank, one value per byte-lane |

There's also a normalization: to switch fastest, the timing is compressed to the tightest rank's value, so nobody has to wait for the slow one.

So it sits last not because earlier steps only trained rank 0, but because it must wait for every per-rank timing to be pinned down before it can measure the switch offset. Static state first, then the switching state can be measured.

### Only the first and last rank pair

Ranks aren't trained pairwise — only first → last.

- 2DPC all 2R: train DIMM0 R0 → R1
- 2DPC one 1R, one 2R: train the 2R DIMM's R2 → R3
- 2DPC all 1R: train DIMM0 R0 → DIMM1 R2

Fly-by arrival order is monotonic: rank 0 nearest, rank N farthest. The first and last ranks have the biggest arrival-time gap — the worst case. The middle ranks fall inside that interval; training this one pair covers all of them and saves time.

## Three — the switching gap: baseline plus max delta plus guardband

Measuring each rank's offset isn't enough. Back-to-back access to two different ranks — how short a gap is allowed? Too short, and the first hasn't finished when the second collides.

The JEDEC baseline is tCCD_L / tCCD_S (CAS-to-CAS minimum spacing; tCCD_L is programmed in MR13, JESD79-5 §3.5.15 + Table 53; different bank groups, including different ranks, use tCCD_L). On top of this spec baseline, add the first-to-last rank's timing delta and a guardband.

It takes the maximum delta (max − min), not the average. The worst case is switching from the rank with the largest timing to the one with the smallest; the physical gap must cover that whole span, plus margin. Same idea as the Vref case: keep every margin at the worst case.

This step sits very late: after Post DFE 2D Centering Late, Sense Amp Optimization, and Roundtrip Latency Optimization, before CTE Cpgc And Cadb Test — finishing the whole DDR Training at 90466ms. Why? The delta depends on every earlier training's result — each rank's DQS/DQ/phase/gate timing is trained first, 2D Centering fixes the center, then the delta can be measured precisely. It's the final calibration, not the foundation.

## Four — the postman's street

A street (one channel's fly-by bus), with four houses along it (four ranks), house 1 near the entrance and house 4 farthest. The postman (controller) comes in from the entrance and delivers house to house — house 1 first, house 4 last. Same action, different door distance, different delivery time.

At each house, he knocks first (CS chip-select, the name call), and only that house opens the door. The other houses see the postman go by (broadcast) but weren't knocked, so they don't open. Switching rank is switching the knock from house 1 to house 4.

All the earlier training is calibrating the delivery time to house 1. Rank-to-rank measures how much later houses 2, 3, and 4 are than house 1, and records an offset for each. Deliver to house 1 today, house 4 tomorrow — no re-scouting, just add the offset.

Only measure house 1 and 4? Because the street is monotonic: house 1 nearest, house 4 farthest, and the 1→4 time gap is the biggest. Houses 2 and 3 sit in between.

Turnaround is: after house 1, how soon can you start house 4? You must allow the walk from the street entrance to the far end, plus a little margin. The parcel can't land before the postman reaches the door.

Look at the shared wire again through walkie-talkies: several walkie-talkies share one channel (several ranks share one DQ/DQS wire); hold the push-to-talk to speak (CS_n selected, output on); release to shut up (tri-state high-impedance); two press at once and they jam (bus contention). The pause between "over" and the next speaker's first word is the rank-to-rank gap.

## Five — how to diagnose

- **Single-rank setups have no such step** — it's skipped when there's only one rank.
- **Symptom**: single-rank access works, but alternating between ranks corrupts data. That's the classic rank-switch misalignment, a different disease from a single-rank margin problem.
- **Bisect**: offset measured wrong (the switch-offset training failed) → after switching to a rank, that rank's eye is off; gap too small (turnaround too tight) → the two accesses collide.

**Golden rule: nearest rank hears first, farthest hears last. Measure the gap, and switching stays on time — without collision.**
