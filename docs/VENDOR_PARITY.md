# Vendor parity campaign

Broad vendor parity is not here yet. In August 2026 the project ran 105
hand-written test designs against independent models, vendor builds where they
were usable, and real L48 hardware. The campaign was much more useful for
finding bugs than for producing a pretty success percentage.

## Results

| Result | Count | What happened |
|---|---:|---|
| `PARITY_SUCCESS` | 24 | Vendor and AGaMEMnon both matched the test |
| `PARITY_SUCCESS_AFTER_FIX` | 1 | Same, after fixing a defect and rerunning |
| `VENDOR_REFERENCE_FAIL` | 10 | The vendor build was unusable or disagreed with the independent model |
| `VENDOR_UNSTABLE` | 2 | Vendor behavior changed between runs/seeds |
| `ROUTABILITY_GAP` | 52 | AGaMEMnon did not produce a usable routed image |
| `CORRECTNESS_ESCAPE` | 13 | AGaMEMnon built cleanly but failed on hardware |
| `HARNESS_INCOMPLETE` | 3 | The test setup did not produce a valid result |
| **Total** | **105** | Hand-written development tests |

There was no sealed holdout set. Treat these as results for these 105 designs,
not as a measured success rate for arbitrary RTL.

## How the comparison worked

Each test had an expected behavior independent of either FPGA backend. Where it
made sense, the same design was also written in a more explicit structural
form. Vendor builds were rerun with fresh seeds to catch unstable references,
and the AGaMEMnon build used the normal strict routing/configuration checks.
The final comparison was made on hardware from SRAM.

Vendor and AGaMEMnon bitstreams did not need to be byte-identical. Different
placement and routing are fine if the hardware does the same thing.

## What worked

Six of 51 user/structural pairs worked in both forms:

| Surface | Tested case |
|---|---|
| SPI0 TX | Mode 3, MSB-first, active-low CS, 1–4-byte transfers on the tested L48 route |
| SPI1 TX | Same bounded transfer test on the second controller |
| I²C0 | Address `0x55`, write `2A A6`, repeated START, read `5A C3 7E`, ACK/ACK/NACK, STOP; plus one stretch test |
| I²C1 | Same fixed transaction on the second controller |
| UART1 TX | Fixed 64-byte pattern, 8-N-1, 9,600/38,400/115,200 baud on PIN_10 |
| UART2 TX | Fixed 64-byte pattern, 8-N-1, 9,600/38,400/115,200 baud on PIN_10 |

Other working cases included:

- UART0 TX/PIN_10 after fixing one selector codeword (`VP-AGM-002`);
- several small fabric/AHB tests: Boolean logic, short shifts, two-bit
  add/subtract, small LFSRs, and selected MCU-entry/interrupt designs;
- physical output tests on PIN_12 and PIN_16;
- one local PLL/shift test.

Hashes, seeds, routes, and board transcripts are in `qualification/*.jsonl` and
the associated checked-in artifacts.

## What did not route

Fifty-two designs never produced a usable image. This is the largest single
bucket in the campaign. Some structural rewrites failed where ordinary RTL
worked; some families failed in both forms. The current router/placer and
recovered graph still have real breadth problems.

Useful landmarks:

- the old X13Y12 ingress coverage problem has working solutions;
- `regbank16` still does not route through the rest of the flow;
- `addsub16` exposes placement differences around the current density limit;
- a 256-bit user-state design routed only after 12 failed attempts, then failed
  on hardware; its structural version did not route at all.

## What built but failed

Thirteen AGaMEMnon images passed the software-side checks and were still wrong
on hardware:

- MCU feedback, FSM update, rotate/reset, and add/reset designs
  (`VP-AGM-001`, `003`–`005`);
- initialized x1/x18 BRAM reads returning zero (`VP-AGM-006`);
- five widely separated registers returning zero despite correct routed
  simulation (`VP-AGM-007`);
- PIN_10/PIN_12 inputs stuck low and SPI0/SPI1 MISO stuck high
  (`VP-AGM-008`);
- a 256-bit state design that matched the first transaction and diverged on the
  second (`VP-AGM-009`).

The exact 13 bad images are now blocked, and seven recurring bad logical graphs
are blocked before routing. Typed SPI0/SPI1 MISO and the affected BRAM profiles
are also disabled directly. That prevents the known failures from being
re-emitted; the underlying causes of the 13 failures are still being worked on.

## The vendor backend also does strange things

Ten vendor references failed and two were unstable. At least one test had the
independent model and AGaMEMnon agree while the vendor result disagreed. In
another, all three outcomes differed.

This matters because `af.exe` is invaluable for recovering bit encodings and
intended topology, but a vendor-generated image is not automatically the right
answer for hardware behavior. The independent test model is what makes those
cases diagnosable instead of turning the comparison into "whichever tool made
this image must be right."

## What we know at each layer

| Layer | In decent shape | Still missing |
|---|---|---|
| Image format | Container, compression/decompression, CRC, preamble, generated base and overlays for supported features | Meaning of reserved/unnamed fields and unsupported hard-block modes |
| Routing encoding | Large exact and position-relative selector tables with conflict checks | Whole-device coverage and several special feeders |
| Routing on silicon | Many exact routes; all 14 entries in the former "dead edge" list conduct in isolated tests | Congested/wide compositions and arbitrary routes |
| Placement | Retained known-good routes and many small fresh designs | Robust dense/wide placement and consistent user/structural behavior |
| Functional hardware | The working tests listed here and in the qualification ledgers | General RTL, broad BRAM/clock/input/peripheral support, fresh CPU-scale routes, other packages |

`research-unsafe` exposes extra vendor-derived or predicted routing knowledge
for reverse-engineering work. It is useful for experiments, not the normal
build path.

For the current user-facing support list, see [STATUS.md](STATUS.md). For the
things that still need work, see [ROADMAP.md](../ROADMAP.md).
