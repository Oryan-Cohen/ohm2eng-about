# Architecture

A high-level tour of how OHM2ENG is built and the decisions behind it. This describes the **design**, not the implementation — the source code is private.

---

## Principle: a thin client, and a server that does the math

The single most important decision in the system:

> **All numeric simulation** — DC operating point, AC / frequency response, transient, Bode, and parameter sweeps — runs **server-side** in a Cloudflare Worker. The browser draws, edits, and turns the schematic into a netlist, then sends it to `/simulate` and renders the result that comes back.

**Why not solve in the browser?** A real circuit solve is heavy — a dense matrix factorization per operating point, repeated for every Newton iteration and every time step. Pushing it to the server keeps the UI smooth on any device, gives the math a predictable CPU budget, and keeps the numerically sensitive core in one place.

```
┌──────────────────────────── Browser (client) ────────────────────────────┐
│  draw / edit   →   build netlist   →   POST /simulate                     │
│                                                       │                    │
│  render results  ◄─   parse JSON   ◄──────────────────┘                    │
└───────────────────────────────────────────────────────────────────────────┘
                                  │  ▲
                             HTTP │  │ JSON
                                  ▼  │
┌────────────────────────── Cloudflare Worker (server) ─────────────────────┐
│  /simulate  →  MNA assembly  →  Newton-Raphson  →  Gaussian elimination    │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## The solver

At the core is a classic SPICE-style numerical engine:

- **Modified Nodal Analysis (MNA)** assembles a system of equations directly from the netlist — one unknown per node voltage, plus branch currents for sources and inductors.
- **Linear systems** are solved by dense Gaussian elimination.
- **Nonlinear devices** — diodes, BJTs, MOSFETs, op-amps — are linearized and solved by **Newton-Raphson iteration**, with GMIN stepping and voltage limiting to keep tough circuits converging instead of diverging.
- **Transient analysis** steps through time, replacing capacitors and inductors with companion models at each step.
- **AC / Bode** analysis linearizes around the DC operating point and sweeps frequency.

None of this is secret — it's the well-understood theory behind every SPICE simulator. The value is in getting it robust, fast, and wrapped in a product that teaches.

---

## One engine, two homes (isomorphic)

The solver is a single module that runs **unchanged** on the Worker *and* is bundled into the client. This makes a nice optimization possible: **small circuits can be solved entirely in the browser** with no network round-trip, while larger ones are sent to the Worker's bounded CPU budget. Same math, same results, chosen automatically by size.

---

## The digital exception

Purely **digital** circuits (logic gates only, no analog) skip the numerical solver entirely and evaluate as **boolean logic in the browser** — there are no differential equations to integrate, just truth tables and clocked state. This keeps digital-logic labs instant and interactive.

---

## The hardware bridge — the same exercise, measured for real

An optional third execution arena sits beside the server solver and the in-browser digital evaluator: **a real circuit on a real breadboard**, read through a small USB measurement card (an ESP32 driving an 8-channel ADC, over Web Serial).

The constraint that shaped everything: **the grading logic must not know which arena it is in**. A measurement is mapped back onto the exact context an exercise's checker already consumes, keyed by component, so every existing check works unchanged against real voltages. Switching a task from *simulation* to *hardware* swaps the **producer** of the numbers — never the question, and never the standard it is held to.

Four rules hold the layer together:

- **The transport is injected.** Nothing above the device abstraction knows a serial port exists — which is why the whole stack was built and tested against a mock driver *before* any board existed.
- **The instrument is untrusted input.** Every number and string crossing the driver boundary is range-checked and scrubbed once, at that boundary — including the converter's own claims about its resolution and reference.
- **It informs, it never locks.** No exercise becomes unsolvable because a board is missing, disconnected mid-answer, or uncalibrated. Simulation is always there.
- **"Not checked" is not "failed".** Calibration reports a distinct *blocked* state rather than a red ✗ — a false failure sends a student hunting a fault that isn't theirs.

Trusting a measurement takes a three-step proof: the link is alive, the ADC chain reads a reference **on the card itself** (never one wired by the student — otherwise a wiring mistake masquerades as a broken card, and vice versa), and a dedicated channel confirms the card is really attached to the breadboard. Only then does the indicator go green.

A **compare** view then draws the simulated and the measured waveform on **one shared vertical axis** — two auto-scaled axes would make a 5 V square wave and 50 mV of noise look identical — aligns them by correlation before judging, and names the gap as gain, offset, or genuine shape.

---

## The learning platform

Beyond the simulator, OHM2ENG runs **genuine emulators** for its computer-architecture course — a Hack CPU, a RISC-V core, a VM translator, and a Jack compiler. Student code is **actually executed and checked against expected behavior**, not pattern-matched against a string. Grading that must not be guessable is verified server-side.

The same "learn by doing, then get verified" loop drives the analog labs, the interview-prep question bank, and a spaced-repetition scheduler that decides what to review and when.

---

## Stack

| Layer | Technology |
|-------|-----------|
| **Client** | Vanilla ES6 modules + HTML5 Canvas (no framework) |
| **Server** | Cloudflare Worker — Hono framework, TypeScript |
| **Solver** | Dense MNA + Newton-Raphson (isomorphic client/server) |
| **Hosting** | Cloudflare Pages + Worker |
| **i18n** | Hebrew / English |

---

*The application is live at [ohm2eng.com](https://ohm2eng.com). This repository is a public overview; the full source is maintained privately.*
