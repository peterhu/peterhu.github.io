---
layout: mypost
title: "DRAM Detective · Case File 0.1: Memory's Family Tree"
categories: [DDR5, Topology]
lang: en
---

Here's my first rule of memory debugging: don't rush into the code. Figure out which *level* the problem lives on first. The memory system has a strict hierarchy — from the CPU all the way down to the DRAM dies — and every level is its own suspect. Point at the wrong level, and you'll chase the wrong guy.

Here's the family tree:

```
CPU
 └── IMC
      └── Channel
           └── DIMM
                └── Subchannel
                     └── Rank
```

<iframe src="/animations/topology-map.html" width="100%" height="740" style="border: none; border-radius: 12px; display: block;" loading="lazy" title="Memory topology interactive map"></iframe>
<p style="text-align: center; color: #888; font-size: 13px; margin-top: 6px;">Auto-plays top to bottom · <a href="/animations/topology-map.html">open full screen ↗</a></p>

**IMC** — the memory controller, the part of the CPU that actually talks to memory. It issues commands, decodes addresses, runs the training. A server CPU has several IMCs, each minding its own channels.

**Channel** — an independent bus out of the IMC. Commands, addresses, data all ride on it. The key thing: channels run in parallel, so when one dies, the others don't care. That "mind your own business" independence is exactly what makes Channel such a clean diagnostic boundary.

**DIMM** — the stick in the slot, the thing you can actually pull out and swap. Two slots per channel, D0 and D1.

**Subchannel** — new in DDR5. Each DIMM is split into two independent channels, 40 bits each (32 data + 8 ECC; non-ECC is 32 bits). A DDR5 DIMM is basically two half-width channels tied together, used like two independent memory channels. DDR4 didn't have this. (JESD79-5 §14.1)

**Rank** — the group of DRAM dies in one subchannel that share a chip-select. One rank per subchannel = single-rank (SR); two = dual-rank (DR).

This part trips people up — I got the order wrong myself at first. **Subchannel cuts the stick in half; Rank groups the dies inside each half.** So Subchannel sits above Rank. Don't flip them.

The hierarchy is a bit like the address on a parcel, from big to small: country → city → street → house number. Reading a byte of memory works the same way — the IMC routes through Channel → DIMM → Subchannel → Rank → Bank → Row/Col, level by level.

Let's look at a real log. Dual-socket DDR5 server, 8 channels per socket, 2 slots per channel. Notice the naming `N{Socket}.C{Channel}.D{Dimm}`:

```
N0.C00.D0: 16GB
N0.C01.D0: 16GB
N0.C02.D0: 32GB
N0.C04.D0: 32GB
N0.C06.D0: 32GB
N1.C00.D0: 16GB
N1.C01.D0: 16GB
N1.C04.D0: 32GB
N1.C05.D0: 32GB
N1.C07.D0: 32GB
```

(Platform, DIMM vendors, SPD serial numbers redacted.)

So what's this info good for? It's how you quickly pin down which level the problem is on. The trick: **check whether this level's siblings died too.**

- Both ranks on a DIMM fail → the DIMM is the problem (siblings died together, so it's the common parent)
- Only one rank fails, the other trains fine → that rank is the problem (its sibling survived)
- All DIMMs on one channel fail, other channels are fine → that channel is the problem
- All channels on a socket fail → the socket or board is the problem (shared power/controller)

Simple logic: if the failure respects a boundary — everything inside dies, everything outside lives — then the fault sits on that boundary or above it. One trick to rule them all. Almost every case file after this will use it.

So next time training fails, don't rush to tune parameters. Pull out this trick and ask: **did its siblings die too?** With the family tree in hand, we can start knocking on doors.
