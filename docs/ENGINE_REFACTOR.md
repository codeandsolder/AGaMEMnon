# Engine-core refactor

**Completed 2026-08-06.** This page is the design record for that refactor. For
the current engine layout, see [ARCHITECTURE.md](ARCHITECTURE.md).

The short version: the old architecture and bitgen code had grown into two
large files full of feature-specific special cases. The refactor moved those
features into separate modules, made bit ownership explicit, and kept all
retained qualified images byte-identical throughout the migration.

At completion:

- `arch.py` and `bitgen_seq.py` were reduced to compatibility shims;
- architecture generation moved into `archgen.py` plus feature modules;
- bit generation moved into a phase-driven `bitgen.py` plus feature modules;
- each chipdb file got one declared owner;
- features got declared writable bit ranges;
- writes outside those ranges, or conflicting active owners, became build
  errors;
- retained routed qualification artifacts packed byte-identically before and
  after the refactor on Linux, Windows and macOS.

The C++ nextpnr backend (`agrv2k.cc`) was deliberately left out of this work.

## Why it was needed

Before the refactor, `arch.py` was about 2,150 lines and `bitgen_seq.py` about
1,340 lines. Both contained hand-threaded cases for routing, clocks, IO, BRAM,
carry, MCU paths and other features.

That caused three practical problems:

1. Two features could accidentally touch the same configuration bits.
2. Emission order depended on where code happened to sit in a large Python
   function.
3. Adding a newly understood route often meant editing shared engine code
   instead of adding data to the relevant feature.

The existing option/constant registry already solved part of the configuration
problem. This refactor applied the same idea to the engine structure.

## Resulting structure

```text
agamemnon/engine/
  registry.py
  chipdb_schema.py
  bit_ownership.py
  features/
    core_logic.py
    routing.py
    clocks.py
    physical_io.py
    carry.py
    bram.py
    mcu_ahb.py
    mcu_gpio.py
    route_through.py
  archgen.py
  bitgen.py
  arch.py
```

Each feature module owns its data files and contributes some combination of:

- wires, PIPs and BELs to the nextpnr device model;
- bitstream fields for placed/routed objects;
- an emission phase;
- the configuration bits it is allowed to write.

This means many new qualified corridors can now be added as data instead of as
more special-case code in a central function.

## Bit ownership

Every feature declares which physical configuration bits it may write. Bitgen
rejects a build if:

- a feature writes outside its declared area; or
- two active features try to own the same bit.

`AGAMEMNON_OWNERSHIP_TRACE` can emit the last-writer map for inspection, but the
checks are always active.

This turns a class of bugs that previously appeared only on hardware into a
normal build-time error.

## Explicit phases

Bitstream emission now has named phases rather than relying on Python statement
order. Broadly, the flow is:

```text
base/default clearing
-> logic
-> routing
-> clocks
-> IO
-> MCU boundaries
-> BRAM
-> preamble
-> CRC
```

The exact current implementation may grow more phases, but the important part
is that ordering is intentional and inspectable.

## nextpnr entry point

`arch.py` still exists because `nextpnr-generic` executes it with `ctx` and
`Loc` injected as globals. It immediately calls normal importable Python code in
`archgen.py`.

That made architecture generation testable without launching nextpnr for every
unit test.

## Migration rule

The refactor used one simple invariant:

> Every retained routed qualification artifact must pack to exactly the same
> bytes after each migration step.

If the bytes changed, the migration step was treated as a behavior change and
was not accepted as a refactor.

The retained pack-regression records are in
`qualification/pack_regression.json` and
`qualification/pack_reproduction_evidence.jsonl`.

This was deliberately a byte-preserving refactor. It did not try to fix broader
routing or silicon-correctness problems at the same time.

## Out of scope

- the C++ nextpnr backend;
- CLI or environment-variable semantics;
- bitstream-format changes;
- support-policy changes;
- requalification of existing designs.

Those were left for separate work so failures could be attributed to one thing
at a time.