# OHM2ENG

![License](https://img.shields.io/badge/license-all%20rights%20reserved-b0413e)
![i18n](https://img.shields.io/badge/i18n-Hebrew%20%7C%20English-4a7fb5)
![Runs on](https://img.shields.io/badge/runs%20on-Cloudflare%20Workers-f38020)

> A browser-based electrical circuit simulator with a deep, interactive education layer.
> **Draw a circuit → press Run → get real DC / AC / transient / Bode / sweep results** — plus full engineering & computer-science courses built on real emulators.

🌐 **Live:** [ohm2eng.com](https://ohm2eng.com)

*A public overview of OHM2ENG. The full project source is private — access is available on request (see [Status](#status)).*

![OHM2ENG — a diode rectifier circuit simulated live, with the oscilloscope showing input vs. rectified output](docs/screenshot.png)

<sub>*A half-wave rectifier running live — the oscilloscope shows the input sine (red) against the rectified output, solved server-side and drawn back in the browser.*</sub>

---

## Purpose

**OHM2ENG exists to be the single place a student needs to succeed — everything, in one browser tab, instead of scattered across a dozen separate tools.** It brings together a canvas for building real circuits with advanced virtual instruments, structured courses (from digital circuits to a comprehensive computer-architecture track that *begins* by building a computer from a single NAND gate and continues on into RISC-V and modern CPU internals), job-interview preparation, and an automatic grading system that checks a student's work against the correct answer. The philosophy is *learn by doing, then get verified*: draw it, run it, build it — and have it graded — with progress tracking and spaced repetition keeping you on course. A teaching platform first, and a serious circuit simulator second.

---

## What is this?

OHM2ENG does two big things:

1. **Analog + digital circuit simulator.** Place components on an HTML5 canvas — resistors, transistors, op-amps, sources, logic gates, ICs — wire them up, hit **Run**, and read results back through real virtual instruments: **oscilloscope, multimeter, Bode plotter, logic analyzer, curve tracer**.

![A guided RC low-pass filter lab with the Bode plotter showing the frequency roll-off](docs/screenshot-lab-bode.png)

<sub>*A guided lab: build an RC low-pass filter, then run an AC sweep on the Bode plotter and watch the gain roll off past its ~159 Hz cutoff — the task panel gives you the theory and checks your circuit.*</sub>

2. **An electrical-engineering & CS learning platform.** A 16-lecture digital-circuits course; a comprehensive **computer-architecture course** whose *first part* is building a working computer from a single NAND logic gate up (through **Hack**, a **VM**, and a **Jack compiler**), and which then goes well beyond that foundation into **RISC-V, datapath & control, pipelining, caches, and virtual memory** — with genuine emulators at every layer; **job-interview prep** (a career center of graded EE/CS questions); and interactive labs — all wrapped in a Leitner **spaced-repetition** scheduler, cloud progress sync, and plan tiers (Guest / Pro).

![Building an XOR gate from NOT, AND, and OR gates in the computer-architecture course](docs/screenshot-logic.png)

<sub>*The computer-architecture course in action: build an XOR gate from NOT / AND / OR primitives, then have it graded against its full truth table — one step on the path from a single logic gate up to a working CPU.*</sub>

---

## The core idea: thin client, the server does the math

The single most important architectural fact:

> **All numeric simulation (DC / AC / transient / Bode / sweep) runs server-side** in a Cloudflare Worker.
> The browser does **not** solve circuits — it draws, edits, and `POST`s to `/simulate`.

The solver is a dense **Modified Nodal Analysis (MNA)** engine with **Newton-Raphson** iteration and full device models, given a bounded CPU budget on the Worker. Purely digital circuits are the exception — they evaluate client-side (boolean logic, no differential equations to solve).

**→ For a deeper look at the design, read [ARCHITECTURE.md](ARCHITECTURE.md).**

---

## Optional: solve the same exercise on real hardware

Every exercise is fully solvable in simulation, and always will be. But a student who wants to can **build the circuit on a breadboard**, plug a small USB measurement card into the computer, and answer the *same* task with *real* measurements.

The point is that nothing about the exercise changes. The grader does not know which arena it is in: a real measurement is fed into the exact context the checker already consumes, so the same question, checked the same way, is now answered by a physical circuit. There is also a **compare** view that plots the simulated and measured waveforms on one shared axis and names the difference — gain, offset, or genuine shape — which is where the interesting learning actually happens.

Design rules worth stating: the bridge **informs, it never locks** (no exercise becomes unsolvable without a board); the instrument is treated as **untrusted input** and validated at one boundary; trusting a measurement requires a **three-step calibration** proved against a reference on the card itself, not against the student's own wiring; and the whole feature stays **hidden from students who have never connected a board**, so nobody meets it before it means anything to them.

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| **Client** | Vanilla ES6 modules + HTML5 Canvas (no framework) |
| **Server** | Cloudflare Worker — Hono framework, TypeScript |
| **Solver** | Dense MNA + Newton-Raphson (~5,600 lines, isomorphic client/server) |
| **Hosting** | Cloudflare Pages + Worker |
| **i18n** | Hebrew / English |

---

## Highlights

- **171 components** across **19 categories** in the parts library
- Real device physics: BJT (Ebers-Moll), MOSFET (Shichman-Hodges), diodes, thyristors, op-amps (soft-saturation + oscillator modes)
- Draggable virtual instruments with FFT, Bode, and cursor measurements
- **Comprehensive computer-architecture course** — its first part builds a computer from a single NAND gate up (Hack / VM / Jack), then it extends well beyond into RISC-V, datapath & control, pipelining, caches, and virtual memory; student code is actually executed and graded, not pattern-matched
- **Job-interview prep** — a career center of graded EE/CS questions
- **Optional hardware bridge** — build the circuit for real, read it over USB, and answer the same graded exercise with real measurements; a compare view puts the simulated and measured waveforms on one axis
- SPICE netlist import / export
- Electrical Rule Check (ERC)
- Spaced-repetition mastery tracking + two-way cloud progress sync

---

## Try it

It's **live and free to start** — open **[ohm2eng.com](https://ohm2eng.com)** in any browser, with nothing to install. Jump straight in as a guest, or create an account to save your progress across devices and follow the full course tracks. Works on desktop and tablet, in English or Hebrew.

---

## Status

> **This repository is only a condensed overview — a summary of what OHM2ENG is.**
> The **complete project**, documented in far greater detail, lives in a separate **private repository**: the full source code (client, Cloudflare Worker solver, all course content and emulators), the in-depth engineering handbook, the CI pipeline, and the full test suites — everything. Access to the full private repository can be granted on request — feel free to reach out (for example, via LinkedIn).

The application is live at **[ohm2eng.com](https://ohm2eng.com)** — no setup required. This is a proprietary project — see [LICENSE](LICENSE).

© OHM2ENG. Built with vanilla ES6, HTML5 Canvas, and Cloudflare Workers.
