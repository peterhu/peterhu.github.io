---
layout: mypost
title: "DRAM Detective · Case File 0.7: The Built-In Test Worker"
categories: [DDR5, Training]
lang: en
---

The memory controller has a built-in self-test engine — a piece of hardware that generates patterns, sweeps addresses, and checks returned data, all at memory speed.

Training uses it to measure the eye. It reuses the normal pins, switched by a mode bit: one path to the DRAM, two ways to drive it.

<iframe src="/animations/built-in-test-worker.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The built-in test worker — fill, fire, knock"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/built-in-test-worker.html">open full screen ↗</a></p>

## One — one path, two ways

The engine is an algorithmic pattern generator, a piece of hardware built-in self-test (BIST). It sweeps addresses, issues commands, and compares data at memory speed, with no CPU involvement per address — that's the fundamental difference from a software memtest.

It isn't a separate chip, and there's no second wire to the DRAM. Physically it shares the same DQ/DQS/CK/CA pins with the memory controller; logically it's a separate path, switched by a mode bit inside the controller. Flipped to test mode, the engine takes over the physical interface, drives DQ with its own pattern buffer, and bypasses the normal read/write queue.

It has to be this way: the DRAM has one physical interface, so there's no room for a second test bus. The test engine lives inside the memory controller, reuses the PHY, and a mode bit flips between them.

One path to the DRAM, two ways to drive it: normal mode is a courier company delivering by address; test mode is the same courier putting on a test uniform, stamping data, scanning addresses — skipping normal sorting.

## Two — four tools

Think of it as a worker who can only do one job: carry a stamp, walk a fixed route, stamp and check as it goes, jotting down every mismatch. Blazing fast, rigid as a board.

| Tool | Hardware | What it does | maps to |
| --- | --- | --- | --- |
| the stamp | the data pattern buffer | data values | pattern (static / walking / cacheline / inverted) |
| the scanner | the address instruction | the address route | address, direction |
| the action list | the command instruction | write / read / hammer | write / read / read-write |
| the recipe | the algorithm instruction | step order | the step sequence |

These four are the engine's hardware registers, not concepts invented in the language above.

### Flexibility for the human, rigidity for the hardware

The engine itself is rigid — four fixed tools, fixed patterns, fixed addresses, fixed actions. But you can put a small language on top, giving flexibility to the script writer and leaving rigidity in the hardware:

```c
test flip01 {
    address = whole_rank;
    write(static(0x5555555555555555), up);
    read(static(0x5555555555555555), up);
    read_write_inv(static(0x5555555555555555), up);
}
```

Each line lands on a different tool: `address = whole_rank` is the scanner setting the route; `write(static(0x55…))` loads the stamp and the recipe draws the "write" card as step one; `read(...)` draws the "read" card as step two; `read_write_inv(...)` draws read then write-inverted (an invert bit), reading 0x55… and writing ~0x55… to test whether the cell can flip.

The human writes a flexible script — write / read / read-write, patterns, directions, in any order. Compile it down and it still lands on those four fixed instructions. Flexibility lives in the language, rigidity in the hardware, and a compiler translates between them.

## Three — three parties, plus knocking

One measurement, three parties:

1. The host fills the form: which pattern, which addresses, how many passes.
2. The engine fires: writes the start bit, bumps the fire count.
3. The engine drives traffic: reads and writes to the DRAM.
4. The DRAM answers: returns the data.
5. The engine checks: compares and counts.
6. The host knocks: polls the done flag, waiting.
7. The host collects: reads the error bitmap.

The striking one is step 6 — the engine doesn't ring a bell when it's done (no interrupt), so the host polls the done flag over and over.

## Four — the unavoidable whys

**Why hardware, not the CPU?** Memory testing has to run at memory speed; any software instruction drags it down and shifts the timing. So address-sweeping, command-issuing, and comparing are all fixed in hardware, and the CPU only shows up at the start and the end.

**Why a stamp, not per-address data?** The data generator is a hardware buffer — load one pattern once, stamp it uniformly over every address. Per-address data would need per-address CPU intervention, and the most precious thing, memory speed, would be gone.

**Why one pattern per round?** Changing the pattern isn't changing a variable; it's stop, reload the buffer, re-take-over, and rerun — all hardware loading. Multiple patterns per round are naturally ruled out.

**Why no bell, why must the host knock?** During training the host is already blocked: after firing, the firmware has nothing else to do but wait. In that state, polling and interrupts do the same job, but polling is much simpler hardware — one less interrupt line, one less interrupt controller. (This is a designer's-view inference — the source doesn't say why, flagged for verification.)

**Finally, credits.** It's the count of in-flight read completions (pipeline depth), not a buffer size, not a parking lot. It's a sweet-spot value: too small and the pipeline idles; too big and too many reads are in flight, their timing trampling each other. The engine runs write-read turnaround-heavy traffic — the bus keeps about-facing — so it needs a shallow pipeline for determinism. In short, credits are work-in-progress in the pipeline.

## Five — four hard boundaries

What the engine can and can't do, in four lines:

| Boundary | Capability | Actual use | Consequence |
| --- | --- | --- | --- |
| the data generator is a stamp | one pattern stamped uniformly | static / inverted / walking / cacheline | no arbitrary per-address data |
| the address generator is a scanner | fixed-order sweep | one order, three directions | no arbitrary address list |
| pattern-per-round | one pattern per round | invert on, background rarely | no multiple patterns per round |
| timing in, voltage out | timing inside the engine, voltage is the machine's job | — | the script doesn't touch voltage |

**A built-in test worker: the CPU fills the form, fires the engine, and knocks on the door — the rest it does by itself, blazing fast, but only this one job.**
