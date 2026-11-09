---
layout: mypost
title: "DRAM Detective · Case File 1.7: Absorb the Lingering Echo (ODT / RTT)"
categories: [DDR5, Training]
lang: en
---

A note travels down a room and hits a hard wall — it bounces back, and the echo smears the next note.

Memory signals do exactly that. A signal races down a trace, hits an impedance mismatch (a connector, a via, a solder joint), and bounces back. That reflection piles onto the next bit and shrinks its eye. **ODT (On-Die Termination) is the acoustic foam on the wall** — tune its thickness (Rtt) to match the room's character (Z0), and the echo dies before it muddies the next note.

<iframe src="/animations/odt-echo-absorber.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="ODT absorbs the reflected echo"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/odt-echo-absorber.html">open full screen ↗</a></p>

## One — the room echo is reflection

A signal travels down a trace, and wherever the impedance doesn't match (connector, via, solder joint), it bounces back like a ball off a wall. The reflected wave stacks on top of the main signal and muddies the following bits.

> Reflection is the room echo; the impedance mismatch is the wall that doesn't absorb.

## Two — the foam on the wall

**ODT (On-Die Termination)** is a termination resistor built *inside* the DRAM die and the controller — not on the board. Its job is to absorb the reflected wave, like acoustic foam on a recording-studio wall.

But the foam must match the room. In signal terms, the termination resistance Rtt must equal the line's **characteristic impedance Z0** — then the reflection dies entirely; if Rtt is off, the echo remains.

**What is Z0?** Z0 is the trace's own characteristic impedance, set by its geometry (width, height above ground, dielectric), and it's *independent of trace length*. It is not a resistor — a resistor burns energy and grows with length; Z0 is the instantaneous impedance the signal sees as it travels.

> Water-pipe picture: the trace is a pipe, Z0 is the pipe's diameter, the signal is the water. A mismatched connector at the end bounces some water back — that's water hammer, i.e. reflection. A connector exactly the pipe's diameter lets the water flow out cleanly.

Z0 is set by four geometric knobs: wider trace → lower Z0; closer to ground → lower Z0; higher dielectric constant → lower Z0; closer differential spacing → lower Z0. Single-ended lines land around 40–60 Ω (DDR5 typically 40–50 Ω). Because the board stackup and dielectric vary, Z0 drifts, so training sweeps Rtt values to find the best match instead of computing it.

## Three — two pairs of speaker and foam

Ron and Rtt both live inside the DRAM die, programmed through its mode registers. The die has two roles:

- **Ron** is the DRAM's *output driver impedance* — the speaker when the DRAM drives data out (the **read** direction).
- **Rtt** is the DRAM's *termination* — the foam when the DRAM receives data (the **write** direction).

Impedance matching pairs up *across chips*, one pair per direction:

| Direction | Speaker (who drives) | Foam (who terminates) |
|---|---|---|
| Read (DRAM → host) | DRAM's Ron | host's ODT |
| Write (host → DRAM) | host's drive strength | DRAM's Rtt |

So "DIMM Ron Training" and "DIMM ODT Training" are not one loop's pair — they are the DRAM's own two roles: speaker on read (Ron), foam on write (Rtt).

A DQ termination splits into five Rtt states, because DDR5 termination is not one fixed foam — it switches by rank role and read / write / idle:

| Rtt state | Who | When |
|---|---|---|
| RttWr | target rank | write (target is receiving) |
| RttNomRd | non-target rank | read (target is driving) |
| RttNomWr | non-target rank | write (target is receiving) |
| RttPark | non-target rank | idle |
| RttParkDqs | DQS line | idle |

In JEDEC terms: RttWr / RttPark live in **MR34 (§3.5.36)**, RttNomWr / RttNomRd in **MR35 (§3.5.37)**, CK/CS ODT in MR32 (§3.5.34), CA/DQS_PARK ODT in MR33 (§3.5.35).

**Why can't one Rtt rule them all?**

1. **The role changes**: the same rank is "target" one moment and "non-target" the next. The target's write termination and the non-target's echo-absorption want different values.
2. **The direction changes**: on a read, the target rank is the speaker (driving, can't terminate itself), and the non-target absorbs with RttNomRd/Park; on a write, the target is the foam (RttWr), the non-target uses RttNomWr.
3. **The non-target must terminate too** — and it's the *accomplice*, not the victim: in a fly-by topology, a write to the target passes the non-target's door; if the non-target's stub is unterminated, it becomes a mirror and reflects the signal back onto the target. Configuring the non-target's Rtt doesn't protect it (it isn't listening) — it silences the mirror.
4. **DQS is separate**: DQS is a differential strobe, not data, with its own routing and load, so it gets its own RttParkDqs.

**Why the speaker needs tuning too:**

Ron is the read-direction speaker (the DRAM's output driver impedance), the symmetric twin of Rtt. The speaker power must also match Z0 — and it splits into two transistors tuned separately:

1. **Symmetry**: read needs the DRAM speaker (Ron) to match Z0 *and* the host foam (host ODT) to match Z0. Tune only the foam and not the speaker, and the signal leaves already distorted.
2. **Pull-up vs pull-down**: the DRAM drives a 1 through the pull-up transistor (PMOS) and a 0 through the pull-down (NMOS); the two have asymmetric impedance, so they're tuned apart. (This is the same root cause as the P/N skew in per-bit de-skew.)
3. **Sweep, don't compute**: Ron drifts with process, temperature and voltage, so training sweeps it to find the widest read eye. Ron lives in the DRAM's MR5.

One nuance: Ron and host ODT are *each* matched to Z0 independently — they do not match each other. Ron anchors Z0 (source reflection goes to zero), host ODT anchors Z0 (far-end reflection goes to zero). So changing host ODT does not force Ron to change. The read eye is a joint function of Ron, host ODT, CTLE and DFE, so training re-centers the eye after each parameter — that's the usual coupling of all training knobs, not something special to Ron.

## Four — how to diagnose when the foam can't save you

ODT/Ron training fails when the speaker and foam are swept through every value, and the *best* setting still leaves the eye too small.

- **Whole eye small (all ranks, all lanes)**: likely a board SI problem — reflection, crosstalk or loss too severe for the foam and speaker to fix. Termination is the medicine; it can't cure a dead patient. Back to Part 3: lower the frequency or fix the board.
- **One lane small**: that one DQ trace's problem (length, crosstalk, contact) — chase that bit.
- **One rank small**: that rank's stub, contact or die — chase that rank (gold fingers, socket, die).

> An eye the termination and drive can't save has its root in signal degradation (reflection, crosstalk, loss). Bisect first: whole eye, one lane, or one rank.

**Golden rule: ODT is the foam on the wall — tune its thickness (Rtt) to match the room (Z0), and the echo dies before it muddies the next note.**
