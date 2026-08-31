# The “does-everything” roadmap

This is the long-term checklist for turning AGaMEMnon from a useful experimental
toolchain into a broadly capable AG32 toolchain. For what works today, see
[STATUS.md](STATUS.md). For near-term priorities, see [ROADMAP.md](../ROADMAP.md).

The 2026-08-24 campaign is a useful snapshot: 25 successes, 52 designs that did
not route, 13 images that built cleanly but failed on hardware, 12 unusable or
unstable vendor references, and 3 incomplete tests. That is enough to show where
the hard problems are, but not enough to assign a meaningful “percent complete.”

## What done looks like

A reasonably complete toolchain needs:

1. an open encoder for the bitstream and configuration fields;
2. a device model covering the useful logic, routing, clocks, memories, IO,
   hard blocks, MCU interfaces, and packages;
3. placement and routing that works on realistic designs without hand-pinned
   routes;
4. correct synthesis and packing;
5. hardware results that agree with the logical model across the supported
   operating range;
6. clear errors for known unsupported cases instead of silently bad images;
7. boring installation, programming, recovery, APIs, docs, and releases.

The first two are much further along than the rest.

## Configuration

The fabric image can be thought of as three parts:

| Area | Current state | Remaining work |
|---|---|---|
| Global/preamble configuration | Generated openly; several PLL/clock profiles work | Decode and test the remaining clock/PLL modes |
| Design-neutral fabric defaults | Generated from open data and byte-identical to the decoded reference body | Name the remaining unknown/reserved fields |
| Design-specific configuration | Partial support for logic, routing, IO, BRAM, clocks and MCU interfaces | Complete the missing resources and modes |

The old dependency on copying a vendor-generated base image is gone. The main
problem now is understanding and correctly using everything layered on top of
that base.

## Routing and placement

Already useful:

- a large recovered selector database;
- exact selector formulas for some regular routing classes;
- conflict detection instead of guessing;
- positive isolated tests for all 14 edges that were once thought to be dead;
- many working small and peripheral-oriented routes on L48.

Still missing:

- a useful whole-chip coverage metric;
- several special feeders around IO, MCU, BRAM and clocks;
- reliable placement/routing for wide designs;
- similar routability for equivalent RTL and primitive-level versions;
- a better model for congestion and density;
- proper timing for clocks, IO, BRAM, PLLs, packages and PVT.

The 52 no-route campaign designs are the obvious regression set for router work.
A useful improvement should fix a class of them, not one hand-picked netlist.

## Logic and state

Small Boolean functions, shifts, arithmetic, LFSRs, counters, fanout and AHB
examples work in several tested configurations.

The important failures are `VP-AGM-001` and `VP-AGM-003` through
`VP-AGM-005`, covering feedback, FSM updates, rotate/reset and add/reset, plus
`VP-AGM-009`, where a moderately large state-heavy design simulated correctly
but diverged on hardware.

To close this area we need to minimize those failures, find the actual causes,
fix the general mechanisms, and add generated stress tests for placement,
reset, clock enables, feedback and density. A sealed holdout only becomes useful
after those rules stop moving every few days.

## MCU/fabric boundary

The retained `public32` map, smaller register banks, constant endpoints, status
overlays and local-interrupt examples give us a useful AHB slave path.

The obvious next steps are:

- make a fresh `regbank16` route;
- fix the placement problems exposed by `addsub16`;
- support application-owned state without replaying an exact old route;
- broaden address/control/burst/reset/clock behavior;
- eventually add AHB master and DMA support.

The current `public32` template is a replay of a reviewed route, not yet a
general register-bank generator.

## BRAM

There are working X13Y4 examples, but BRAM is still one of the least trustworthy
parts of the open flow. `VP-AGM-006` produced zero reads from new x1/x18
configurations even though the currently modeled INIT and configuration fields
looked right.

The next job is to find the missing static/read-path behavior. After that come
other sites, widths, ports, clocks, registered outputs, writes, mixed-width
modes, collision behavior and inference.

## PLL and clocks

Many HSE=8 divider/output-frequency points work. That does not solve clock
distribution: `VP-AGM-007` showed a five-site state design staying at zero even
though the routed logical model was correct.

Remaining work includes clock regions and seams, gating/reset, other PLL
outputs and HSE sources, phase/duty/feedback/bypass modes, and a real
placement-aware timing model.

## Carry

Carry works at several exact footprints: same-tile chains, one long X20
corridor, and one tested seam. It still needs wider placement coverage,
multiple simultaneous chains, branching rules, density tests and timing.

## IO and peripherals

The current positive set includes L48 outputs, some OE paths, UART0/1/2 TX,
SPI0/1 TX and I²C0/1 transactions. The main counterexample is `VP-AGM-008`:
new PIN_10/PIN_12 input paths and SPI0/SPI1 MISO paths did not work.

Long term this needs:

- generic input and bidirectional routing across supported pins;
- pull, drive, slew, Schmitt, bank-voltage and simultaneous-IO behavior;
- UART RX and more framing/mode coverage;
- repaired SPI MISO, then duplex, other modes, dual/quad and DMA/interrupt;
- wider I²C coverage including stretching, arbitration and simultaneous buses;
- timers, CAN, USB, Ethernet, ADC/DAC/comparators, RTC and peripheral DMA;
- independent testing of L64, Q32, L100 and the AG32VH/PSRAM parts.

## CPU-scale and real workloads

`serv-blinky` is a useful retained integration example, but replaying one known
route does not tell us whether fresh CPU-scale designs are reliable.

Before calling this area healthy we need repeatable fresh routing, direct BRAM
and register-file tests, much broader instruction/exception/interrupt coverage,
and several unrelated real applications.

## Toolchain quality

The backend also needs ordinary product work:

- reproducible Windows/Linux/macOS installation;
- deterministic database generation;
- useful placer/router failure diagnostics;
- reliable DAP/USB/UART programming and recovery;
- automated hardware-in-the-loop tests;
- stable project, SDK and primitive APIs;
- documentation generated from, or at least checked against, the same data used
  by the tests.

## Current priorities

1. Explain `VP-AGM-006` through `VP-AGM-009`.
2. Make the wide AHB/state designs place and route without one-off exceptions.
3. Fix physical input/SPI MISO before adding more RX features.
4. Automate more hardware testing.
5. Once the architecture rules settle down, build a sealed holdout suite.

The finish line is simple to describe: arbitrary-looking designs should usually
build, supported features should usually work, and when they cannot work the
tool should know why.