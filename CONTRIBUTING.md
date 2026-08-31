# Contributing to AGaMEMnon

Contributions can be code, documentation, reverse-engineering notes, test
designs, or hardware results.

The most useful hardware reports make it easy for someone else to reproduce the
setup and distinguish “didn't route,” “built but simulated wrong,” “vendor tool
behaved oddly,” and “worked in the model but failed on the chip.”

## Before opening a change

1. Check existing issues and [ROADMAP.md](ROADMAP.md).
2. Keep unrelated refactors separate from behavior changes.
3. Note any third-party data or tools used; see [NOTICE.md](NOTICE.md).
4. For large architecture changes, open an issue first and explain what part of
   the device model is changing and how you plan to test it.

## Development setup

```sh
git clone https://github.com/codeandsolder/AGaMEMnon
cd AGaMEMnon
python -m pip install -e ".[programming]"
python -m pip install pytest
agamemnon doctor --no-hardware
python tools/check_docs.py
python tools/check_path_leaks.py
pytest -q
```

Most tests do not need hardware. End-to-end Verilog builds additionally need
Yosys and the AGRV2K nextpnr backend.

## Pull requests

Please include:

- what changed and why;
- how you tested it;
- commands needed to reproduce the result;
- relevant board/package/clock/tool versions for hardware changes;
- provenance/licensing for newly added recovered data;
- any remaining known limitation.

If a result only applies to one pin, route, clock, width or package, say that.
There is no need to turn every PR description into a qualification report, but
do not silently generalize a one-off hardware result into family-wide support.

## Hardware results

Hardware observations normally go into the append-only JSONL files under
`qualification/` when an existing schema fits. If an old result turns out to be
wrong, add a correcting record rather than rewriting the old one.

Useful fields include:

| Field | Example |
|---|---|
| Device | `AG32VF303CCT6` |
| Fabric/package | `AGRV2KL48`, LQFP-48 |
| Board | vendor dev board / board revision |
| Probe | CMSIS-DAP, USB loader, Pico UART |
| Tools | AGaMEMnon commit, Yosys, nextpnr, compiler, OpenOCD |
| Inputs | source, PCF, routed JSON, firmware and hashes |
| Conditions | clocks, voltage, wiring, reset/boot state |
| Observation | measured pin/register/serial/readback result |
| Recovery | backup/restore result if flash was modified |

For a new fabric path, an FCB-accepted image is a useful first check, but the
interesting result is what the configured circuit actually did.

## Code and documentation

- Support Python 3.8 or newer.
- Prefer SRAM for new hardware experiments; persistent writes should require
  explicit user intent and verification.
- Reject known unsupported cases rather than inventing a plausible encoding.
- Add regression tests when fixing parser, packer, safety or architecture bugs.
- Mark examples that have only been built/simulated and not run on hardware.
- Use package-specific pin names for physical IO results.
- Write docs for engineers, not auditors: state what works, what failed and what
  remains unknown without repeating the same disclaimer in every paragraph.

## Repository hygiene

Do not commit secrets, personal flash backups, identifying serial numbers, or
vendor packages that cannot legally be redistributed. Generated build output
should only be committed when it is an intentional test/qualification fixture
with a clear purpose.

By contributing, you confirm that you have the right to submit the material
under the repository's license and notices.