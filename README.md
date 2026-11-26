# POWERLINK Cycle Time Budget Calculator

A small, open, browser-based calculator for the minimum achievable
[Ethernet POWERLINK](https://en.wikipedia.org/wiki/Ethernet_Powerlink) cycle time of a given
network — and, more usefully, for seeing **which part of the budget is actually binding**.

**→ [Open the calculator](https://aygunspace.github.io/powerlink-cycle-budget/)**

No install, no dependencies, no build step. One HTML file.

---

## Why this exists

"How can I calculate the POWERLINK cycle time?" is a standing question for anyone sizing a
network, and it sits in the official FAQ as exactly that. The public answer is qualitative and
correct — it depends on the amount of data, the number of nodes, and how often they need to
communicate — but it is not a number you can put in front of a customer who is asking whether
40 axes will run at 400 µs.

Meanwhile the EtherCAT side has had an open web cycle-time calculator available for years.

This is an attempt at the POWERLINK arithmetic, in the open, with every assumption visible and
arguable. It is built entirely from the publicly documented cycle structure and standard
IEEE 802.3 frame timing. **No proprietary or vendor data is included.**

---

## What it computes

| Input | |
|---|---|
| Nodes polled every cycle | continuously polled CNs |
| Multiplexed nodes | low-priority CNs sharing a slot, spread over *n* cycles |
| Process data | PReq and PRes payload per node, in bytes |
| Poll Response Chaining | chained PRes without individual PReq |
| Infrastructure | hub / cut-through switch / store-and-forward, and hop count |
| Node turnaround | device-dependent PReq → PRes response time |
| Asynchronous phase | ASnd frame size |
| Idle margin | safety headroom, % |

| Output | |
|---|---|
| Minimum cycle time | and whether your target is achievable |
| Phase breakdown | SoC, isochronous, SoA, ASnd, idle — as a chart and a table |
| Binding constraint | which term to attack first if the target is missed |
| Infrastructure comparison | the same network on hubs vs cut-through vs store-and-forward |
| Bus utilisation | share of the cycle actually carrying frames |

---

## The headline finding

The infrastructure comparison is the part worth looking at first.

Every frame in a POWERLINK cycle crosses the network infrastructure, and with a PReq/PRes pair
per node that delay is paid twice per node, multiplied by hop depth. The POWERLINK
specification's own performance examples assume a **0.5 µs hub delay**. A cut-through switch is
around **5 µs**; store-and-forward is upwards of **10 µs**.

The result is that a topology decision made for convenience can cost more cycle time than any
amount of process-data tuning — by close to an order of magnitude in the unfavourable case. The
tool shows the three side by side against your target so the trade-off is visible rather than
discovered during commissioning.

---

## Method

**Frame timing.** Fast Ethernet at 100 Mbit/s → 0.08 µs per byte. Each frame on the wire costs
`38 + payload` bytes: 7 preamble + 1 SFD + 14 MAC header + 4 FCS + 12 inter-frame gap. The
POWERLINK header is taken as 24 bytes inside the Ethernet payload, and the payload is floored at
the 46-byte Ethernet minimum — which is why a frame carrying no process data still costs
**6.72 µs**.

```
frameTime(data) = (38 + max(46, 24 + data)) × 0.08 µs
```

**Cycle structure.** SoC → isochronous phase (PReq to each polled CN, PRes back) → SoA opening
the asynchronous phase → one ASnd frame → idle until the next SoC.

```
polled      = nContinuous + ceil(nMultiplexed / spread)
perNode     = PRC ? tPRes : tPReq + tPRes
cycleMin    = (tSoC + polled × perNode + tSoA + tASnd) × (1 + margin)
```

**Infrastructure delay.** Each frame is charged `hops × per-device delay`, added to every frame
in the cycle.

---

## Assumptions and limitations

Read these before quoting a number anywhere that matters.

- **Node turnaround is not from anyone's data sheet.** It is a parameter you must set from your
  own devices. The 1 µs default is a placeholder for sanity-checking, not a specification.
  Everything downstream of it inherits its error.
- **This is a budget, not a configuration result.** The authoritative figure for a real network
  comes from the engineering tool that configures it, against the actual device description
  files. Use this for feasibility screening and for reasoning about where the time goes.
- **Not modelled:** per-device internal latency differences, PRes timeout and error handling,
  hot-plugging, redundancy, SoA/ASnd scheduling policy across cycles, cross-traffic, cable
  propagation (negligible at these distances), and anything specific to a particular
  manufacturer's implementation.
- The protocol supports up to 240 nodes in one network; the arithmetic here does not enforce any
  other vendor-specific limit.
- Poll Response Chaining is modelled as "no individual PReq per chained node". This captures the
  essential saving but not the full chaining schedule.

Corrections are genuinely welcome — if an assumption here is wrong, I would rather know. Open an
issue.

---

## Running it locally

```bash
git clone https://github.com/aygunspace/powerlink-cycle-budget.git
cd powerlink-cycle-budget
open index.html        # or just double-click it
```

There is no toolchain. `index.html` is the whole application: vanilla JavaScript, canvas for the
charts, no external scripts.

---

## Licence

MIT — see [LICENSE](LICENSE). Use it, fork it, fix it.

---

## Disclaimer

Independent personal project by **Emre Aygün**. Not affiliated with, endorsed by, or
representing any automation manufacturer. Built from publicly available protocol documentation
and standard Ethernet frame timing. Figures are first-order estimates for concept-stage
reasoning and must not be used as the basis for a committed network design.
