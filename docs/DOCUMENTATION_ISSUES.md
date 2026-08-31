# Documentation issues

This file tracks contradictions, stale claims and documentation that still needs
cleanup. It is intentionally a human-maintained list; generated ledgers stay
generated.

## Resolved on this branch

### Non-L48 strict builds

The old prose disagreed about whether non-L48 devices could emit strict images.
The implementation in `agamemnon/engine/claim_policy.py` answers this clearly:

- pad-free/fabric-logic-only builds are allowed on Q32/L64/L100;
- physical/electrical surfaces are restricted to L48 in strict mode;
- `research-unsafe` has its separate rules.

`ARCHITECTURE.md` and `AG32_OVERVIEW.md` now say this. The generated
`FAMILY_COVERAGE_MATRIX.md` was already consistent with the implementation.

**Still stale:** `USAGE.md` contains an older sentence saying strict image
emission rejects all non-L48 packages. That paragraph should be corrected when
`USAGE.md` gets its larger cleanup.

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

The overview now uses one consistent statement: 45 accepted fabric
`(SYSCLK,HSE)` pairs, 43 measured on the reference board with HSE=8, plus two
byte-exact 12/16 MHz HSE profiles not run on that board.

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

### MCU clock terminology

`MCU_CLOCKS.md` contains several generations of clock conclusions and is useful
as an investigation record, but it is hard to tell at a glance which numbers
are current. In particular:

- MTIME measured 14.08 MHz in one SRAM-loaded setup;
- UART0's measured reference was around 14.47 MHz;
- SPI0's absolute reference remains unresolved;
- the external-AHB bus clock has a measured 1:1 ratio to MTIME, not a settled
  absolute frequency.

A short “current answer” table at the top would make the long historical detail
much easier to use.

## Remaining prose cleanup

### `USAGE.md`

This is now one of the largest reader-facing holdouts. It mixes command
reference, hardware campaign results, checkpoint archaeology and repeated
support caveats.

Suggested split:

- keep common CLI syntax/options in `USAGE.md`;
- move exact retained checkpoint recipes to a dedicated replay/qualification
  reference;
- keep hardware evidence in `STATUS.md` / hardware pages;
- fix the stale non-L48 package statement noted above.

The opening `IMPORTANT` box is also redundant with the status page.

### `MCU_CLOCKS.md`

Keep the measurements and history, but add a short current-state table first and
move retracted/obsolete interpretations under a historical section.

### `FABRIC_DEFAULT_CANVAS.md`

This is valuable reverse-engineering archaeology, but the top has several
TL;DRs and repeated “what this does not prove” paragraphs. A cleaner structure
would be:

1. what `fabric_default.bin` is today;
2. file/layout facts;
3. what is decoded;
4. what is still unknown;
5. historical corrections.

Do not delete the measurement tables; they are the useful part.

### `CONDUCTION_REFRAME_STATUS.md`

Keep this mostly as a lab log. It records wrong turns and reversals that should
not be sanitized away. The main improvement would be a short front-page index:
current conclusion, important reversals, then the chronological log.

### `HARDWARE_VALIDATION.md`

Likewise, prefer a concise index/current summary at the top rather than deleting
the detailed bench records. The raw negative results are useful and should stay.

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