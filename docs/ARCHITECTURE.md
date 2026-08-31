# Architecture

AGaMEMnon turns Verilog into an AG32 fabric image with an open synthesis,
place/route and bit-generation flow:

```text
Verilog
  -> Yosys technology mapping
  -> generated AGRV2K device database
  -> nextpnr `agrv2k` backend
  -> routed JSON
  -> AGaMEMnon bitgen
  -> uncompressed SRAM + compressed flash images
```

The implementation is data-driven where possible: recovered routing, mux,
package and configuration information lives under `agamemnon/chipdb/`, while
Python feature modules turn that data into a nextpnr graph and a bitstream.

For the current list of things that actually work on hardware, see
[STATUS.md](STATUS.md). This page is about how the tool is put together.

## Target device

The AGRV2K fabric contains roughly:

- 2,112 LUT4s;
- 2,112 flip-flops;
- four 9-Kbit BRAMs;
- one PLL and several global clocks;
- MCU/fabric interfaces;
- programmable IO around the hard MCU/peripheral subsystem.

AGaMEMnon knows four package-level fabric targets:

- `AGRV2KL100`
- `AGRV2KL64`
- `AGRV2KL48`
- `AGRV2KQ32`

Bond maps exist for all four. L48 is the main hardware-tested target.

A **pad-free, fabric-logic-only** strict build is allowed for the other packages
because the AGRV2K fabric is shared. Physical/electrical features such as pad
input/output/OE are still restricted to L48 in strict mode until those packages
get their own hardware work. This distinction is enforced in
`agamemnon/engine/claim_policy.py`.

## Synthesis

`agamemnon/synth/` contains the Yosys scripts, primitive definitions and
technology maps. Ordinary Verilog is lowered to AGRV2K primitives for:

- LUTs and flip-flops;
- clocks;
- physical IO;
- MCU boundary cells;
- carry chains;
- BRAM.

Some resources are deliberately narrower than their theoretical hardware
capability. Carry and BRAM, for example, have several recovered modes but only a
smaller subset is enabled by the normal flow.

## Chip database

`agamemnon/chipdb/` contains the recovered architecture data. Major classes
include:

- about 50k named routing nodes;
- BELs for logic, IO, BRAM, clocks and MCU boundary blocks;
- ordinary and special routing edges;
- selector/codeword tables;
- configuration-bit locations;
- package bond maps;
- carry/BRAM/IO corridor data;
- timing estimates;
- the data used to generate the design-neutral base image.

The old `fabric_default.bin` is still retained as a decode/differential
reference, but normal builds generate the base image from public tables.

Large derived tables are ordinary Git files; Git LFS is not required.

## Device-database generation

`agamemnon/engine/archgen.py` builds the active architecture graph from the
chip database. Feature modules under `agamemnon/engine/features/` contribute
wires, PIPs and BELs.

`emit_uarch_db.py` can flatten that graph into CSV files consumed by the C++
nextpnr backend:

| File | Contents |
|---|---|
| `dev_meta.csv` | device metadata |
| `dev_wires.csv` | wires and coordinates |
| `dev_bels.csv` | BELs and locations |
| `dev_belpins.csv` | BEL pins |
| `dev_pips.csv` | routing edges and delays |

The generated graph is filtered according to the selected package and enabled
features. Ambiguous routing selectors are normally removed before nextpnr sees
them.

## Routing

Most configurable routing edges select one input of an RMUX/IMUX structure by
programming a small selector codeword.

The normal routing database prefers:

1. exact physical observations;
2. relative selector patterns that agree everywhere they have been observed;
3. a few recovered closed-form rules for regular routing classes.

Conflicting selector data is not silently averaged into a release encoding.

The default `agamemnon build --uarch` can also use **tier-2** edges: cases where
the selector codeword is known but that exact physical edge has not been
individually witnessed conducting. Their use is recorded in
`<output>.confidence.json`.

`--release-strict` restricts routing to the tighter witnessed set.

For the details and current routing counts, see
[ROUTING_ADMISSION.md](ROUTING_ADMISSION.md).

### Research mode

`--research-unsafe` exposes a much broader recovered graph, including ambiguous
or predicted routing information useful for reverse engineering. It writes a
policy/provenance sidecar showing what the image used.

It is intentionally separate from the normal build path.

## Special routing cases

Not everything in vendor route data is a free wire.

For example:

```text
IMUX -> alta_slice -> OMUX
```

passes through a real LUT. AGaMEMnon must instantiate/configure that LUT instead
of treating the path as a transparent PIP. Failing to model this correctly was
the cause of an early stuck-data bug on the MCU AHB read path.

Likewise, some BRAM and route-through paths need complete multi-field
configuration footprints rather than just one obvious mux selector. These are
kept as explicit feature data instead of generalized from geometry too early.

## Placement and routing backend

The C++ nextpnr backend is under:

```text
agamemnon/engine/uarch/agrv2k/
```

It handles:

- LUT/FF packing;
- constants and clocks;
- regional placement;
- fanout splitting/retry;
- MCU entry/exit corridors;
- physical IO endpoints;
- BRAM packing;
- carry placement;
- timing estimates;
- exact checkpoint replay where requested.

This is still one of the main incomplete areas. The August 2026 campaign had 52
designs that did not produce a route, and equivalent RTL/primitive forms can
have very different routability.

## Bit generation

`agamemnon/engine/bitgen.py` converts routed JSON into the raw fabric image.
Broadly it:

1. creates the design-neutral base frame;
2. clears design-dependent fields;
3. applies logic and routing;
4. applies clocks, IO, carry, MCU and BRAM configuration;
5. regenerates the 164-byte preamble;
6. computes the final CRC.

`agamemnon/engine/to_bin.py` adds the eight-byte device header for the
99,944-byte uncompressed SRAM image. `lzw_codec.py` creates the compressed flash
form.

See [BITSTREAM_FORMAT.md](BITSTREAM_FORMAT.md).

## Feature modules and bit ownership

The Python engine is split into feature modules such as:

- `core_logic.py`
- `routing.py`
- `clocks.py`
- `physical_io.py`
- `carry.py`
- `bram.py`
- `mcu_ahb.py`
- `mcu_gpio.py`
- `route_through.py`

Each feature owns its chip-database files and the configuration-bit regions it
is allowed to write.

Bitgen rejects writes outside those declared regions and conflicting active
owners. `AGAMEMNON_OWNERSHIP_TRACE` can additionally dump the final bit-owner
map for debugging.

The refactor that introduced this structure is documented in
[ENGINE_REFACTOR.md](ENGINE_REFACTOR.md).

## Known-bad image guards

Several hardware failures are retained as exact negative examples. Before
writing output, bitgen checks:

- canonical logical-graph fingerprints for several known failures;
- exact uncompressed image hashes for retained bad images.

Matches are refused. These guards prevent known regressions from being emitted
again; they are not a general classifier for every possible bad design.

## Verification

`verify_netlist.py` evaluates placed LUTs, flip-flops, carry chains and MCU read
paths from the routed design.

`agamemnon.sim.ahb` provides a cycle-level model for the External-AHB slave
interface, with a matching synthesizable Verilog model.

These tools are useful for catching synthesis/routing mistakes before touching
hardware. They do not model every physical effect. Several campaign failures
simulated correctly and still failed on the chip, especially BRAM, physical
input, far-region state and dense-state cases.

## Programming

Programming is a separate layer from architecture generation. AGaMEMnon
supports:

- CMSIS-DAP/SWD through its patched OpenOCD;
- a flash-resident USB CDC uploader;
- a Pico-assisted mask-ROM UART path for recovery work.

New fabric work is normally loaded from SRAM first. See
[PROGRAMMING.md](PROGRAMMING.md).

## Where the remaining difficulty is

The bitstream container and much of the basic routing encoding are now fairly
well understood. The harder remaining problems are:

- robust placement/routing for larger designs;
- BRAM configuration and behavior;
- physical input paths;
- clock distribution across the fabric;
- dense/state-heavy compositions;
- broader hard-peripheral routing;
- package-specific hardware testing outside L48.

In other words, the project has moved from “what do these bytes mean?” toward
“can we reliably turn arbitrary-ish RTL into a physical configuration that does
what the routed model says?”