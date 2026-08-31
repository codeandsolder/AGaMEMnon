# Engine configuration and experimental switches

AGaMEMnon has accumulated a lot of environment variables while reverse
engineering the chip. They are registered in `agamemnon/engine/registry.py`,
which records the default, type, consumer and maturity of each option. The
generated [claim-policy ledger](CLAIM_POLICY_LEDGER.md) is the exhaustive list;
this page explains the parts a human is likely to care about.

`agamemnon manifest` dumps the registry as JSON. `--scope arch` and
`--scope bitgen` limit it to one side of the engine.

## Maturity labels

- `release`: part of the normal build path;
- `experimental`: useful for a specific investigation, but not enabled normally;
- `archival`: kept so old experiments can be replayed;
- `diagnostic`: changes logging or inspection rather than the emitted design.

There is also a separate evidence field used by the generated policy machinery.
Unless you are changing that machinery, the practical distinction is simpler:
release options are normal; experimental ones are opt-in; archival ones are for
reproduction; diagnostic ones should not change functionality.

## Normal builds

You should not normally set engine flags by hand. Use the CLI/project settings:

- `--freq` or `[fabric].freq` for fabric frequency;
- `AGAMEMNON_SYSCLK` and `AGAMEMNON_HSE` for low-level clock selection;
- the board/project manifest for package, PCF, carry and MCU/fabric choices.

Direct one-off builds default to L48. `AGAMEMNON_DEVICE` is the lower-level
package selector used by the project loader.

The release bit generator rejects selectors and feature combinations it does
not know how to encode safely. Known bad logical graphs and exact images are
also blocked before output is written.

## `research-unsafe`

`agamemnon build --research-unsafe` enables recovered information that is useful
for reverse engineering but too ambiguous for the normal flow. Among other
things it can use:

- the broad enumerated routing graph;
- recovered RMUX/IMUX crossbar data;
- selector majority/context data;
- decoded mesh-template fallbacks;
- predicted selector values.

Every research-unsafe image gets a `.policy.json` sidecar recording which kinds
of evidence were actually used. An unresolved selector still stops the build.

The equivalent low-level settings are:

```text
AGAMEMNON_STRICT_POLICY=research-unsafe
AGAMEMNON_RESEARCH_UNSAFE=1
```

`agamemnon/chipdb/research_knowledge_manifest.json` hashes the public recovered
database. `selector_conflict_atlas.agdb` contains 74,103 conflicted physical
edge keys from 733,862 parsed observed keys. The much larger raw vendor-derived
workbench corpus is not shipped.

## Experimental-strict mode

Some experiments are precise enough to emit but are intentionally not part of
the release path. They use:

```text
AGAMEMNON_STRICT_POLICY=experimental-strict
AGAMEMNON_EXPERIMENTAL_FEATURES=<comma-separated feature IDs>
```

The experiment itself must also be enabled. The resulting image gets a policy
sidecar so it cannot be confused with a normal release build.

Two current examples are BRAM configuration experiments and a small set of
routing-selector experiments.

## BRAM experiments

`AGAMEMNON_BRAM_EXPERIMENTAL_CONFIG` exposes a limited set of recovered BRAM
configuration fields on AGRV2KL48 X13Y1 through X13Y4.

Enable it with:

```text
AGAMEMNON_STRICT_POLICY=experimental-strict
AGAMEMNON_EXPERIMENTAL_FEATURES=AGAMEMNON_BRAM_EXPERIMENTAL_CONFIG
AGAMEMNON_BRAM_EXPERIMENTAL_CONFIG=1
```

The recovered configuration table currently includes 39 rows. Useful hardware
observations include:

- `PORTA_OUTREG` adds one Port-A read clock in the tested X13Y4 x18 case;
- `PORTB_OUTREG` likewise behaves as one extra Port-B read clock in the tested
  dual-port case;
- `PACKEDMODE` changes the observed value sets in write-shaped and dual-port
  tests, but its mechanism is not yet understood;
- `CLKMODE` has shown no visible effect in the tested compositions;
- the old wrapper-visible write result was not actual BRAM storage mutation;
- one fixed-address x18 write path through `TMUX09 -> KMUX03` has four retained,
  hash-bound working profiles.

That last case is available through the qualified BRAM-write path, but it is a
very specific replay. General BRAM writes, arbitrary TMUX/KMUX routing, other
addresses, widths, sites, modes and collision behavior are still open.

The experimental config gate allows only one new non-baseline field/value per
BRAM cell. Unsupported values, other packages, and untested combinations are
rejected rather than guessed.

The scalar `AsyncReset0` input and its measured `IMUX32 -> TileAsyncMUX00`
selector are also represented explicitly. The live BRAM experiments around it
are still research material rather than general routing support.

## Routing-selector experiment

`agamemnon/chipdb/routing_selector_admission.json` contains six reviewed L48
RMUX30 selector rows. These are exact route/codeword experiments and are kept
separate from the normal selector table.

Enable them with:

```text
AGAMEMNON_STRICT_POLICY=experimental-strict
AGAMEMNON_EXPERIMENTAL_FEATURES=AGAMEMNON_ROUTING_SELECTOR_EXPERIMENT
AGAMEMNON_ROUTING_SELECTOR_EXPERIMENT=1
```

Each row identifies one physical route and the configuration bits used for that
route. The experiment can add an encoding for an edge already present in the
architecture graph; it cannot invent topology that is not modeled.

## Boolean environment variables

Historical engine switches use shell-presence semantics:

```text
unset variable  -> false
empty variable  -> false
any non-empty value -> true
```

This means `AGAMEMNON_FOO=0` is **true**. To disable one, remove it from the
environment:

```powershell
Remove-Item Env:AGAMEMNON_FOO
```

or:

```sh
unset AGAMEMNON_FOO
```

This convention is ugly but intentionally preserved because old campaign
scripts depend on it.

## Registry and provenance

`EngineOptions.digest()` gives generated device databases a stable digest of
the relevant registered inputs. Tests reject `AGAMEMNON_*` switches used by the
architecture generator or bit generator if they are missing from the registry.

The registry also contains a small set of important fitted constants: LUT
width, MCU-edge coordinates, clock selectors, output slices, BRAM presentation
values, raw/CRC sizes and the CRC polynomial. The large CSV/JSON/AGDB chip
files remain the actual architecture database; the registry is not a duplicate
of them.

`qualification/claim_policy_dry_run.json` exercises the default policy against
retained artifacts. Regenerate the policy files with:

```sh
python tools/generate_claim_policy_ledger.py --write
```

CI uses the same command with `--check`.

## Entry points

`archgen.py` exposes `build(ctx, Loc, environ=None)`. The nextpnr-facing
`arch.py` is only a shim because nextpnr injects `ctx` and `Loc` as globals.

`bitgen_seq.py` similarly delegates to the importable bit-generator code. Both
architecture generation and bit generation accept explicit environment maps in
tests, so experiments do not need to mutate the process environment.