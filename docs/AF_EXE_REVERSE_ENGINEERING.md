# Reverse-engineering the AG32 fabric back-end

This page is the story of how the open FPGA flow was recovered and which early
ideas turned out to be wrong. For the current feature list, see
[STATUS.md](STATUS.md). For the 105-design comparison campaign, see
[VENDOR_PARITY.md](VENDOR_PARITY.md).

## The black box

The AG32 combines a RISC-V MCU with a small AGRV2K FPGA. The vendor tools use a
closed Windows backend, `af.exe`, for packing, placement, routing and bitstream
generation. Public documentation does not describe the routing fabric or
bitstream in enough detail to reproduce that flow.

AGaMEMnon's target is:

```text
Verilog -> Yosys -> nextpnr -> open bitgen -> SRAM/flash image
```

The vendor backend is still extremely useful as a source of examples and
encodings, but it is not needed by the normal AGaMEMnon build path.

## What was recovered

### Architecture data

The vendor device data contains routing topology, muxes, clocks/PLL information,
and configuration-bit maps in encoded archives. The archive transform was
recovered and checked by decoding and re-encoding it byte-for-byte.

AGaMEMnon converts the useful data into ordinary chip-database files that can be
reviewed and used without loading the vendor executable.

### Bitstream format

The fabric image has a small global preamble, a large configuration body,
compression, and a CRC. AGaMEMnon now generates the preamble, design-neutral
body, supported feature overlays, compression and CRC itself.

The old `fabric_default.bin` is retained as a decode/differential reference, not
as a dependency of normal builds.

### Routing

A large fraction of the regular routing fabric can be described from recovered
selector observations plus a few geometric rules. Where several observations
agree, the same relative rule can be reused. Where they conflict, the release
flow refuses to guess.

One useful reverse-engineering result was negative: some dense crossbar wiring
is generated procedurally from geometry rather than stored as one obvious table
inside the device database. The open architecture reconstructs it the same way.

### Hard blocks and MCU boundaries

Differential builds and board tests recovered pieces of the PLL, BRAM, IO,
External-AHB, local-interrupt and peripheral-to-pad configuration. These areas
are much less complete than ordinary LUT/routing support and are still a major
source of bugs.

## The “dead routing edge” mistake

An early hardware campaign found several large designs where signals stopped
working on routes that looked legal. The initial conclusion was that some
individual routing edges simply did not conduct on silicon, so the router grew a
negative-edge blacklist.

That turned out to be the wrong model.

When the suspect edges were tested in smaller matched designs, the same edges
conducted:

- in vendor-routed images;
- in naturally routed AGaMEMnon images;
- when AGaMEMnon was explicitly forced through the suspect edge;
- and eventually in direct pad-to-pad witnesses.

All 14 edges in the old negative catalogue have positive isolated hardware
tests now, so the production negative table is empty.

What failed originally was the larger composition, not necessarily the named
edge. Congestion, corridor interactions or some other shared configuration
problem had been blamed on whichever edge the experiment happened to be
tracking.

That correction was important, but it did not magically fix wide designs. Large
AHB/state designs still expose placement, routing and hardware-correctness
problems. The full chronological mess is preserved in
[CONDUCTION_REFRAME_STATUS.md](CONDUCTION_REFRAME_STATUS.md).

## The 105-design campaign

Once small routing and selector coverage improved, 105 hand-written designs were
run through a controlled model/vendor/open comparison:

| Outcome | Count |
|---|---:|
| AGaMEMnon + model + usable vendor reference agreed | 25 |
| Vendor reference failed or was unstable | 12 |
| AGaMEMnon did not produce a qualifying route | 52 |
| AGaMEMnon produced an image that failed on hardware | 13 |
| Test harness incomplete | 3 |

This changed the shape of the project. The main problem is no longer “recover
one more selector table.” There are at least three independent failure classes.

### Vendor output is useful but can also be wrong

Ten vendor references failed their expected behavior and two were unstable. In
some cases AGaMEMnon agreed with the independent model while the vendor image did
not.

For reverse engineering, vendor routes remain excellent examples of intended
placement/topology and configuration codewords. They just cannot be the only
functional reference.

### A clean build can still fail on the chip

Thirteen AGaMEMnon images passed the normal build checks and then failed their
hardware tests. Examples include:

- initialized BRAM reads returning zero;
- PIN_10/PIN_12 inputs staying low;
- SPI0/SPI1 MISO staying high;
- state disappearing in a five-region design;
- a 256-bit state design diverging on the second transaction.

Known exact failures are blocked where practical, but there is not yet a general
rule that detects every neighboring bad composition.

### Equivalent RTL can route very differently

Ordinary RTL and explicit primitive-level versions of the same idea often end
up with very different placement/routing results. Some pass while the
structural equivalent does not route at all.

The current wide-design frontier includes `regbank16`, `addsub16` and the
256-bit state tests. These need better placement/routing algorithms, not more
one-off selector data.

## A useful way to think about the remaining work

There are roughly five stages between Verilog and a working board:

1. **Encoding:** do we know which bits configure the resource?
2. **Availability:** does the public tool expose that resource?
3. **Placement/routing:** can nextpnr build this particular design?
4. **Logical behavior:** does the routed model implement the intended logic?
5. **Hardware behavior:** does the physical chip actually do it?

A lot of early confusion came from treating success at one stage as evidence for
the next. The recent campaign gives concrete counterexamples for almost every
such shortcut.

## What works reasonably well now

There are useful tested subsets of:

- LUT and flip-flop logic;
- carry chains;
- general routing;
- External-AHB slave paths;
- local interrupts;
- physical output/OE paths;
- PLL clock points;
- selected BRAM modes;
- DAP/USB programming;
- UART0/1/2 TX;
- SPI0/1 TX;
- I²C0/1 transactions.

The exact boundaries and known failures change as the reverse engineering
moves, so [STATUS.md](STATUS.md) is the better place for the current list.

## What still needs explaining

The most useful next targets are:

- the BRAM failures (`VP-AGM-006`);
- physical input/SPI MISO (`VP-AGM-008`);
- far-region clock/state delivery (`VP-AGM-007`);
- dense-state divergence (`VP-AGM-009`);
- wide AHB/state placement and routing;
- the remaining peripheral modes and other packages.

The main lesson from the reverse engineering so far is pleasantly mundane:
there is no single magic source of truth. The vendor tools, recovered data,
logical simulation and board tests each answer different questions.