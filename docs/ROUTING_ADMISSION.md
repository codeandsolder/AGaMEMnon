# Routing admission

For a routing edge, AGaMEMnon needs answers to two different questions:

1. **Do we know the mux codeword?** This determines whether the bitstream can
   actually select that input.
2. **Has this route been seen working at this physical position?** This tells us
   how much direct evidence we have for the route itself.

Those are related, but not the same thing. The default router can use an edge
whose codeword is known exactly even when nobody has tested that exact physical
occurrence yet. It records those edges so they are easy to find and test later.
Ambiguous mux encodings are simply not used.

Whole-design hardware failures are tracked separately in [STATUS.md](STATUS.md).
The August campaign found examples where every routed/software-side check looked
fine but the complete design still failed, so this page is deliberately about
routing data rather than trying to summarize overall device support.

## The three tiers

| Tier | Meaning | What AGaMEMnon does |
|---|---|---|
| 1 — **witnessed** | This exact physical edge has a route/silicon/reviewed witness and its selector encoding is known | Use it normally |
| 2 — **encoding-certain** | The selector encoding is known, but this exact physical occurrence has no direct conduction witness | Use it and list it in `<output>.confidence.json` |
| 3 — **encoding-ambiguous** | Selector observations conflict or the encoding is unknown | Refuse it |

The old 14-entry "dead edge" list is gone. All 14 edges conduct in isolated
positive tests; the original congested design still failed. That is a useful
reminder that a working individual edge and a working large composition are
not the same experiment.

## Where an exact codeword comes from

AGaMEMnon accepts three sources, in this order:

- **`clean-physical`** — an exact conflict-free observation for this physical
  edge in `sel_edge_pairs.agdb`;
- **`unanimous-relative`** — the same tile-relative edge has the same codeword
  everywhere it has been observed;
- **`byte-exact-closed-form`** — a regular index formula reproduces every
  observed codeword in that class with zero mismatches.

Two closed forms currently pass that test:

| Form | Agreement with corpus |
|---|---:|
| intra-tile `OMUX[3z+1] -> IMUX` | 65,902 / 65,902 |
| same-tile / one-east `RMUX <- OMUX` | 37,552 / 37,552 |

A candidate `IMUX <- RMUX` formula gets 126,180 rows right but has 51
mismatches, so it is not admitted.

As of 2026-08-20 the two accepted closed forms add no new routing edges beyond
the observation tables: all 542 distinct intra-tile `OMUX[3z+1] -> IMUX` pairs
and all surviving `RMUX <- OMUX` shapes already have unanimous relative keys.
The formulas remain useful as cross-checks.

## Choosing a routing model

```sh
agamemnon build design.v --uarch                    # default: tiers 1 + 2
agamemnon build design.v --uarch --release-strict   # tier 1 only
agamemnon build design.v --uarch --research-unsafe --require-clean-selectors
                                                    # experimental graph, clean selector encodings
```

The environment-variable equivalent is
`AGAMEMNON_ROUTING_ADMISSION=release-strict|tiered|tiered-tables`.
`tiered-tables` is an A/B mode that enables tier 2 but disables the closed-form
fallbacks.

Use the default for ordinary development. Use `--release-strict` when you want
every routed edge in that build to have a witness at its exact position.
`--require-clean-selectors` lets experimental builds use extra architecture
features without using conflicted/predicted selector encodings.

## Confidence manifest

A normal tiered build writes `<output>.confidence.json`. For each tier-2 edge
actually used, it records:

- source, destination, pip and programming tile;
- where the selector codeword came from;
- how many observations support that codeword;
- which nets used the edge;
- the evidence row that would promote it to tier 1.

Example output:

```text
[confidence] 6 of 412 routed edge(s) are tier-2 encoding-certain,
             touching 3 net(s)
[confidence]   X14Y9_RMUX13.X14Y10_RMUX49  (2 net(s), unanimous-relative)
[confidence]   ...
[confidence] manifest -> blinky.bin.confidence.json
```

This is mostly a convenient work queue for the hardware witnessing rig. The
observation count matters: a relative codeword seen once and one seen 158 times
are both unanimous, but clearly not equally interesting to test next.

## What tiering changes in practice

Measured on AGRV2KL48 on 2026-08-20:

| | release-strict | tiered | research-unsafe |
|---|---:|---:|---:|
| routing pips | 245,630 | **324,971** (+32.3%) | 384,335 |
| wires with no way in | 22,527 | 22,167 | 22,125 |
| bel-input sinks with only one feed | 4,415 | 4,337 | — |
| bel-input sinks with six or more feeds | 6,665 | 8,450 | — |

Tier 2 consists of 54,944 `clean-physical` edges and 24,397
`unanimous-relative` edges. The relative group is backed by a median of 86
physical observations; 507 have only one.

The extra graph coverage has not yet translated into many more successfully
routed examples. Across 42 designs built both ways, 32 routed under both models
and 10 failed under both. Eleven of the 32 successful tiered builds did use
some tier-2 edges (61 of 528 routed pips), but there were no strict-vs-tiered
success conversions in that sample.

The ten shared failures all hit the same MCU-entry problem: the relevant
`BufMUX` has exactly one downhill pip in every graph, including
`research-unsafe`. Those alternatives are missing from the MCU-edge corridor
data itself, so widening the admission policy cannot fix them.

Tiering does improve raw reachability. Of 70 audited wires that were targetable
but unreachable under the strict graph, 6 became reachable through promoted
`corpus_route` evidence, another 32 through tier-2 admission, and 32 remain
unreachable. The remaining 32 are IO-ring wires with no clean selector data.

Fmax moved both up and down in the small paired sample (about -18 to +25 MHz),
with no useful trend.

## Selector-table sanity checks

`agamemnon/engine/selector_injectivity.py` checks selector tables when they are
loaded. It rejects cases where one physical mux gives the same codeword to two
inputs, or a table names a source the device graph says cannot feed that mux.

This is separate from routing admission: admission decides whether the router
may use an edge; the injectivity checks decide whether a selector table is
self-consistent enough to emit.

## Boundary-terminal oddity

The east/south MCU boundary fallback table has fourteen source names but only
twelve distinct `(lo, hi)` words. Two pairs collide:

- `RMUX20` / `RMUX92` -> `(2,6)`
- `RMUX49` / `RMUX25` -> `(0,4)`

The twelve sources with direct witnesses all have distinct words. `RMUX25` and
`RMUX92` are the two extras with no witnessed row anywhere, so their fallback
entries are disabled. The witnessed `RMUX20` and `RMUX49` entries remain.

This may not mean the vendor data is actually contradictory. There is evidence
for an additional selector position (index 8) used for parked/constant-tied
sinks, so the real encoding may have another dimension beyond `(lo, hi)`. Until
that is understood, the unwitnessed fallback guesses stay disabled.

The practical effect today is zero: no observed RRG route into a BBMUXE needs
`RMUX25` or `RMUX92` without an exact tuple. The guard matters only if a future
route tries to use one.
