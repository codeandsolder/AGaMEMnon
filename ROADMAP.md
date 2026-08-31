# Roadmap

For what works today, see [docs/STATUS.md](docs/STATUS.md). This page is the
shorter list of what to fix next.

The August 2026 campaign found 25 working model/vendor/open comparisons, 52
designs that did not route, and 13 images that built cleanly but failed on
hardware. The next useful work is therefore mostly correctness and routability,
not recovering one more obscure feature.

## P0: do not regress known working designs

- Keep retained routes/images reproducible.
- When a reviewed checkpoint changes, inspect the route/config difference rather
  than blindly updating its hash.
- Keep exact known-bad images/logical graphs blocked.
- Keep qualification records append-only so old conclusions can be corrected
  without erasing the history.

See [docs/LANDING_A_CHIPDB_CHANGE.md](docs/LANDING_A_CHIPDB_CHANGE.md) for the
checkpoint workflow.

## P1: explain the current hardware failures

These are more important than adding adjacent features because the existing
logical/router checks did not catch them.

| Defect | Problem | Next useful experiment |
|---|---|---|
| `VP-AGM-001` | MCU read-data feedback diverges on hardware | Minimize with matched route/placement variants |
| `VP-AGM-003` | FSM loses a next-state bit | Trace clock/data/config on the smallest reproducer |
| `VP-AGM-004` | four-bit rotate/reset vehicle fails | Separate reset/startup from rotate datapath |
| `VP-AGM-005` | add/sub forms share a bad reset state | Minimize the common reset/update path |
| `VP-AGM-006` | initialized BRAM reads zero | Find missing static/read-path configuration or bad output corridor |
| `VP-AGM-007` | far-spread registers stay at zero | Compare clock/data delivery by region |
| `VP-AGM-008` | PIN inputs and SPI MISO fail | Recover the complete pad/input chain |
| `VP-AGM-009` | 256-bit state diverges after routing | Minimize while preserving the second-transaction failure |

The goal for each is a general cause/fix, not a route pinned around one failing
example.

## P2: make wider designs route reliably

Current stress cases:

- `regbank16` still produces no image;
- `addsub16` exposes placement problems;
- the 256-bit user design needed many attempts and then failed on hardware;
- its structural equivalent does not route;
- `public32` is still a retained exact route rather than a generic bank.

Work here should focus on placer diagnostics, corridor pressure and negotiated
routing. Then expand fresh AHB state beyond the currently retained footprints.

AHB master/DMA can wait until wide AHB slave designs are routine.

## P3: peripheral coverage

### UART

- UART3/4 TX;
- UART0-4 RX;
- more framing, break and flow control;
- interrupts/DMA;
- alternate pins/packages.

### SPI

Fix MISO first. Then:

- duplex/RX;
- more modes;
- dual/quad;
- DMA/interrupt;
- simultaneous controllers and alternate pins.

### I²C

- more lengths/transaction shapes;
- I²C1 stretching and longer stretches;
- 10-bit addressing;
- arbitration/multimaster;
- simultaneous buses;
- DMA/interrupt and electrical margins.

### Other blocks

Timers, CAN with a transceiver, USB host/OTG, Ethernet with a PHY, external
ADC/DAC/comparator tests, RTC clocking and peripheral-linked DMA all need more
work.

## P4: BRAM, clocks and carry

### BRAM

First fix the read/static-config problem behind `VP-AGM-006`. Then expand sites,
widths, ports, address range, writes, mixed widths, independent clocks, output
registers and collision behavior.

### Clocks

Resolve the far-region state failure before claiming broad clock reach. Then
map regions/seams, gating/reset, alternate PLL outputs, phase/duty/feedback
options and more HSE sources.

### Carry

Expand beyond the currently tested same-tile, X20-corridor and seam cases;
include multiple chains, density and timing.

## P5: routing model

- Measure actual whole-device topology coverage, not only coverage of recovered
  corpus rows.
- Fill missing special-block/IO feeders.
- Improve the 52 no-route campaign designs with general algorithms.
- Add generated/metamorphic workloads after the current correctness failures
  are better understood.
- Build a sealed holdout set once the architecture rules stop changing.

## P6: packages and boards

- Finish L64 bring-up and explain its AHB mismatch.
- Test Q32 and L100 on real boards.
- Treat AG32VH/PSRAM as a separate track.
- Finish the target-side Pico/UART recovery test.
- Keep tool bundles and OpenOCD installs reproducible across operating systems.

## P7: CPU-scale designs

The retained SERV route is useful, but fresh CPU-scale work should wait until
wide placement and BRAM are less fragile.

Then:

1. fresh-route SERV repeatedly without route pins;
2. test register-file writes and BRAM directly;
3. broaden instruction/load/store/branch/CSR/interrupt/exception coverage;
4. add unrelated applications;
5. include CPU-scale cases in the eventual holdout suite.

A roadmap item is done when the implementation and the relevant hardware tests
are in the repository and the status page can state plainly what now works.