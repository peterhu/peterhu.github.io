---
layout: mypost
title: "DRAM Detective · Case File 0.0: The Fading Library"
categories: [DDR5, Training]
lang: en
---

DDR5 stores 0 and 1 in capacitors, finds cells by row and column address, and ships data out on both edges of a differential clock.

Every training step, in the end, is serving those three jobs — so before touching anything, get to know the ten pins.

<iframe src="/animations/fading-library.html" width="100%" height="720" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="The fading library — store, find, transfer"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/fading-library.html">open full screen ↗</a></p>

## One — the storage is a pile of leaky capacitors

The "D" in DRAM is dynamic. A bit is a capacitor: charged is 1, discharged is 0. The trouble is the capacitor leaks — leave it long enough and the charge drains away on its own, so the data has to be rewritten now and then. That's refresh. No refresh, no memory.

Row and column addresses select the cell, then the capacitor connects to the outside circuitry: discharging is a read, charging is a write.

Reading needs a helper. The signal on the capacitor is too weak to measure directly, so whenever a row is activated, the whole row is first moved into a sense amplifier (the row buffer), which amplifies it and judges 0 or 1. When the column address arrives, the right bit goes out.

Why not SRAM? SRAM latches with transistors and needs no refresh, but a bit costs six transistors; DRAM costs one capacitor plus one transistor — far cheaper and denser, at the price of refresh. Memory sticks use DRAM because that density-for-cost trade is the whole game.

## Two — double data rate and prefetch: carry more per trip

DDR's "double" means the clock CK carries data on both the rising edge and the falling edge — two beats per cycle.

But the memory array inside can't go faster: capacitor charge/discharge is physics, and the core frequency tops out around 100–400 MHz. To go faster outside, there's only one move — fetch more bits per access inside, then hand them out one by one on a much faster bus. That's prefetch.

- SDR: 1n, one beat per clock
- DDR1: 2n; DDR2: 4n; DDR3/DDR4: 8n
- **DDR5: 16n** — one burst is BL16 (Burst Length 16)

Like a warehouse: the mover inside works at one fixed speed, so have him carry more boxes per trip, and let the courier van outside take them box by box on both edges. DDR5 is the mover carrying 16 boxes a trip (16n prefetch), the van taking one box per edge.

## Three — from the controller down to one cell

Counting down from the CPU:

```text
CPU → Channel → Sub-channel → DIMM → Rank → Chip → Bank Group → Bank → Row / Column
```

- **Channel**: an independent set of buses out of the controller. DDR5 splits a channel into **2 sub-channels**, each with 32 DQ, 4 DQS pairs, and its own CA/CK/CS. This is DDR5's biggest change from DDR4: 1 channel = 2 sub-channels.
- **DIMM**: the memory stick.
- **Rank**: the group of chips hanging on one CS_n. The controller reads/writes one rank at a time, 64 bits (72 with ECC).
- **Chip**: one DRAM device, width x4 / x8 / x16.
- **Bank Group / Bank**: independent storage blocks cut out inside the chip, for parallelism.
- **Row / Column**: the actual cells inside a bank.

### Address and width — don't mix them up

Bank is not a 4×16 thing; it's two levels: at standard 16Gb density, **4 bank groups (BG[1:0]), each with 4 banks (BA[1:0]) — 4×4 = 16 banks**; the high-density 24Gb/32Gb parts have 8 groups × 4 banks = 32.

Why cut it so fine? Parallelism. If a chip were one big array, it could activate one row at a time, and you'd have to precharge before switching rows. Once split:

- Different banks can each activate their own row, because each bank has its own sense amplifier and row buffer.
- While bank A is being read or written, bank B can be activating in parallel — no idle waiting.
- Bank group is the grouping one level up, letting cross-group access parallelize even more.

Each bank is then laid out as row × column. A row is a line, and activating it reads the whole line into the sense amplifier; a column is a position, and when the column address arrives, the right slice is picked out of that row.

The easy thing to mix up is DQ — it's nothing like bank/row/col.

| Dimension | What it is | Who decides |
| --- | --- | --- |
| BG / bank / row / col | the address (which cell) | the address bits on the CA bus |
| DQ | the width (how many bits at once) | the chip's x4 / x8 / x16 config |

A column's width equals the DQ width. An x8 chip reads one column address and spits out 8 bits (DQ[7:0] side by side); x16 spits out 16 (DQ[15:0]). So a chip's capacity is:

```text
chip capacity = banks × rows × columns × DQ width
```

In the library: bank group is a wing (each wing lights itself), bank is a shelf (each shelf can pull its own layer), row is a layer (activate = move the whole layer to the desk), column is a spot on the layer, and DQ is how many books the librarian grabs side by side at once — 8 for x8, 16 for x16.

## Four — read/write in four steps, all waiting

Give an address and the data comes out? Not so fast. Reading or writing a DRAM is four steps:

1. **Activate**: bank address plus row address selects a row; the whole row moves into the sense amplifier (row buffer) — the gate opens.
2. **Read / Write**: the column address arrives; read a bit from the buffer, or write a bit in.
3. **Precharge**: to switch rows in the same bank, close the current row, write it back, reset the sense amplifier, and free the space for a new row.
4. **Refresh**: the capacitor leaks, so periodically rewrite every row — room-temperature spec is a full round within 64 ms. Auto-refresh for normal use, self-refresh for standby.

Between the steps there's always a wait; these intervals come up constantly in training:

| Timing | The gap between |
| --- | --- |
| tRCD | activate → ready to read/write |
| CL | read command → data on the bus |
| tWR | last write data → ready to precharge |
| tRP | precharge → ready to activate again |
| tRFC | how long one refresh takes |
| tREFI | how long between two refreshes |

Who configures, who trains, how each is computed, and which training step uses it — that's all in the 0.8 timing panorama; for now, just know the names.

## Five — what the ten pins do

| Pin | Full name | What it does | Direction |
| --- | --- | --- | --- |
| DQ[7:0] | Data | the data itself | bidirectional |
| DQS_t/DQS_c | Data Strobe | the data strobe (differential beat alongside DQ) | bidirectional |
| CK_t/CK_c | Clock | the clock (differential) | controller → DRAM |
| CA[13:0] | Command/Address | command and address (14 bits, encoded) | controller → DRAM |
| CS_n | Chip Select | chip select, which rank | controller → DRAM |
| CKE | Clock Enable | clock enable (enter/exit low power) | controller → DRAM |
| ODT | On-Die Termination | termination switch (absorbs reflections) | controller → DRAM |
| ALERT_n | Alert | error flag (parity error, open-drain) | DRAM → controller |
| DM/DMI | Data Mask / Invert | data mask + inversion | bidirectional |
| RESET_n | Reset | reset | controller → DRAM |

Two spots are easiest to confuse with DDR4:

1. **Commands ride a CA bus, not a pile of separate pins.** DDR4 has RAS#, CAS#, WE#, BA, A0-A15 each on its own pin; DDR5 folds them all into one **CA[13:0]**, with commands encoded — Activate, Read, Write, Precharge, Refresh each has a code. Case 0.6's "CA is the words, CS is the name call" is exactly this.
2. **DQS and CK are both differential.** A pair of lines, t and c, mirror each other up and down; the crossing point is the decision instant, naturally noise-resistant. Case 1.12's "differential needs no Vref" roots here.

## Six — DDR4 to DDR5, three changes

| | DDR4 | DDR5 |
| --- | --- | --- |
| command/address | RAS#/CAS#/WE#/BA/A separate pins | CA[13:0] encoded bus |
| prefetch | 8n (BL8) | 16n (BL16) |
| channel | 1 channel, 1 bus set | 1 channel, 2 sub-channels |

## Seven — walk the library once more

Stack it all onto a library: the books (data) are written in fading ink, and after a while the words go faint, so they must be re-inked on a schedule — that's refresh.

- **Bank Group / Bank**: the library is split into wings, each wing with a few shelves.
- **Row**: one layer of a shelf.
- **Column**: a spot on that layer.
- **Sense amplifier**: the librarian's desk. Activate is moving a whole layer onto the desk; read/write is taking or putting a book; precharge is returning the books and clearing the desk.
- **CS_n**: the doorbell — only the named rank (that set of shelves) answers; CA is the words — which wing, which layer, which spot, what to do.
- **DQS**: the beat that says "this book is valid now"; **CK**: the clock of the whole library.

**The ink fades, the cell is found by address, and the word goes out on both clock edges — learn the ten pins first, and you'll know which wire each training step is turning.**
