---
layout: mypost
title: "DRAM Detective · Case File 0.2: SPD — the DIMM's ID Card"
categories: [DDR5, SPD]
lang: en
---

Halt! Who goes there? Plug a DIMM into a slot and the BIOS has no idea who just showed up — how big, how fast, who made it, nothing. Same as meeting a stranger: the first thing you've got to figure out is who they are and what they can do. And the bluntest way to do it? Check their ID card.

That ID card is the **SPD** — Serial Presence Detect. A tiny EEPROM soldered onto the DIMM, 1024 bytes, recording exactly one thing: *who am I, and what can I do?*

## How the BIOS reads it

Not through the fancy high-speed buses. It goes over **SMBus** — a slow, two-wire bus, one clock line and one data line. Crude, ancient, good enough. Every DIMM has a fixed address on this bus: `0x50` through `0x57`, one per slot.

Here's the interesting bit: the address isn't set by software. It's hardwired by a resistor on the DIMM's PCB, soldered at the factory. So every stick carries its own "room number" — plug it into any slot, its address is already decided. No collisions.

Where does that room number come from? Hardcore stuff: **one resistor encodes three address bits**. The SPD's 7-bit address is two halves — the top 4 bits `1010` are the "building number" (SPD device type, pinned by spec), and the low 3 bits are the "room number" HID, encoded by a resistor on the HSA pin. Eight resistor values = eight room numbers: 10KΩ→0x50, 15.4KΩ→0x51 … 196KΩ→0x57 (JESD300-5 Table 89).

Think of an old telephone keypad: one wire, each key strings in a different resistor, and the exchange measures the line voltage to know which key you pressed. DDR4's SPD needed three digital address pins (A0/A1/A2); DDR5 does it with a single analog pin. The room number is soldered at the factory, so the address is set before the stick ever ships.

## What's on the ID card

1024 bytes, nearly all of them meaningful. The ones you'll actually look at:

| Bytes | What it says | Example |
|---|---|---|
| Byte 2 | DDR generation | 0x12 = DDR5 |
| Byte 3 | Module type | 0x01 = RDIMM, 0x02 = UDIMM, 0x04 = LRDIMM |
| Byte 4 | Density (die count + die size) | 16 Gb ×8 |
| Bytes 20–21 | tCKAVG, the minimum clock period | 357 ps → 5600 MT/s |
| Bytes 30–37 | Timings: tAA(CL), tRCD, tRP, tRAS | 40-39-39-77 |
| Bytes 512–513 | Vendor ID (JEP106) | 0x80CE = Samsung |
| Bytes 521–550 | Part number (ASCII) | M321R4GA0BB0-CQK |

<iframe src="/animations/spd-id-card.html" width="100%" height="620" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="SPD ID card animation"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays · <a href="/animations/spd-id-card.html">open full screen ↗</a></p>

A few spots that bite — I fell into every one of them (JESD400-5B):

- **Vendor ID is big-endian** — Byte 512 is the continuation code, Byte 513 is the vendor code. Read it backwards and Samsung turns into someone else.
- **Timings are little-endian** — LSB in the low byte, the opposite of the vendor ID. Don't reuse the same parsing habit for both.
- The whole thing is guarded by a **CRC-16** at Bytes 510–511, covering 0–509. One corrupted byte anywhere and the checksum trips.

## One more thing: SPD isn't a fixed table

Don't assume SPD is one fixed layout — it differs in two dimensions:

- **DDR4 vs DDR5**: DDR4 is the old 512-byte format; DDR5 doubles it to 1024 bytes and switches to an "overlay" scheme (JESD400-5).
- **Module types within DDR5**: RDIMM / UDIMM / LRDIMM SPDs aren't wholly different — they're "common + specific". Bytes 0–239 are common (everyone uses them); Bytes 240–447 are written differently per module type. How does the BIOS know which layout to parse? It reads the **key bytes** first — Byte 2 = `0x12` (DDR5), Byte 3 = module type (0x01 RDIMM / 0x02 UDIMM / 0x04 LRDIMM) — then parses the rest accordingly.
- **LPDDR5 is a different beast**: it follows JESD209-5, not DDR5's JESD79-5, and it's usually soldered down with no DIMM at all, so the SPD mechanism is completely different. Leave it out for now.

## Real evidence from the log

Back to the boot log from Case File 0.1 — the order memory init runs:

```
Initialize Memory → Gather SPD Data → Socket DIMM Information
```

"Gather SPD Data" is one of the very first steps. The BIOS reads every SPD before moving on. And the DIMM census it prints afterwards — every field was copied off the ID card:

```
DIMM: Samsung   16GB (16Gbx8 SR)   4800 40-39-39   M321R2GA3BB6-CQKET
```

Vendor, density, speed, timings, part number — none of it guessed, all of it read from SPD. Evidence, not vibes.

## Want to read it yourself?

Watching the log is watching the BIOS read. But you can read it yourself — I wrote a UEFI Shell tool, **SpdTest** (the first app in `ddr5-shellkit`), that reads the whole SPD over SMBus, verifies the CRC, and turns it into plain English. Built from JEDEC specs only, no vendor NDA'd source.

Three commands:

```
SpdTest.efi scan                              # probe 0x50..0x57, list present DIMMs
SpdTest.efi read -c <ctrl> -ch <ch> -d <dimm> # read + parse + verify one DIMM
SpdTest.efi read -c <ctrl> -ch <ch> -d <dimm> -x  # also hex-dump the raw bytes
```

`read` on a real SK Hynix 16 GB CSODIMM:

```
===== SPD Summary =====
Module Type      : CSODIMM
Density          : 16 GB  (1 rank x 16 Gb x8)
Max Speed        : 6410 MT/s (tCKAVG = 312 ps)
Timings          : CL-52  tRCD-52  tRP-52  tRAS-103
Module Mfg       : SK Hynix (0x80AD)
Part Number      : HMCG78AHBVA312N
=======================
```

Source at [github.com/peterhu/ddr5-shellkit](https://github.com/peterhu/ddr5-shellkit). Those gotchas — big-endian, little-endian, CRC — are exactly the holes this tool fell into during debugging. Point `read -x`'s raw bytes at the table above and you'll spot `12` (DDR5), `80 AD` (SK Hynix), and the ASCII `HMCG78AHBVA312N` sitting right there.

## The golden rule

> **Name first, interrogate later.**

Training — the interrogation — comes later. Step one is always reading the SPD to learn who you're dealing with and what they can do. Skip it and you're configuring a memory controller blindfolded — what voltage, what timing, RDIMM or UDIMM — guessing the whole way. And guessing doesn't get you far.
