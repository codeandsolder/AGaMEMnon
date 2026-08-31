# Documentation issues

This file tracks places where the prose disagrees with itself, overstates what
we know, or makes a simple point much harder to read than it needs to be.

`STATUS.md` is still the current support list. This file is the cleanup queue.

## Contradictions / needs one answer

### Non-L48 builds

`AG32_OVERVIEW.md` says pad-free configuration-accept builds are supported for
all four package targets, while `ARCHITECTURE.md` says strict image emission
refuses every non-L48 target.

These cannot both describe the current CLI. Check the code and make both pages
say the same thing. Keep separate answers for:

- generating a device database;
- routing a pad-free design;
- emitting an image;
- routing package pins;
- hardware qualification.

### UART mask-ROM / Pico recovery status

`PROGRAMMING.md` calls the UART mask-ROM/Pico path recovery-capable, but later
says the target-side five-wire setup has not completed hardware qualification.
The README also calls it the flash-independent recovery route.

The likely distinction is simple: the ROM protocol exists and is independent
of main flash, while the current Pico-to-L48 bench setup still needs its final
target-side run. Say that directly everywhere and avoid using "qualified" for
two different layers of the stack.

### "AGaMEMnon will never emit a bad bitstream"

The old README said:

> The output of this project will never emit a bitstream that will fail on real silicon.

`STATUS.md` records 13 clean AGaMEMnon images that failed their hardware tests.
Known bad images/compositions are now blocked, but that is not the same as a
guarantee for arbitrary new designs.

Fixed in the documentation cleanup branch by removing the absolute claim.

## Misleading or stronger than the evidence shown

### `af.exe` "has no conduction model"

Several pages state as fact that `af.exe` is conduction-blind or has no model of
which wires conduct. The observed fact is that the vendor backend can produce
routes/designs that fail on hardware, and the reverse engineering has found
cases where its intended topology is not enough to predict working silicon.

If the binary analysis actually proves there is no conduction model, point to
that evidence. Otherwise phrase this as observed behavior rather than an
implementation fact about a closed binary.

### "Electrically dead edges"

Older prose talks about dead routing edges. The newer conduction work says all
14 entries in the former negative catalogue conduct in isolated witnesses and
that the original failures were composition/congestion dependent. Old pages
should stop using those 14 as examples of intrinsically dead wires.

### Fork README clones upstream

The fork's quick-start command currently clones `bbenchoff/AGaMEMnon` instead of
`codeandsolder/AGaMEMnon`. If the fork is meant to be directly usable, the
command should point at the fork. If this is intentionally an upstream-only
README, say why.

### Clock-profile counts are hard to reconcile

`AG32_OVERVIEW.md` describes 45 accepted `(SYSCLK,HSE)` pairs as seven
byte-exact vendor-oracle profiles plus 38 additional HSE=8 measurements.
`STATUS.md` describes 43 measured HSE=8 rates plus two byte-exact profiles that
need unavailable 12/16 MHz HSEs.

Those may be the same 45 points with different grouping, but a reader should
not have to reverse-engineer the arithmetic. Use one table or one shared source
for the count.

## Prose cleanup queue

These are not necessarily wrong; they are just written like internal policy or
qualification paperwork instead of documentation for someone trying to use or
understand the project.

### `STATUS.md`

The support matrix is useful, but the introduction and release-health section
repeat the same warning in several forms: a passing build is not proof, exact
results do not generalize, known failures are blocked, and root causes are
still open. Say each once. Keep the defect table and the actual numbers.

Phrases worth removing or translating include:

- "authoritative public support boundary";
- "silicon-qualified exact subset" when "tested on this exact setup" works;
- "correctness escape" outside the defect table;
- "containment is the release-safety gate and is met";
- repeated "does not qualify ..." tails on every paragraph when the table can
  simply list what works and what does not.

### `VENDOR_PARITY.md`

The campaign results are interesting. The repeated discussion of population
claims, exact claim syntax, evidence layers, and what a reader is allowed to
infer makes the page feel adversarial.

Keep the 105-design result table, what was tested, surprising vendor failures,
and the failure families. One plain sentence saying the sample was hand-written
and had no holdout set is enough.

### `AG32_OVERVIEW.md`

The overview should be the friendly entry point. Its opening currently starts
with qualification-policy language before explaining why the chip is
interesting. Replace that with a short state-of-the-project paragraph and link
to `STATUS.md` for details.

The long External-AHB paragraph also compresses too many unrelated results into
one block. Summarize the working examples and link the detailed AHB page.

### `ROUTING_ADMISSION.md`

The two useful questions are:

1. is there evidence that the route physically works?
2. do we know the mux codeword?

Everything after that should explain the three routing tiers and the measured
coverage. The current opening adds a third whole-design disclaimer, then later
repeats it in several forms. The "loud, local, and diagnosable" description of
conduction failures is also too absolute given the later composition-level
failures.

### `ROADMAP.md`

A roadmap can say what is broken and what to do next without restating the
release process in every priority. Keep the defect-specific experiments and
engineering tasks; shorten procedural phrases such as "required closure",
"release boundary reproducible and fail-closed", and the long promotion rule.

### `CLAIM_POLICY_LEDGER.md`

This file is generated. Do not hand-edit it into friendly prose. Treat it as a
machine-readable/generated reference and keep human documentation elsewhere.
