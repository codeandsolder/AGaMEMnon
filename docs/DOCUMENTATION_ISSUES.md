# Documentation issues

This file tracks contradictions, stale claims and documentation that still needs
cleanup. It is intentionally human-maintained; generated ledgers stay generated.

## Resolved on this branch

### Non-L48 strict builds

The old prose disagreed about whether non-L48 devices could emit strict images.
The implementation in `agamemnon/engine/claim_policy.py` answers this clearly:

- pad-free/fabric-logic-only builds are allowed on Q32/L64/L100;
- physical/electrical surfaces are restricted to L48 in strict mode;
- `research-unsafe` has its separate rules.

`ARCHITECTURE.md`, `AG32_OVERVIEW.md` and `USAGE.md` now say this. The generated
`FAMILY_COVERAGE_MATRIX.md` was already consistent with the implementation.

### Absolute “never emits a bad bitstream” promise

The old README claimed AGaMEMnon would never emit a bitstream that failed on
real silicon. The hardware campaign contains 13 counterexamples.

Removed. The docs now say the useful thing: known failures are blocked where
possible, but new configurations can still be wrong.

### Fork quick-start URLs

Several fork docs cloned `bbenchoff/AGaMEMnon`. The reader-facing README,
installation and contributing instructions now clone
`codeandsolder/AGaMEMnon`.

### UART ROM / Pico wording

The docs now distinguish the two facts:

- the chip has a flash-independent UART mask-ROM recovery mechanism;
- the current Pico-to-L48 five-wire target setup still needs its final hardware
  run.

Avoid shortening both into “UART recovery qualified.”

### Clock-profile count

The overview/usage docs now use one consistent statement: 45 accepted fabric
`(SYSCLK,HSE)` pairs, 43 measured on the reference board with HSE=8, plus two
byte-exact 12/16 MHz HSE profiles not run on that board.

### MCU clock summary

`MCU_CLOCKS.md` now starts with the current answer instead of making readers
reconstruct it from several generations of experiments:

- MTIME: 14.08 MHz in one SRAM-loaded L48 setup;
- UART0 reference: about 14.47 MHz;
- SPI0 absolute reference: unresolved;
- External-AHB bus clock: measured 1:1 with MTIME, absolute rate unresolved.

The invalid old ~258 MHz SPI interpretation is kept as a short historical
correction rather than competing with the current answer.

### `USAGE.md`

The command reference no longer contains pages of exact AHB/BRAM/IO campaign
archaeology. It keeps the actual commands/options and links detailed hardware
results to the pages that own them.

### Vendor canvas overview

`FABRIC_DEFAULT_CANVAS.md` no longer has several competing TL;DRs. It now starts
with what the file is used for today, then keeps the important format numbers,
reconstruction stages, correction of the old LUT/`0xFF` theory, and FCB CRC
experiment.

## Claims that still need checking

### `af.exe` and “no conduction model”

Old prose stated as implementation fact that `af.exe` has no conduction model.
The observed fact is weaker and sufficient: vendor-generated routes can fail on
hardware and the vendor output does not reliably predict every physical
composition.

The README and reverse-engineering narrative now use observed behavior instead.
Search older research logs before treating every remaining “conduction-blind”
phrase as established binary-analysis fact. If decompilation really proves the
stronger statement, link that evidence where the claim is made.

### Old “dead edge” terminology

All 14 entries in the old per-edge negative catalogue later conducted in
isolated tests. Historical logs can keep the old terminology when describing
what was believed at the time, but current summaries should call those failures
composition/congestion-context failures rather than intrinsically dead wires.

`CONDUCTION_REFRAME_STATUS.md` is deliberately a chronological record, so do
not rewrite old entries to pretend the project knew the answer earlier.

## Remaining prose cleanup

### `CONDUCTION_REFRAME_STATUS.md`

Keep this mostly as a lab log. It records wrong turns and reversals that should
not be sanitized away. It already has a wind-down summary; future cleanup should
mainly stop new current-state material from accumulating above the chronological
log.

### `HARDWARE_VALIDATION.md`

Prefer a concise index/current summary at the top rather than deleting detailed
bench records. The raw negative results are useful and should stay.

### Large subsystem research pages

Several AHB/BRAM/IO research pages are intentionally detailed. When they become
reader-facing references, prefer adding a small “current answer” table and
moving old hypotheses below it rather than deleting useful archaeology.

### `CLAIM_POLICY_LEDGER.md`

Generated file. Do not edit it for style. Human-readable explanations belong in
`ENGINE_CONFIGURATION.md` and `STATUS.md`.

### `FAMILY_COVERAGE_MATRIX.md`

Also generated. Its prose is still fairly bureaucratic, but the fix belongs in
`tools/generate_family_coverage_matrix.py` / its source JSON rather than a hand
edit to the generated Markdown.

## Style rule going forward

For reader-facing docs, prefer:

- **works** — with the useful scope when needed;
- **broken** — with the defect or observed behavior;
- **unknown/untested** — when we genuinely do not know.

Use evidence-tier/policy vocabulary when it explains an actual tool behavior,
not as a ritual disclaimer around every engineering statement.