---
layout: mypost
title: "DRAM Detective · Case File 1.4: Eyes on All Sides, Steady at the Center (Read 2D Centering)"
categories: [DDR5, Training]
lang: en
---

Reading is **standing blindfolded on an invisible diamond island**. The water is everywhere around you — step off and you're gone. All you can do is tap the ground and ask, *steady?* The host maps the island with two rulers: **east–west is timing** (the sampling point — *when* you sample), **north–south is voltage** (the decision line Vref — *how high counts as a 1*). Walk the whole island, find its exact center, and stand there — farthest from all four shores, where neither timing jitter nor voltage noise can push you off. **2D Centering sweeps out this island's 2D map and stands at its center.**

<iframe src="/animations/2d-centering-island.html" width="100%" height="640" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="2D Centering diamond island animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/2d-centering-island.html">open full screen ↗</a></p>

## The door and the eye

The read direction has **two steps**, each doing one job:

```
① Open the door right (Receive Enable) — let the DQS strobe in.
② Sample the data right (2D Centering) — latch the DQ correctly.
```

| | Receive Enable (1.2) | 2D Centering (1.4) |
|---|---|---|
| Handles | **the door** (DQS gate) | **the eye** (data eye) |
| Cares about | **when DQS arrives** | **where the eye's center is** |
| Tunes | the gate's open moment | the sampling point + the Vref decision line |
| Dimensions | **one** (time: phase and which beat) | **two** (timing X × voltage Y) |
| Touches Vref? | **no** (log Vref Count = 0) | **yes** (33 Vref points, Vref Count = 16425) |
| Feedback | did DQS arrive (one 0/1 or a count) | did data read back right (CPGC writes a pattern, reads back, compares bit by bit) |
| Object | the strobe **DQS** | the data **DQ** |

**Receive Enable aligns the strobe DQS; 2D Centering centers the data DQ eye.**

Receive Enable answers only "did DQS arrive, did the door open right" — it never asks whether the data was sampled correctly. **A correctly opened door does not mean the data was correctly sampled.**

### Why catching DQS is not the same as catching DQ

DQS and DQ are **source-synchronous, walking hand in hand** — the DRAM sends them out together, nominally aligned but actually a hair off. The host latches DQ with the DQS edge, but a small skew **tDQSQ** sits between them.

So Receive Enable opens the door, DQS walks in, **and DQ follows it in**. But DQS being inside doesn't tell you whether DQ is being read as 0 or 1. Judging DQ needs two more things, neither of which Receive Enable touched:

- **Sampling point**: at which DQS phase do you latch DQ.
- **Vref**: how high counts as a 1, how low counts as a 0.

Receive Enable only opened the door; the sampling point and Vref are still at defaults. DQ comes in, but it can't be judged reliably yet — **that's exactly the gap 2D Centering fills.**

The same DQS phase is tuned by two different steps, for two different purposes:

| | Tuning the DQS phase to… |
|---|---|
| Receive Enable | **the door's** position — open it at DQS's arrival to catch it |
| 2D Centering | **the sampling point's** position — drop it in the eye's center so DQ is judged right |

**Why both?**

```
Receive Enable opens the door first; 2D Centering then centers the eye.
Only then does "open the door" mean "get the right data."
```

1. **Receive Enable first** — open the door at the instant DQS arrives, or the data is lost.
2. **2D Centering second** — the door is open and data is in, but the sampling point and decision line aren't centered yet; the read window is small, and one timing jitter or one voltage noise flips a bit. Tune both dimensions so all four margins are maximal, and the data lands steady.

- Receive Enable **sets the door by the schedule** — learn what time the postman (DQS) arrives, open the door then; the postman is in but not yet signed for.
- 2D Centering **is the receiving desk** — now that the postman is in, tune *when* to sign (sampling point) and *how high counts as a valid signature* (Vref), maximizing the window.

> **Receive Enable handles "caught"; 2D Centering handles "caught steady." One is the door, one is the eye.**

---

## What a read burst looks like

A read burst's DQS runs three segments:

```
preamble tRPRE (DQS toggles a few times) → data burst BL (DQS toggles, latching data) → postamble tRPST (DQS holds a moment)
```

The host knows all three lengths — the spec fixed them or wrote them into a Mode Register:

| Segment | Parameter | Who sets it |
|---|---|---|
| Preamble | tRPRE (read preamble length) | spec / MR8 |
| Data | BL (burst length, DDR5 default BL16) | spec / MR |
| Postamble | tRPST (read postamble length) | spec / MR |

**BL's unit is per DQ line**: BL16 means each DQ line carries 16 bits in one burst (16 transfers × 1 bit each); BL8 (DDR4) means 8 bits. The whole bus: 64 DQ lines × BL16 = 1024 bits = 128 bytes per burst.

**Preamble is one per burst, not one per cacheline**: a cacheline is 64 bytes (not 64 bits); one 128-byte burst is 2 cachelines, and **the preamble always follows the burst.**

**When does DQS end? The host counts it out** — preamble, burst, and postamble are all agreed beat counts. The moment the host sends READ it knows which beat DQS starts and which beat it ends; the gate shuts the instant the postamble ends.

> DQS is a train: it whistles the preamble on the way in, unloads, and flashes its tail-light on the way out. The host holds a timetable, so it doesn't judge when the train leaves — it counts the beats, and shuts the gate the moment the tail-light dies.

---

## One — why the read side needs two dimensions

The read DQS is born uncertain (JESD79-5 §4.7.2.1 Table 250):

| Uncertainty | spec value | meaning |
|---|---|---|
| **tDQSCK** (position) | ±0.240~±0.300 tCK | DQS's **arrival position** is uncertain, different per die |
| **tDQSCKi** (jitter) | 0.410~0.490 tCK | DQS's **jitter** inside one die |

Position + jitter make the read DQS a moving target — **shaking on the time axis, unsteady on the voltage axis.**

**Note: tDQSCK and tDQSCKi are both timing-axis uncertainty, not voltage.** The source comment pins it down: tDQSCK is DQS's rising-edge **position** relative to CK; tDQSCKi is DQS's rising-edge **variance window** (how wide it jitters). Both live on the time axis (earlier/later), not the voltage axis (higher/lower). **Voltage is Vref, a separate, independent parameter.**

So the 2D shmoo's two axes:

| Axis | Dimension | matching uncertainty |
|---|---|---|
| **X (left–right)** | timing | tDQSCK (position) + tDQSCKi (jitter) |
| **Y (up–down)** | voltage | **Vref** (decision line) |

One dimension alone is never enough:

- Tune **timing** only (sampling point): the point is centered, but the voltage decision line isn't — the voltage dimension stays off.
- Tune **voltage** only (Vref): the line is centered, but the sampling point isn't — the timing dimension stays off.

> **2D Centering doesn't tune timing alone; it centers timing and voltage together.** Timing is the X axis (left–right); voltage Vref is the Y axis (up–down); sweep both to find the eye's true center.

| Type | Examples | Has MR? | Tunable? |
|---|---|---|---|
| **Programmable parameters** | CWL, CL, tWPRE | yes | host writes the MR |
| **Intrinsic (spec only gives an upper bound)** | **tDQSCK, tDQSCKi**, tAA | no | fixed by silicon, can't be changed |

### tDQSCK/tDQSCKi are intrinsic, not programmable — 2D Centering doesn't tune them, it absorbs them

2D Centering **doesn't tune tDQSCK/tDQSCKi**; it tunes the host's own **sampling point and Vref**, to absorb the uncertainty these two intrinsic parameters cause:

| Layer | changed? |
|---|---|
| **Parameter layer** (tDQSCK/tDQSCKi) | no (intrinsic to the DRAM, can't change) |
| **Response layer** (sampling point + Vref) | yes (stand in the eye's center) |
| **Effect layer** (can it sample reliably) | from "no idea where the target is" → "standing on the bullseye" |

**How each one gets absorbed:**

- **tDQSCK (position uncertainty)** — the eye's position on the time axis is unknown; sweep the DQS phase (X axis) to find the eye's time-axis center.
- **tDQSCKi (jitter)** — each read's DQS arrival wobbles inside the window; the eye's edges blur and shrink; sweep out the actual, jitter-shrunken eye and stand in its center.

> tDQSCK/tDQSCKi are like a target that wobbles furiously by nature; 2D Centering is adjusting the gun mount to the target's center of motion (the host tunes *itself*, not the target). The target still wobbles, but you've pushed the muzzle to its center, so hit rate is maximal.

---

## Two — what is the eye?

Superpose hundreds of thousands of bit waveforms and a **clean diamond region** remains — that safe region where sampling is stable is the eye.

Three elements:

- **The diamond (the eye itself)**: the safe region where the waveform's top and bottom edges overlap.
- **The decision line (Vref)**: a horizontal cut that splits 0 and 1 on the voltage axis.
- **The sampling point (DQS phase)**: decides *which instant* on the time axis you sample.

When the sampling point sits dead center, all four margins are thick — the signal is steadiest.

**First, keep two "Vrefs" straight:**

| | What it is | Who sets it |
|---|---|---|
| **Decision-line Vref (threshold)** | **one value** — how high counts as a 1 | the knob you tune |
| **Vref eye height** | **a range** (eye top − eye bottom) | measured; set by signal quality, not directly tunable |

- **The decision line** is the ruler "above Vref → 1, below Vref → 0" — a single value.
- **The eye height** is, at one sampling point, **the range of Vref where 0/1 is still judged correctly** when you sweep Vref low to high. The upper bound is the Vref where "1" starts being misread as "0"; the lower bound is where "0" starts being misread as "1"; the span between them is the eye height.

So training tunes **the one threshold knob**; the eye height is a swept-out range (physical reality, can't be enlarged). In the end you plant the threshold in the middle of the eye height, farthest from both the eye's top and bottom.

**A trap: eye height → 0 does not mean the center (reference Vref) → 0.**

- Eye height = v_high − v_low (**the range**) — at the diamond's left/right tips it does shrink to 0.
- Center = (v_high + v_low)/2 (**the reference Vref**) — it goes to the **mid-voltage ≈ 0.55 V** (the middle of a 0~1.1 V swing), not 0.

Geometrically: the left/right tips pinch into a point **at mid-height** (squeezed vertically flat) — that point's voltage is still the mid-value 0.55 V, not the bottom 0 V.

**And the sampling point isn't even at the real tip**: ±T sits at **1/3 of the way out** (T = min(12, EW/2)/3), not at the true tip — that's exactly why the `/3` exists: don't probe the true tip (eye height = 0 there, degenerate, no center to measure); at 1/3 out the eye is still open (narrower than center, but non-zero), so its center is **a voltage you can actually measure.**

So: **narrow eye height ≠ failed center**. Eye height is "how wide"; center is "where"; the two are unrelated. The reference Vref uses "where" (the mid-voltage), not "how wide."

### Why are the decision line (Vref) and sampling point (DQS phase) orthogonal?

Because they cut **two independent physical quantities**:

Voltage level and timing are unrelated — raising the decision line doesn't change when the sampling point arrives; moving the sampling point earlier doesn't change how high the line sits. So the two dimensions are **orthogonal**, and you can tune them independently and sweep them into one 2D picture.

> The decision line is the standard for "how tall counts as tall"; the sampling point is "what time you pick up the parcel." Raise the ruler too high and short people get mistaken for tall; go to pick up the parcel too early and you make the trip for nothing.

---

## Three — the 2D shmoo (2D Centering's true body)

**The eye diagram and the shmoo aren't the same thing, but both describe the same object — the eye.**

| | Eye diagram | Shmoo |
|---|---|---|
| What it is | **continuous** diamond from overlaid waveforms | **discrete** pass/fail grid |
| How it comes about | hundreds of thousands of overlaid waveforms (oscilloscope view) | sweep two knobs (timing × Vref), testing each combo read-right-or-not |
| Axes | voltage × time (the signal's natural axes) | timing (sampling point) × Vref (decision line) — the host's two knobs |

**Relation: the shmoo's green region — the pass cells — is like the eye's open diamond.** The shmoo is tested cell by cell; the eye is overlaid continuously.

**Why does training use the shmoo, not the eye diagram?** Because the hardware has no oscilloscope to see a continuous waveform — it can only set timing and Vref to some value, fire a pile of data, read it back and compare, and see whether that combination is right. Sweep all combinations and plot the passing cells, and you get the shmoo's green region — the eye's discrete version.

**Why "no oscilloscope, no continuous waveform"? It's not that it can't measure the signal — it can't measure the *analog waveform*, only the *digital right-or-wrong*:**

| | Eye diagram | Shmoo |
|---|---|---|
| What sees it | **oscilloscope** (a physical instrument outside the chip, probe on the host/DRAM pin) | **the training hardware itself** (comparator + CPGC) |
| What it reads | **continuous** voltage waveform (analog) | **right / wrong** (digital, one pass/fail) |

The lab oscilloscope can read the real waveform on both host and DRAM pins at once — *that's* where the eye's diamond comes from. But the training hardware inside the chip takes a completely different path: the receive path's **comparator**, at the sampling point (`RxDqsDelay`) and the decision line (Vref), crushes one instant of analog voltage into a 0/1; then CPGC writes a pattern, reads back, compares lane by lane, and ends with nothing but a pass/fail.

**The crux is "the comparator crushes to 0/1":** the slicer latches only one instant per bit and only one Vref threshold. Once latched, the voltage curve's rising edge / falling edge / noise-jitter information is all discarded — all that remains is whether this bit is a 0 or a 1. So the training hardware **can never see the diamond from its own receive path** — each look yields only a "read right / read wrong" binary answer at one (Vref, phase) point.

**Then how does the eye's shape get recovered?** Move (Vref, phase) around cell by cell, record where right/wrong flips, and the green region's outline emerges. Like the blind men and the elephant — can't see the whole thing, but feel it point by point and the outline appears. The shmoo's green region is the eye's discrete version.

**Why not just install an oscilloscope in the training hardware?**

1. **Area/cost** — one DQ is one lane; dozens of lanes each get one comparator. Add a high-speed ADC and waveform storage per lane and the silicon area and power explode — while training never needs the continuous waveform. Pure waste.
2. **Information is enough** — the only decision training makes is "where's the green region's center," which needs only per-point pass/fail, not the voltage curve itself. One per-lane comparator plus an error counter is enough.
3. **Division of labor** — the eye diagram is a debug tool **for humans** (why is SI bad, where's the reflection, how much jitter); the shmoo is a runtime tool **for firmware** (where's safe, where's the center). Same object (the eye), two instruments to see it — one analog-continuous, one digital-discrete.

> The oscilloscope is a camera taking continuous photos of the eye; the training hardware is blind, poking cell by cell and asking only "does this cell hurt?", until the pain-free region outlines the eye's shape.

**From another angle, the eye diagram is the shmoo's superset.** The eye is the continuous full portrait (every time × voltage point carries information); the shmoo only samples pass/fail at grid points, so the shmoo is the eye's discretized/quantized version. The only subtle difference is axis semantics: the eye's axes are signal level × time; the shmoo's axes are the host knobs Vref × sampling instant — the same 2D space, relabeled.

> The eye is a continuous photo; the shmoo is that photo pixelated, counting pass/fail per cell — a subset sampled out of the eye.

**The word "shmoo" comes from a round, blobby creature in the 1948 American comic *Li'l Abner*.** Test engineers noticed the passing region's shape was plump and round just like it, and named the 2D pass/fail plot a "shmoo plot." It's a comic-strip pun.

Lay timing and voltage on one picture — **X axis is timing (DQS phase), Y axis is Vref (decision-line height)** — test each cell read-right-or-not, and that's the shmoo:

```
Vref ↑ (voltage / decision line)
 high │ . . . . . . . . . .         . = read wrong (fail)
      │ . . . . █ █ █ . . .         █ = read right (pass), green = eye open
      │ . . . █ █ █ █ █ . .
      │ . . █ █ █ █ █ █ █ . .
 mid  │ . . █ █ █ ★ █ █ █ . .       ★ = best point (green region's geometric center)
      │ . . █ █ █ █ █ █ █ . .
      │ . . . █ █ █ █ █ . .
      │ . . . . █ █ █ . . .
 low  │ . . . . . . . . . .
      └────────────────────────→ timing (DQS phase)
      left                      right
```

**2D Centering sweeps this picture out cell by cell, finds the green region (the eye), then takes its geometric center ★.**

---

## Four — why the center, not the edge?

Because **the center maximizes all four margins**:

- **Timing jitter** (DQS phase wobbles left–right) — at the center, left and right both have room; the wobble can't leave the green.
- **Voltage noise** (Vref wobbles up–down) — at the center, up and down both have room; the noise can't leave the green.

Stand at the edge and one twitch in that direction drops you out of the eye — a wrong bit.

> Stand dead center and you're on the bullseye — however the arrow (timing jitter) and the wind (voltage noise) push, you're still on target. Stand at the edge and one slip is a miss.

### tDQSQ is fixed, but its effect is absorbed in two steps

Earlier we saw a small skew **tDQSQ** between DQS and DQ. It's an intrinsic spec parameter (JEDEC79-5, upper-bounded), a physical property of the DRAM — **not itself trained**, the host can't change it. But the DQ-eye-offset-from-DQS it causes must be compensated, and it's **different per bit** (each DQ bit skews from DQS differently).

| Who | Absorbs what |
|---|---|
| **2D Centering** | sweep the DQS phase to find the DQ eye's **overall** center — absorbs the **common** tDQSQ |
| **per-bit de-skew** | each DQ bit de-skewed one by one — absorbs the **per-bit** tDQSQ differences |

2D Centering sweeps **one** DQS phase, one sampling point, so it can only find the *overall* eye center — it can't handle how much each bit individually deviates. Per-bit de-skew (called right after 2D Centering in the source) aligns bit by bit, absorbing each bit's tDQSQ difference.

**Why "common," not "average"? Because *measuring* and *adjusting* are two different granularities:**

- **Measuring is per-bit** — CPGC keeps one error counter per lane and judges pass/fail bit by bit; it reads each bit's eye boundary clearly.
- **But the knob is shared** — 2D Centering tunes one per-DQS sampling point (`RxDqsDelay`, one value for a whole group of DQ bits); one knob can only turn to one value, so it physically cannot give each bit a different sampling point.

So per-bit pass/fail judgment's job is to take the **intersection** of all bits' eyes and keep the worst bit inside — but the result can only be **one overall center**, absorbing the offset all bits share; each bit's residual skew is left to the next step's per-bit de-skew.

> DQS is **the single baton in a choir conductor's hand**; the DQ bits are **a row of singers following the beat**. One baton, one sweep — **the whole choir moves together**: that's 2D Centering's sampling-point knob, which can only make the whole choir early/late together, absorbing the choir's **common** timing offset. But the conductor can **hear** each singer's rushing (per-bit pass/fail, measured finely) while holding only one baton (shared knob, can't single one out) — so he sets a beat that keeps even the worst rusher on time, giving the worst one a safety net. To fix one singer's rushing, you'd have to hand **each singer their own metronome** (per-bit de-skew).

> **tDQSQ is fixed and untrainable; its common offset is absorbed by 2D Centering, its per-bit differences by a separate training step.**

---

## Five — why does 2D Centering take so long?

**First, the big picture: finding the center of a diamond island, blindfolded.**

On the ground sits a **diamond safety island** (= the eye / green region). Around it is water — step off and you fall in (read wrong = fail); step on the island and you're steady (read right = pass). You're **blindfolded** (the training hardware can't see the waveform), so all you can do is "tap once and ask, steady?" The goal: **stand at the island's exact center**, farthest from all four shores, where neither timing jitter nor voltage noise can push you off.

Two rulers: **east–west = timing** (sampling point `RxDqsDelay`, *when* you sample), **north–south = voltage** (decision line `Vref`, *how high counts as a 1*).

```
        north (voltage high)
         ↑
      ╱ ╲
     ╱   ╲      ← nearer the left/right edges, the vertical span shrinks
    ╱  ★  ╲     ← center: vertical span thickest
   ╱       ╲
  ╱╲       ╱╲   ← left/right tips: the vertical span pinches to a point
west ←───────────→ east (timing)
        south (voltage low)
```

**Coarse sweep (one-dimensional Vref coarse-tuning — map the skeleton first):**

1. **Measure the width** — stand on a fixed north–south line (Vref parked at default), tap east until you fall in (right wall), tap west until you fall in (left wall) → you know "how wide" the island is. (sweep timing, default Vref)
2. **Pick 3 east–west points** — center + left + right. (the diamond is narrower vertically near the edges; measuring only the center misses that trap)
3. **Measure the heights** — stand on each of the 3 points and walk north–south → you know "how tall" the island is at each point. (sweep Vref, timing parked at that point)
4. **Take the shortest height as the reference** — of the 3, pick the shortest. **The shortest comes from the left/right points, not the center — the center is actually the thickest** — so taking the shortest is taking the worst case; don't let the center fool you. (narrowest Vref eye height → reference Vref)

**Fine sweep (2D fine sweep — fill in the body):** around that reference line, sweep up/down 16 steps × across the full width, cell by cell, and outline the whole island (the shmoo green region), then stand on the geometric center ★.

**Why coarse then fine:** sweeping the whole grid cell by cell = 33 Vref steps × full width × 2 ranks × ≤3 iterations ≈ 25 seconds. The coarse sweep first spends a small cost to fix "about how tall the island is," so the fine sweep only sweeps offsets around it (16 steps up/down) instead of absolute voltage from zero.

Here's the log bill for 2D Centering Late: **25443 ms**, 35.41 million knocks, 25486 CPGC tests, 198 samples, 16425 Vref sweeps.

```
198 samples (= 33 Vref × 2 ranks × ≤3 iterations)
  × 129 CPGC tests / sample
= 25486 CPGC tests (write start_test=1, one ignition)
  × 1390 knocks / test
= 35.41 million knocks (read the test_done bit)
  × 0.72 µs / knock
= 25443 ms = 25.4 seconds
```

**Each cell = one ignition + repeated knocking.** Ignition is setting CPGC's `start_test=1`, after which the CPGC engine writes a pattern to memory, reads it back, and compares — an automatic grading machine, CPU never touching the data. Then the host **reads the `test_done` bit every so often to ask "graded yet?"**, until all channels are done — knocking is the most time-eating link in those 25 seconds.

**The scan structure, four layers outside-in:**

```
outer   33 Vref offsets (voltage Y axis, ±16 steps around the coarse reference)
  └─ middle  2 ranks (each rank measured once)
       └─ inner  ≤3 iterations
            └─ innermost  sweep the DQS phase (timing X axis, find the eye's left/right edges)
                       │ first big step 8 to approach the edge
                       │ then fine step 1 to hug it
                       └─ each cell = one ignition + knocking
                            ignition = CPGC writes pattern → reads back → compares
                            knocking = reading test_done ≈ 1390 times
                                 └─ per-bit pass/fail (read the error counter)
```

**Where does the reference Vref come from?** 2D Centering doesn't jump straight to 2D. First a **1D Vref coarse-tune** roughly sets a Vref reference (the decision line's initial height), then the 2D fine sweep runs ±16 steps around it.

```
① 1D coarse-tune Vref (set the reference decision line first)
     ↓
② 2D fine sweep (16 steps up/down around the reference = 33 steps × DQS phase × rank × iteration)
     ↓
③ per-bit de-skew (Case File 1.5)
```

**How is the 1D Vref coarse-tune implemented? Sample 3 timing points first, then sweep Vref at those 3 points:**

1. **First measure the timing eye width** — at default Vref, sweep timing (DQS phase) to find the eye's width on the time axis (left/right edges).
2. **Pick 3 timing points** — inside the timing eye, take the center and one each side, 3 points total (T = min(12, EW/2)/3, i.e. center 0, sides ±T).
3. **Sweep Vref once at each of the 3 points** — move timing to each point and measure the Vref eye height (upper/lower edges on the voltage axis).
4. **Aggregate, take the narrowest Vref eye height to set the center** — of the 3 points, take the narrowest and use it to set the reference Vref.
5. **The fine sweep follows** — the 2D fine sweep runs ±16 steps around this reference.

**How do you sweep timing, exactly? Ever played minesweeper?**

- **What you change**: the sampling point `RxDqsDelay` (per-DQS read sampling phase — the timing X-axis knob).
- **How you change it**: add/subtract a number to the current `RxDqsDelay` — "+8/−1" = shift right 8 / left 1.
- **Step**: coarse step 8 to approach, fine step 1 to hug the edge.

From the current sampling point P0 (P0 is pass), probe once right, once left:

```
right coarse probe: +8 → test → pass → +8 → test → pass → … → hit the wall FAIL
right fine hug:     −1 → test → … until just pass = right edge R
left coarse probe:  −8 → test → … → hit the wall FAIL
left fine hug:      +1 → test → … until just pass = left edge L
eye width = R − L
```

Each step costs the same as the 2D fine sweep: one ignition, then knock and read the error counter to judge pass/fail per bit.

> Finding a wall: take a big step 8 across until you hit it, then a small step 1 back to hug the base; hug both walls and the span between them is the eye's width on the time axis.

**Why sample 3 timing points instead of one?** Because the eye is a **diamond**, not a rectangle — **the Vref eye height is narrower near the timing edges**:

```
  near the timing edges the Vref eye height shrinks
        ╱ ╲
       ╱   ╲      ← center: Vref eye height thickest
      ╱  ★  ╲
     ╱       ╲
    ╱╲       ╱╲   ← edges (±T): Vref eye height narrows
```

Sweep Vref only at the eye's center and you get the *thickest* Vref eye height — but the center being widest doesn't mean the whole timing range is safe. So also sample at center ±T, and **take the narrowest of the 3 to set the center**, guaranteeing the *worst point* in the timing range still survives.

**Why is T = min(12, EW/2)/3?** EW is the eye width swept out in step 1:

- **EW/2** — the half-width: the furthest you could walk from center to edge.
- **min(12, EW/2)** — capped at 12. However wide the eye is, probe at most 12 PI ticks out — too close to the edge is the diamond's **tip**, where Vref margin is already at 0, thin and noisy, no information.
- **/3** — take 1/3: the side points sit 1/3 of the way from center to edge, not all the way to the edge.

So the three points are { −T, 0, +T }. In numbers: eye width ≥ 24 PI ticks (half-width ≥ 12) gives T = 4, points { −4, 0, +4 }; a narrow eye gives T = EW/6, shrinking linearly. The two side points **deviate at most ±4 PI ticks from center** (1 tCK ≈ 128 PI, ≈ 1/32 tCK) — the diamond narrows so fast once you leave center that you don't need to walk far.

The side points must be **non-zero** (or you'd never see the narrowing), must not **reach the edge** (tip degenerates), must not **wander far** — 1/3 is the sweet spot among the three. The 12 and the 1/3 are characterized empirical values, like the 33 Vref steps — not derived.

The "0" among the three points isn't the geometric center — it's the **current sampling setting as the origin**; ±T is an offset from it. Strictly, that 0 is the current `RxDqsDelay`, not (left + right)/2.

> **Coarse = sweep timing first to fix 3 points (eye center + sides), then sweep Vref at those 3 and take the narrowest height as the reference; fine = sweep the full timing × 33 offsets around that reference, cell by cell.**

**Why offsets, not absolutes?** Coarse already fixed a good starting point; fine only micro-tunes around it — sweeping offsets beats sweeping absolute Vref: cheaper and more focused.

**Why 33 Vref points?** They are **33 offsets around one reference Vref** — 16 steps up + 16 down + center 0 = 33. But **the step isn't fixed** — the source computes it dynamically: **step ≈ (max eye height + 8) / 33, range ≈ ±(max eye height + 8) / 2**, with the eye height measured in step 3. (That comment's {0, +2…+32, −2…−32} is only a placeholder default table; the real code overwrites it dynamically.)

**Why dynamic, not hardcoded 2?** Because the step must guarantee the eye's tallest and shortest points both measure 0 — the swept range must cover the full voltage height of the eye. Eye height differs per DIMM/rank, so the step follows it: taller eye → bigger step, wider range. The +8 adds a little slack to ensure you sweep past the eye's top and bottom (read-wrong = 0), pinning the edges. (The unit is Vref *code*, not mV; mV per code depends on the platform's Vref resolution — to be confirmed.)

**Why measure each of the 2 ranks separately?** Because **fly-by makes each rank's clock (CK) arrive at a different time** (see Case File 1.3), so **each rank's eye sits in a different place** — timing center and Vref center both differ.

- The same DQS phase and same Vref is the eye center for rank 0 but may be off for rank 1.
- So 2D Centering **measures each rank once and sets each one's own eye center**, writing back its own sampling point and Vref.

**Inner ≤3 iterations?** Not repeating the same thing 3 times — each round swaps in a different **bus turnaround interval (t_rrsr)** and re-measures margin, as a **stress test**.

- **t_rrsr** is the read-read same-rank turnaround interval (a JEDEC timing parameter): the bus "U-turn" gap between two back-to-back READ commands.
- **Tighter t_rrsr** (smaller gap) → reads more back-to-back → less bus turnaround/recovery time → **the eye is squeezed smaller**.
- So it uses **3 t_rrsr levels** (tight/mid/loose), measures margin each, and **takes the worst (tightest t_rrsr) result** — guaranteeing the eye is still big enough and sampling still steady when the bus is busiest in real running.

> Measure "how wide the eye can open" not just on an empty road, but also in bumper-to-bumper rush hour — try 3 traffic densities and take the worst jam's result, and only then is it truly stable.

### Why sweep the edge coarse-then-fine (big step 8 to approach, fine step 1 to hug)?

2D Centering sweeps the eye's edge **big-step first, small-step second**:

- first a **step of 8** (coarse) to charge right until it overshoots and hits the edge;
- then a **step of 1** (fine) to back up and hug the edge exactly.

(**Units**: 8 and 1 are both **PI ticks** — the phase interpolator's smallest division; 1 tCK ≈ 128 PI ticks, so step 8 = 1/16 beat, step 1 = 1/128 beat, both **sub-beat** precision.)

**Why?** Finding an edge must be both **fast** and **exact**: step 8 charges up fast to save time; step 1 hugs the edge for precision. All-1 stepping crawls too slowly; all-8 stepping pins the edge only to within 8 — too coarse.

**Key: `RxDqsDelay` (sampling point) ≠ `RecEnDelay` (the door).** The same PHY has two per-strobe DQS delays — don't mix them:

| knob | what it is | tuned by | case |
|---|---|---|---|
| **`RecEnDelay`** | the gate's open moment | Receive Enable | 1.2 |
| **`RxDqsDelay`** | the sampling point's phase | 2D Centering | 1.4 (this file) |

Receive Enable tunes `RecEnDelay` (when the door opens); 2D Centering tunes `RxDqsDelay` (once the door is open, at which eye phase to sample) — one is the door, one is the eye.

### Why does CPGC use a pseudo-random LFSR instead of fixed all-0/all-1?

The pattern CPGC writes and reads back isn't fixed all-0 or all-1 — it's an **LFSR (linear-feedback shift register) pseudo-random sequence, rolled on the fly**.

**Why?** A fixed pattern **misses errors only a specific bit sequence triggers** — crosstalk that only "0101 alternating" excites, adjacent-lane interference that only "a long run of 1s then a 0" triggers. A pseudo-random sequence covers every bit combination, simulating worst-case real data.

Better still: CPGC uses **two different LFSR seeds** — one for the aggressor lane, one for the victim lane — **deliberately creating adjacent-lane crosstalk**, testing whether one DQ toggling disturbs its neighbor. This echoes "crosstalk is the troublemaking neighbor": CPGC actively plays the troublemaker to test what the system can take.

> Fixed all-0/1 is a rote-memorization test; a pseudo-random LFSR is a random full exam paper — rote questions miss the ones you can't answer, random questions reveal your real level.

---

## The golden rule

> The geometric center of the eye leaves margin on all four sides — timing jitter and voltage noise can't shake it. Eyes on all sides, steady at the center.
