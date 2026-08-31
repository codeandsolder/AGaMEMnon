# AGaMEMnon projects

An `agamemnon.toml` file keeps the Verilog sources, MCU firmware, pin mapping,
clocks and output layout for one project in one place.

Create a project with:

```text
agamemnon new hello --board ag32vf303-l48 --template mcu-blink
cd hello
agamemnon doctor
agamemnon build
agamemnon run --transport dap
```

Available templates:

- `mcu-blink`
- `fpga-io`
- `mcu-fpga`
- `mcu-fpga-registers`
- `serv-blinky`
- `uart`
- `usb-cdc`
- `safe-recovery`

## Which template should I use?

### `mcu-blink`

The simplest starting point. It does not use the FPGA fabric at all, so it is a
good way to verify the RISC-V compiler, linker and programming path first.

### `fpga-io`

A small fresh FPGA build that drives a static pattern through four LUTs to the
four tested L48 LED outputs.

The old `fpga-blink` template used a counter and required many more flip-flop
placements than are currently reliable. The name remains as a deprecated alias
for `fpga-io` rather than bringing that old design back.

### `mcu-fpga` / `mcu-fpga-registers`

These are aliases for the same retained L48 AHB register-bank example:

```text
+0x0  ID32       0x4147414d
+0x4  scratch16
+0x8  counter3
+0xc  W1C1
```

At the moment this template replays a reviewed routed design instead of freshly
routing a generic register bank. The generated project checks the source,
routed netlist, board/device and output hashes so accidental changes are caught.

If the composer reports a checkpoint/hash mismatch, inspect the route/config
change rather than simply updating the expected hash. See
[LANDING_A_CHIPDB_CHANGE.md](LANDING_A_CHIPDB_CHANGE.md).

### `serv-blinky`

Replays the retained L48 SERV route and builds a small MCU loader around it. It
is useful as a large integration example, but it does not freshly place and
route SERV from source.

### `uart`

A starting point for the supported UART path. Check [STATUS.md](STATUS.md) for
which UART directions/controllers currently work.

### `usb-cdc`

Creates an external PlatformIO project for the tested flash-resident USB CDC
uploader/application flow. It references the AGM SDK rather than copying that
SDK into this repository.

### `safe-recovery`

Sets up a project around the programming/recovery workflows rather than a new
FPGA design.

## Manifest format

Example:

```toml
[project]
name = "example"
board = "ag32vf303-l48"
device = "AGRV2KL48"

[fabric]
sources = ["logic/bus.v", "logic/top.v"]
top = "top"
pcf = "board.pcf"
output = "build/fabric.bin"
uarch = true
mcu_bridge = true
freq = 10
hse = 8

[mcu]
sources = ["src/main.c", "src/interrupts.c"]
include_dirs = ["include"]
linker = "@sdk/link_sram.ld"
output = "build/mcu.bin"
march = "rv32imac"
mabi = "ilp32"

[flash]
mcu_address = 0x80010000
fabric_address = 0x80008100
```

Maintained linker-script shortcuts are:

- `@sdk/link_sram.ld`
- `@sdk/link_flash.ld`
- `@sdk/link_usb_app.ld`

You can also point `linker` at your own script.

Fabric projects can contain multiple Verilog files and an explicit top module.
For one-off builds, the CLI equivalents are repeated `--source` arguments plus
`--top`.

`freq` selects both the generated fabric clock profile and the timing target.
The default is 10 MHz unless overridden by `--freq`.

## Running a project

For development, the normal path is:

```text
agamemnon build
agamemnon run --transport dap
```

DAP loads the MCU and fabric into SRAM and does not modify flash.

USB `run` only jumps to an already installed application unless `--flash` and a
backup are explicitly requested. The mask-ROM UART path is mainly for recovery
and needs the extra L48 wiring described in [PROGRAMMING.md](PROGRAMMING.md).