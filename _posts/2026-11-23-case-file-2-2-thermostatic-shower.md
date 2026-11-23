---
layout: mypost
title: "DRAM Detective · Case File 2.2: The Thermostatic Shower"
categories: [DDR5, Training]
lang: en
---

The clock tree has three stages: the PLL multiplies a slow reference into the fast DDR clock, the DLL aligns that clock and slices out 0°/90°/180°/270°, and the PI fine-tunes between adjacent phases. It works like a shower — the PLL is the water heater, the DLL is the thermostatic valve, the PI is the shower head.

The valve has only eight notches, and every time you turn the water off and on, it can land one notch off. That is exactly why training takes the tightest margin: to survive whichever notch it lands on.

<iframe src="/animations/the-thermostatic-shower.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The thermostatic shower — how fast vs how late"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/the-thermostatic-shower.html">open full screen ↗</a></p>

## One — from a slow clock to many phases

Case 2.1 ended with the PI seesaw, and its two ends are I (0°) and Q (90°). Those two clocks don't come from nowhere — the clock tree builds them stage by stage:

```text
slow reference ──PLL ×N──► fast clock ──DLL align──► multi-phase (0°/90°/180°/270°)
```

The PLL (phase-locked loop) owns frequency: it multiplies the slow reference into the fast clock DDR needs. The DLL (delay-locked loop) owns phase: it aligns that clock and slices out fixed phases. The PI owns the finest slice: it interpolates between two adjacent phases.

Water heater → thermostatic valve → shower head. "Move DQS right by 3 ticks" walks this whole chain. Remember the split of labor: the PLL handles how fast, the DLL handles how late, the PI handles that one finer tick.

## Two — shifting phase is delay, not frequency

"Shifting phase" trips everyone up, so let's kill the confusion: shifting phase is delay, not frequency. The DLL never touches frequency — its output frequency always equals its input frequency. All it does is move the same clock edge a little later in time.

Three words first:

| Word | What it means | Who owns it |
| --- | --- | --- |
| frequency | cycles per second, fast or slow | PLL |
| phase | where in the 360° cycle the edge sits | DLL, PI |
| delay | the time equivalent of a phase difference, early or late | DLL, PI |

Phase and delay are two names for the same thing:

```text
one cycle T = 360°
90° of phase = T/4 of time   ← a quarter cycle later
```

Inside the DLL is a delay line — a chain of buffers. The signal slows a little at each buffer; tapping out at the Nth buffer gives a clock that is N notches late. The point: it is still the same frequency, just later. So 0°, 90°, 180°, 270° are not four frequencies — they are four copies of the same clock, each delayed a different amount:

| Phase | Delayed by |
| --- | --- |
| 0° | 0 (reference) |
| 90° | ¼ cycle |
| 180° | ½ cycle |
| 270° | ¾ cycle |

Why delay at all? To line two signals up in time. DQS and DQ reach the DRAM at different times — their traces differ in length and load. Training nudges DQS a little later so its edge lands in the center of the DQ eye. That nudge is shifting phase, which is delay. "Move DQS right by 3 ticks" means three delay ticks.

One picture to lock in the split of labor:

```text
reference ──PLL──► fast clock      frequency only (slow → fast)
fast clock ──DLL──► multi-phase     delay only (same frequency, 0/¼/½/¾ cycle later)
adjacent ───PI───► finer phase      delay only (interpolated between two phases)
```

All three stages leave frequency alone; only early and late change.

Another way to see it: runners on a track, one lap = one cycle. Frequency is laps per minute (fast or slow); phase is where on the track a runner is right now. The PLL makes everyone run faster (frequency). The DLL doesn't change the speed — it moves one runner's starting point back a little (delay), so that runner is always a fixed angle behind. Moving the start line is shifting phase.

## Three — the PLL inside: an oscillator that tunes itself

The PLL has to make "N times the reference", and it does that with an oscillator (VCO). How fast the VCO runs is set by a control voltage: higher runs faster, lower runs slower.

How does it land on exactly N times, on its own? A feedback loop:

1. First divide the VCO frequency by N, to shrink it to the reference's scale so they can be compared.
2. Compare: too slow, raise the voltage; too fast, lower it.
3. The voltage moves, the VCO frequency moves, the divided result moves — compare again.
4. Repeat until the divided result exactly equals the reference, then lock.

At the moment it locks, VCO frequency = N × reference. Multiplication done.

Think of tuning a guitar: the tuning fork is the reference (the standard pitch), the string is the VCO (pitch is frequency, turning the peg is the control voltage), the ear is the phase detector (hears the gap), the hand is the charge pump and filter (turns by however much it's off). The only difference — you don't tune to the same pitch, you tune to exactly N times it. The ear keeps listening, the hand keeps turning, until it's right.

This analog circuit lives in the PHY silicon. The firmware only ever sees its register interface — enable, lock, and the ratio — never the VCO, the divider, or the phase detector's transistors.

## Four — the DLL: eight notches, three of them

The DLL locks by analog tuning, not by hitting an exact value in one shot. Its tuning is 3 bits — eight notches — and the lock point is pieced together across those eight notches.

| Field | Width | Meaning |
| --- | --- | --- |
| tuning | 3 bits | analog tuning, sets the DLL's delay (8 notches) |
| lock | 1 bit | lock detect, 1 means the DLL is locked |
| reset | 1 bit | reset the DLL |
| lock status | 1 bit each | timeout / weak lock / long lock, three separate flags |

There are three DLLs in the PHY (not the one inside the DRAM — that's a different story), each with its own job:

| DLL | What it aligns | Granularity |
| --- | --- | --- |
| data | the DQ/DQS data path — aligns DQS to the center of the DQ eye | one per DQS strobe (per byte lane) |
| command | the command/address (CA) bus — aligns CA to CK | one per sub-channel |
| clock | CK itself, the root phase | one per channel |

Set the target temperature (the lock point), and the thermostatic valve adjusts the hot/cold mix on its own (3 bits = 8 notches) until the water holds steady. All the firmware does is read the lock flag, over and over, and wait for it to turn on.

## Five — why turning the water off lands a notch off

Every time training adjusts a knob — Vref, ODT, EQ — it resets that DLL and has it lock again.

The problem is the word "lock". It isn't a one-shot exact value, it's analog convergence: the tuning is only 3 bits, eight notches, and the lock point is pieced together across them. On every relock the landing spot can jitter a little and fall onto the neighboring notch. The same board might lock on notch 5 today and notch 6 tomorrow — both are normal.

Turn the water off and on, and the valve re-mixes to roughly the same temperature, just a touch different. That's why per-bit de-skew resets the DLL twice — off and on — and takes the tightest of the two: whatever notch it lands on, sampling must still not fail. The margin training leaves behind is the cushion for exactly that one notch of drift.

## Six — a few whys

Why not merge the PLL and the DLL into one? Because one owns frequency and the other owns phase. The PLL has a VCO and can change frequency; the DLL has only a delay line and can only move phase later, never change frequency. Neither can do the other's job.

Why is it called delay-locking and not frequency-locking? Because the frequency is already fixed by the PLL; all that's left is to align the phase — that is, delay a fixed amount. No VCO, so the circuit is simpler than a PLL, locks faster, and doesn't accumulate jitter the way a PLL does.

Why three DLLs? Because data, command, and clock are not the same road:

- **Different lengths**: DQ/DQS is point-to-point straight to the DRAM — short and well matched; CA and CK run fly-by, through the RCD, long and heavily loaded. With that much spread, one delay line can't lock all three.
- **Different targets**: the data DLL aligns DQS to the DQ eye center, the command DLL aligns CA to CK, and the clock DLL is the root phase the first two take as reference.
- **Different granularity**: data has one strobe per byte lane and each lane drifts differently, so each must be locked on its own; command is one bus per sub-channel; clock is one per channel. Lock them separately and they never drag each other.
- **Power, independently**: in standby you can power down just the data DLL to save power (data and command Vref even have different exit latencies). Couple all three together and you can't shut off just one.

When you read those delay values in a training log, walk back up this chain and you'll know who handed each tick to you.

**The PLL decides how fast the clock runs; the DLL decides how late it arrives — phase is delay, not frequency. Every relock lands a notch off, so training takes the tightest margin to survive that one notch.**
