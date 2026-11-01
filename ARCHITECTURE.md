# Architecture

A high-level tour of how OHM2ENG is built and the decisions behind it. This describes the **design**, not the implementation - the source code is private.

---

## Principle: one circuit, swappable result producers

The single most important decision in the system:

> The browser **owns the circuit** - it draws it, edits it, and turns the schematic into a netlist. It does **not** own the answer. Where the numbers come from is a separate, swappable layer, and the checker that grades an exercise never learns which producer answered.

```
┌──────────────────────────── Browser (client) ─────────────────────────────┐
│  draw / edit   →   build netlist   →   pick a producer                    │
│  draw results  ◄─   update state   ◄────────────────┐                     │
└─────────────────────────────────────────────────────┼─────────────────────┘
             │                    │                   │            ▲
     in-page │               HTTP │            Web Serial          │
             ▼                    ▼                   ▼            │
┌───────────────────────┐ ┌─────────────────┐ ┌──────────────────────────┐
│ In the browser        │ │ Cloudflare      │ │ The OHM2ENG card         │
│ • the engine itself   │ │ Worker          │ │ real measurements from a │
│   (small circuits;    │ │ • the same      │ │ real breadboard          │
│   transient in a      │ │   engine        │ │                          │
│   background worker)  │ │                 │ │                          │
│ • pure digital logic  │ │                 │ │                          │
└───────────────────────┘ └─────────────────┘ └──────────────────────────┘
   default (small circuits)  large circuits       optional, USB-attached
```

**One engine, two homes.** The solver is a single module that runs **unchanged** in the browser *and* on the Worker. Any circuit up to 30 nodes is solved right in the page - no network round-trip per Run, with the transient in a background worker so the UI stays responsive and *Stop* is a real abort. Larger circuits go to the Worker, where a dense solve does not block the tab and the math runs under a bounded work budget. Same math, same results, chosen automatically by size.

**The digital exception.** Purely **digital** circuits (logic gates only, no analog) skip the numerical solver entirely and evaluate as **boolean logic in the browser** - there are no differential equations to integrate, just truth tables and clocked state. This keeps digital-logic labs instant and interactive.

---

## The solver

At the core is a SPICE-style numerical engine:

- **Modified Nodal Analysis (MNA)** assembles a system of equations directly from the netlist - one unknown per node voltage, plus branch currents for sources and inductors.
- **Linear systems** are solved by dense Gaussian elimination.
- **Nonlinear devices** - diodes, BJTs, MOSFETs, JFETs, op-amps - are linearized and solved by **Newton-Raphson iteration**, with GMIN stepping and voltage limiting to keep tough circuits converging instead of diverging.
- **Transient analysis** steps through time, replacing capacitors and inductors with companion models at each step.
- **Small-signal analyses** linearize around the DC operating point: AC / Bode, **noise**, the **transfer function** (gain, input and output resistance) and **pole-zero**.
- On top of those: **parameter sweeps**, **Monte-Carlo** tolerance runs (every R, C and L drawn inside its tolerance), and **Fourier / THD** of a transient, laid out the way SPICE prints it.

Devices are modelled the way SPICE models them - Gummel-Poon BJTs; MOSFETs with a smooth subthreshold-to-saturation channel, a body diode and capacitances; junction capacitance on diodes, BJTs and JFETs - and many parts take their numbers from the manufacturer's datasheet rather than from a generic default.

**Held to a reference.** A simulator that only agrees with itself proves nothing, so the engine is checked against **ngspice**, the open-source reference: tests compare it against golden ngspice output for SPICE export, noise, transfer function, pole-zero and Fourier, and differential fuzzers generate random circuits, run them through both simulators and flag any disagreement.

None of the theory is secret - it's the well-understood mathematics behind every SPICE simulator. The value is in getting it robust, accurate, and wrapped in a product that teaches.

---

## The hardware bridge - the same exercise, measured for real

An optional third producer sits beside the in-browser engine and the server: **a real circuit on a real breadboard**, read through **the OHM2ENG card** over Web Serial. One USB card both **measures** the breadboard and **feeds** it - a 4-channel scope and voltmeter, a function generator, a programmable DC supply, a current meter and five digital lines, all over the cable that also powers it.

The constraint that shaped everything: **the grading logic must not know which producer answered**. A measurement is mapped back onto the exact context an exercise's checker already consumes, keyed by component, so every existing check works unchanged against real voltages. Switching a task from *simulation* to *hardware* swaps the **producer** of the numbers - never the question, and never the standard it is held to. The circuit the student drew also decides what the card outputs: a generator on the canvas drives the card's generator, a supply drives the matching rail or the programmable source, and a part the card cannot provide is named, with the reason.

Five rules hold the layer together:

- **The transport is injected.** Nothing above the device abstraction knows a serial port exists - which is why the whole stack was built and tested against a mock card *before* any board existed.
- **The instrument is untrusted input.** Every number and string crossing the driver boundary is range-checked and scrubbed once, at that boundary - including the card's own claims about its resolution, ranges and features.
- **The card declares, the software adapts.** A board states what it can do; a missing capability stays on screen, greyed, with the reason - it never silently disappears.
- **It informs, it never locks.** No exercise becomes unsolvable because a board is missing, disconnected mid-answer, or uncalibrated. Simulation is always there.
- **"Not checked" is not "failed".** Calibration reports a distinct *blocked* state rather than a red ✗ - a false failure sends a student hunting a fault that isn't theirs.

Trusting a measurement takes a three-step proof: the link is alive, the measuring chain reads a reference **on the card itself** (never one wired by the student - otherwise a wiring mistake masquerades as a broken card, and vice versa), and a final sanity check confirms the card's own supply is what it should be. Only then does the indicator go green. The outputs stay disconnected until the card is ready, so nothing reaches the breadboard by accident.

A **compare** view then draws the simulated and the measured waveform on **one shared vertical axis** - two auto-scaled axes would make a 5 V square wave and 50 mV of noise look identical - aligns them by correlation before judging, and names the gap as gain, offset, or genuine shape.

---

## The learning platform

Beyond the simulator, OHM2ENG runs **genuine emulators** for its computer-architecture course - a Hack CPU, a RISC-V core, a VM translator, and a Jack compiler. Student code is **actually executed and checked against expected behavior**, not pattern-matched against a string. Grading that must not be guessable is verified server-side.

The same "learn by doing, then get verified" loop drives the **graded labs** - each section checked by solving the student's own circuit, reading its waveforms, or running its truth table - the interview-prep question bank, and a spaced-repetition scheduler that decides what to review and when.

**Lecturers** get a class view: students join a class with a code, and the lecturer follows their progress and study time. It is the one screen in the product that shows someone else's work, so access is decided on the server from the signed-in identity - the hidden menu entry is a convenience, not the guard.

---

## Stack

| Layer | Technology |
|-------|-----------|
| **Client** | Vanilla ES6 modules + HTML5 Canvas (no framework) |
| **Server** | Cloudflare Worker - Hono framework, TypeScript |
| **Solver** | Dense MNA + Newton-Raphson (isomorphic client/server), held to ngspice |
| **Hardware** | Web Serial → the OHM2ENG USB card, its own firmware |
| **Hosting** | Cloudflare Pages + Worker |
| **i18n** | Hebrew / English |

---

*The application is live at [ohm2eng.com](https://ohm2eng.com). This repository is a public overview; the full source is maintained privately.*
