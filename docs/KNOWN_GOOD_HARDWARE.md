# Known-good hardware

Last updated: 2026-08-24

This page records the board, probes and wiring used for most AGaMEMnon hardware
work. Feature support still lives in [STATUS.md](STATUS.md).

## Reference board

| Field | Value |
|---|---|
| Board | AGM AG32VF303 LQFP-48 development board |
| MCU marking | `AG32VF303CCT6` |
| Fabric target | `AGRV2KL48` |
| Device ID | `0x40200001` |
| RISC-V `misa` | `0x40801125` (`RV32IMAFC`) |
| Main flash | 256 KiB |
| SRAM | 128 KiB |
| HSE used in most tests | 8 MHz |
| Board revision | Not identified on the current fixture |

If your chip marking or package differs, treat it as a different target rather
than assuming the same board profile applies.

## Programming/debug transports

| Transport | Current state |
|---|---|
| AGM CMSIS-DAP + AGaMEMnon OpenOCD | Used successfully for probe, SRAM MCU/fabric loads, full flash backup, sector programming and readback |
| Target USB + uploader 2.1 (`cafe:4001`) | Used successfully for identify/read/erase/write/verify/GO/reset; uploader must be installed first |
| Raspberry Pi Pico 2 UART bridge | Bridge firmware/USB side tested; target UART recovery wiring still needs the documented five-wire addition |

Stock OpenOCD does not have AGM's `riscv -dap` target extension. Install the
paired build with:

```text
agamemnon install-openocd
agamemnon doctor --probe-dap
```

## Pico qualification harness

| AG32 L48 pin | Pico pin | Use |
|---|---|---|
| `PIN_25` | GP12 | output/input/OE tests |
| `PIN_26` | GP13 | output/OE tests |
| `PIN_27` | GP16 | output/OE tests |
| `PIN_28` | GP17 | output/OE tests |
| `PIN_10` | GP4 | top-edge output, UART TX capture, OE control |
| `PIN_11` | GP1 | top-edge output, I²C SDA |
| `PIN_12` | GP0 | top-edge output, input stimulus, SPI SCK |
| `PIN_13` | GP3 | top-edge output, SPI CSN |
| `PIN_14` | GP5 | top-edge output, SPI MOSI |
| `PIN_15` | GP2 | top-edge output, I²C SCL |
| `PIN_16` | GP6 | top-edge output |
| `PIN_17` | GP7 | top-edge output, SPI IO1 |
| `PIN_18` | GP8 | top-edge output / undriven control |
| `PIN_19` | GP9 | top-edge output / registered input |

The full current harness map is in
[HAL_FPGA_REFERENCE.md](HAL_FPGA_REFERENCE.md).

UART0 full-duplex tests on `PIN_30`/`PIN_31` use the board's DAP serial path
rather than this Pico harness. I²C/SPI tests use the checked-in RP2350 slave
oracles under `qualification/`.

The UART mask-ROM recovery wiring is a separate modification documented in
[UART_BOOTLOADER.md](UART_BOOTLOADER.md).

## Tool versions

Pinned tool versions are recorded in
[`tools/bundle/manifest.json`](../tools/bundle/manifest.json).

The AGaMEMnon OpenOCD build is based around patched commit `f96d840a`; exact
sources, patches and hashes are in `tools/openocd/manifest.json`.

Full on-device OpenOCD records exist for Windows and Apple Silicon macOS:

- [`evidence/openocd-windows-ag32.json`](evidence/openocd-windows-ag32.json)
- [`evidence/openocd-macos-ag32.json`](evidence/openocd-macos-ag32.json)

Linux and Intel macOS builds are produced and tested in CI, but do not currently
have separate host-specific hardware runs recorded here.

## Adding another board or setup

Include the chip marking, board revision, probe/transport, wiring, host OS,
tool versions, source/artifact hashes and the observed result. Hardware records
belong under `qualification/` when an existing schema fits.