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
