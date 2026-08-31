# Command-line usage

Install the package and check the environment:

```bash
pip install -e .
agamemnon --help
agamemnon doctor --no-hardware
```

`python -m agamemnon.cli` is equivalent to `agamemnon`.

Building FPGA images additionally needs Yosys and the AGRV2K nextpnr backend.
SWD/DAP programming needs AGaMEMnon's OpenOCD build:

```text
agamemnon install-openocd
agamemnon doctor --probe-dap
```

See [INSTALLATION.md](INSTALLATION.md) for setup and [PROJECTS.md](PROJECTS.md)
for project manifests/templates.

## Start a project

```text
agamemnon new hello --board ag32vf303-l48 --template mcu-blink
cd hello
agamemnon doctor
agamemnon build
agamemnon run --transport dap
```

`mcu-blink` does not use the FPGA fabric, so it is a good first check of the MCU
toolchain and programming path.

## `doctor`

Useful forms:

```text
agamemnon doctor --no-hardware
agamemnon doctor --json --no-hardware
agamemnon doctor --probe-dap
```

`doctor` checks the installed tools and reports the capabilities that are
available. UART target probing is only attempted with an explicit
`--uart-port`, because entering the mask ROM resets the target.

## Qualification report

```text
agamemnon qualify --artifact build/design.bin --output qualification-report.json
```

This is read-only. It records the host/tool report, support-matrix version and
SHA-256 hashes for supplied artifacts. User-home paths are redacted so the file
is easier to share.

See [QUALIFICATION_REPORT.md](QUALIFICATION_REPORT.md).

## Tool configuration

Most users should prefer project/CLI settings. These environment variables are
mainly useful for development and custom installations:

| Variable | Meaning |
|---|---|
| `AGAMEMNON_OSS` | OSS CAD Suite root |
| `AGAMEMNON_UARCH_NEXTPNR` | AGRV2K-enabled `nextpnr-generic` executable |
| `AGAMEMNON_UARCH_NEXTPNR_RUNTIME` | optional runtime DLL directory on Windows |
| `AGAMEMNON_OPENOCD` | explicit OpenOCD executable override |
| `AGAMEMNON_OOCD_CFG` | alternate OpenOCD target config |
| `AGAMEMNON_OOCD_SCRIPTS` | OpenOCD script directory |
| `AGAMEMNON_DEVICE` | package/fabric target; default `AGRV2KL48` |
| `AGAMEMNON_DATA` | alternate chip-database directory |
| `AGAMEMNON_ENGINE` | alternate engine directory |
| `AGAMEMNON_SYSCLK` | fabric frequency when `--freq` is omitted |
| `AGAMEMNON_HSE` | HSE frequency in MHz |

Yosys and nextpnr are launched in separate environments so OSS CAD Suite's DLLs
cannot accidentally shadow a separately built nextpnr runtime.

For low-level experimental engine switches, see
[ENGINE_CONFIGURATION.md](ENGINE_CONFIGURATION.md).

## Build a fabric image

```bash
agamemnon build design.v --uarch -o design.bin
```

A successful build normally writes:

```text
design.bin       99,944-byte uncompressed SRAM image
design.bin.comp  compressed flash image
```

Common options:

| Option | Effect |
|---|---|
| `--pcf FILE` | apply `set_io <port> PIN_<n>` package constraints |
| `--mcu` | expose the MCU/fabric bridge for supported designs |
| `--leds` | expose the maintained LED outputs |
| `--release-strict` | route only through the tighter witnessed-edge set |
| `--research-unsafe` | enable broader recovered/ambiguous routing data and write a policy sidecar |
| `--require-clean-selectors` | with an experimental policy, keep only conflict-free selector encodings |
| `--hard-carry` | use supported dedicated carry placement (compatibility spelling) |
| `--no-hard-carry` | implement arithmetic with LUTs instead of dedicated carry |
| `--cap N` | placement density hint; default 5 |
| `--maxfo N` | fanout threshold used by split-net retry |
| `--compact-maxd N` | experimental regional placement-radius limit |
| `--freq MHz` | set fabric PLL profile and timing target |
| `--verify` | run routed-netlist verification after build |
| `--verify-cycles N` | cycle count for `--verify` |
| `--write-routed FILE` | save the placed/routed JSON |
| `--pin BEL` | pin a generic slice, e.g. `X10Y4_SLICE0` |
| `--baseline FILE` | use an alternate design-neutral canvas/reference |

The default routing model and `--release-strict` are explained in
[ROUTING_ADMISSION.md](ROUTING_ADMISSION.md). `--research-unsafe` is intended
for reverse engineering and hardware probes, not ordinary projects.

If a build fails because of an unsupported selector or feature, bitgen removes
the requested output rather than leaving a stale image behind.

## Exact replay options

A few retained hardware-test designs are available as exact replays:

| Option | Use |
|---|---|
| `--qualified-checkpoint PROFILE` | replay a retained source/route/image profile after checking its hashes |
| `--qualified-bram-write PROFILE` | select one of the retained X13Y4 BRAM source profiles |
| project `qualified_profile = "..."` | select an exact project-level replay such as the reviewed public32 map |

These are useful regression/integration fixtures. They are intentionally strict:
changing the source logic or physical route stops being the same replay.

The `mcu-fpga` project currently uses the retained four-word L48 public32 map:

```text
+0x0  ID32       0x4147414d
+0x4  scratch16
+0x8  counter3
+0xc  W1C1
```

Fresh generic AHB register-bank routing is still a work in progress. See
[MCU_AHB_REGISTER_BANK.md](MCU_AHB_REGISTER_BANK.md) and
[PROJECTS.md](PROJECTS.md) for the retained profiles.

The BRAM profiles affected by `VP-AGM-006` are refused rather than replayed as
working examples. Current BRAM state is in [STATUS.md](STATUS.md).

## Physical IO

Example L48 build:

```bash
agamemnon build examples/designs/comb.v --uarch \
  --pcf examples/constraints/comb_proven_L48.pcf -o comb.bin
```

Package targets are:

```text
AGRV2KL100
AGRV2KL64
AGRV2KL48
AGRV2KQ32
```

`PIN_n` uses decimal physical package lead numbers.

Pad-free/fabric-logic-only strict builds are allowed on the other package
targets. Physical/electrical IO in strict mode is currently L48-only; the other
bond maps are recovered but have not had equivalent board qualification.

L48 has several working output/input/OE routes, but input routing in particular
is not generally solved: newer PIN_10/PIN_12 and SPI MISO compositions exposed
`VP-AGM-008`.

See [MCU_PIN_ROUTING.md](MCU_PIN_ROUTING.md) for the useful current pin map and
[STATUS.md](STATUS.md) for known failures.

## Carry

Dedicated carry is used automatically where the current allocator has a
supported physical footprint. Use:

```bash
agamemnon build arithmetic.v --uarch --no-hard-carry -o arithmetic.bin
```

when you want the all-LUT implementation for comparison.

Current carry coverage is described in [STATUS.md](STATUS.md); it is not yet a
whole-chip carry placer.

## Fabric frequency and timing

```bash
agamemnon build design.v --uarch --freq 25 -o design.bin
```

`--freq` controls both the emitted fabric clock profile and nextpnr timing
target. It overrides `AGAMEMNON_SYSCLK`.

The reference-board table currently has 43 measured HSE=8 points from 4 to
248 MHz plus two byte-exact profiles using 12/16 MHz HSE inputs. Timing outside
the characterized local subset remains conservative.

These are fabric clocks, not MCU core frequencies. See [MCU_CLOCKS.md](MCU_CLOCKS.md).

## Verify a routed netlist

```bash
agamemnon verify design_routed.json --cycles 64
agamemnon verify design_routed.json --observed 0,1,2,3
```

The verifier evaluates routed LUT/flip-flop connectivity, carry chains and MCU
read-lane binding. `--observed` compares supplied hardware values with the
reachable simulated values.

It is a logical/routed-netlist checker; physical-path bugs can still require a
board test.

## Inspect and compare images

```bash
agamemnon explain design.bin
agamemnon explain design.bin --json -o design.json
agamemnon diff before.bin after.bin
```

`explain` reports container type, hashes, CRC, recognized preamble profile and
named configuration features. `diff` separates named feature changes from raw
unmapped-byte changes.

## Bitstream conversion commands

```bash
# Routed JSON -> images
agamemnon pack design_routed.json design.bin

# Image -> raw configuration
agamemnon unpack design.bin.comp -o raw.img
agamemnon decode design.bin.comp -o raw.img

# Raw configuration -> compressed image
agamemnon encode raw.img -o design.bin.comp

# Lossless text representation
agamemnon to-agasc fabric.bin -o fabric.agasc
agamemnon from-agasc fabric.agasc -o rebuilt.bin
agamemnon from-agasc fabric.agasc --uncompressed -o rebuilt-raw.bin

# Edit one placed LUT without rerouting
agamemnon edit-lut fabric.bin --le 20,12,1 --init 0x96e9 -o edited.bin
```

`.agasc` keeps known asserted features by name and unknown asserted bits as raw
records. See [BITSTREAM_FORMAT.md](BITSTREAM_FORMAT.md).

Packing a routed netlist produced with the research graph requires the matching
research policy:

```bash
agamemnon pack --research-unsafe design_routed.json design.bin
```

## Hardware transports

The common commands use `--transport` consistently:

| Transport | Typical use | Recovery if main flash is bad? |
|---|---|---|
| `dap` | development, SRAM loading, flash backup/program | Yes |
| `usb` | convenient installed uploader | No |
| `uart` | mask-ROM recovery path | Yes in principle; L48 Pico target wiring is still being finished |

`dap` is the default.

Examples:

```text
agamemnon probe --transport dap
agamemnon probe --transport usb --port COM7
agamemnon probe --transport uart --port COM6
```

## SRAM execution

```bash
agamemnon sram firmware.bin --fabric design.bin --words 10
```

The default SRAM layout is:

```text
0x20000000  firmware
0x20001000  result/mailbox words
0x20002000  fabric image
```

This is the normal development path because it leaves flash unchanged.

## Flash backup and programming

```bash
agamemnon backup full-flash.bin
agamemnon flash design.bin.comp --addr 0x80008100 --backup full-flash.bin
```

`flash` works in complete 4-KiB erase sectors and verifies the written bytes by
readback. Keep the decompressor and existing boot layout intact when replacing a
compressed fabric image.

USB examples:

```text
agamemnon backup full.bin --transport usb
agamemnon flash app.bin --addr 0x80010000 --backup full.bin --transport usb
agamemnon go 0x80010000 --transport usb
```

Persistent writes require a full backup. See [PROGRAMMING.md](PROGRAMMING.md)
before changing flash.

## Image planning

```bash
agamemnon image --fabric design.bin --mcu firmware.bin
agamemnon image --fabric design.bin --mcu firmware.bin \
  --plan-json build/boot-plan.json
agamemnon image --fabric design.bin --mcu firmware.bin \
  --flash --backup full-flash.bin
```

Without `--flash`, `image` only plans the layout. `--plan-json` records the
artifact hashes, flash ranges and erase geometry without touching hardware.

`--write-options` is an explicit experimental option-byte operation and requires
a separate 128-byte option backup in addition to the normal full-flash backup.

## Mask-ROM UART commands

With the Pico bridge and documented wiring:

```bash
agamemnon uart-probe --port COM6
agamemnon uart-backup full-flash.bin --port COM6
agamemnon uart-flash firmware.bin --addr 0x80000000 \
  --backup pre-write.bin --port COM6
agamemnon uart-reset --port COM6
```

`uart-flash` preserves untouched bytes in affected sectors and verifies the
complete readback before resetting into flash.

Native USB DFU is not implemented. See [UART_BOOTLOADER.md](UART_BOOTLOADER.md).

## Recover a wedged AHB experiment

A broken External-AHB design can hold the CPU bus badly enough that the normal
OpenOCD halt sequence stops working while the debug module remains reachable.
Use:

```bash
python tools/recover_wedged_ag32.py \
  --openocd "$AGAMEMNON_OPENOCD" \
  --scripts "$AGAMEMNON_OOCD_SCRIPTS"
```

The helper asserts the debug module's reset, halts the recovered core, checks the
device ID and issues a normal reset. It does not repair disconnected debug
hardware or persistent flash corruption.

## Run a project and monitor serial output

```bash
agamemnon run --transport dap
agamemnon run --transport usb --flash --backup full-flash.bin
agamemnon monitor --port COM7 --baud 115200
```

With DAP, `run` loads the current project's MCU/fabric images into SRAM. With
USB it starts the project's MCU application; `--flash` programs it first and
therefore requires a backup.

`monitor` is a plain serial terminal; its default baud is 115200.

For project fields and template behavior, see [PROJECTS.md](PROJECTS.md).