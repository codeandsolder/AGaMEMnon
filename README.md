# AGaMEMnon

The [AG32](https://www.agm-micro.com/) is a microcontroller with a small FPGA
bolted to it. It's a real RV32IMAFC core with hard peripherals (UART, SPI, I²C,
CAN, USB, Ethernet MAC, timers, ADC/DAC, GPIO), _plus_ a small programmable
fabric sitting between those peripherals and the pins:
<p align="center">
<table>
<tr>
<th align="left">RISC-V MCU</th>
<th align="left">FPGA fabric</th>
</tr>
<tr valign="top">
<td>
<ul>
<li>RV32IMAFC core, up to 248&nbsp;MHz, hardware FPU</li>
<li>256&nbsp;KB Flash (zero-wait), 128&nbsp;KB SRAM</li>
<li>5&times; UART &middot; 2&times; I²C &middot; SPI</li>
<li>1&times; CAN&nbsp;2.0 &middot; USB&nbsp;FS+OTG &middot; Ethernet MAC</li>
<li>3&times; 12-bit ADC (17&nbsp;ch, 3&nbsp;MSPS) &middot;
2&times; 10-bit DAC</li>
<li>2&times; comparator &middot; RTC &middot; watchdog</li>
<li>basic + advanced timers</li>
</ul>
</td>
<td>
<ul>
<li>2112 LUT4s</li>
<li>2112 flip-flops</li>
<li>4 block RAMs</li>
<li>1 PLL</li>
<li>5 global clocks</li>
<li>architecture advertises up to 128 fabric I/O</li>
</ul>
</td>
</tr>
</table>
</p>

The fabric can be independent logic, a pin-routing layer for hard peripherals,
or a memory-mapped coprocessor beside the MCU. The
[AG32 overview](docs/AG32_OVERVIEW.md) explains the device, naming, clocks,
boot paths, packages, and documentation landscape.

In principle the fabric gives you flexible UART placement, state machines in
signal paths, memory-mapped custom peripherals, runtime muxing, and other FPGA
tricks around a normal MCU. Think of the AG32 as something between a Raspberry
Pi Pico with much more flexible PIO and a PSoC whose programmable blocks are an
actual FPGA fabric.

The AG32 has almost no English-language documentation. The normal vendor FPGA
flow is a Windows-heavy Altera Quartus II fork plus a black-box backend called
`af.exe`, distributed through a Baidu Netdisk link. There is no documented open
bitstream format and the useful architecture data lives inside vendor tools.

*AGaMEMnon* takes Verilog and produces a flashable AG32 fabric bitstream —
synthesis, pack, place, route, bitstream generation, and programming, with no
vendor binary in the path. It also includes an SDK for the RISC-V side.

This is roughly IceStorm for a chip almost nobody has heard of. Verilog can be
synthesized, placed, routed, loaded on real silicon, and connected to the MCU.
Some things work, many still do not, and a few designs appear to fail even when
built with the original vendor backend. The current state is documented in
[STATUS.md](docs/STATUS.md).

There's a writeup of the reverse engineering
[here](http://bbenchoff.com/pages/AGaMEMnon.html).

Watch the video demo:

[![AGaMEMnon video demo][video-thumbnail]][video-demo]

[video-thumbnail]: https://img.youtube.com/vi/udDq3NHxerc/maxresdefault.jpg
[video-demo]: https://www.youtube.com/watch?v=udDq3NHxerc

## What this project is

The goal is an open replacement for the AG32 programmable-logic toolchain. The
vendor flow is roughly:

* **Yosys** — a modified copy maps Verilog to the vendor primitive library.
  AGaMEMnon recovered the LUT, BRAM, carry and IO cell definitions from that
  flow. The vendor Yosys does not place or route.
* **`af.exe`** — the actual FPGA backend. It packs, places, routes and emits the
  bitstream. It also contains the architecture database: routing topology,
  mux encodings, clock/PLL data and configuration-bit maps. Recovering this is
  most of the reverse-engineering work.
* **Quartus / `Supra.exe`** — mostly the surrounding migration/frontend tools
  for AGM's Altera-like FPGA families. Packing, placement and routing still end
  up in `af.exe`.

`af.exe` is a very useful source for encodings and intended topology, but it is
not a perfect model of the silicon. It can produce designs that do not behave
as expected on the chip, and the current test campaign has found vendor failures
and unstable vendor results as well as AGaMEMnon failures.

AGaMEMnon therefore keeps the reverse-engineered encoding data separate from
hardware testing. Known bad images and several known bad logical compositions
are blocked. Unknown designs can still be wrong, so support is currently much
narrower than "arbitrary Verilog works".

## Quick start

```sh
git clone https://github.com/codeandsolder/AGaMEMnon
cd AGaMEMnon
python3 -m pip install -e ".[programming]"
agamemnon doctor --no-hardware
```

All required data is stored as ordinary Git objects; Git LFS is not required.
The repository includes a routed counter fixture that can be verified without
a board or FPGA toolchain:

```sh
agamemnon verify tests/fixtures/counter_ahb_routed.json --cycles 8
```

Create the fabric-free starting project:

```sh
agamemnon new hello --board ag32vf303-l48
cd hello
agamemnon build
agamemnon run --transport dap
```

This default needs only a compatible RISC-V GCC and runs from volatile SRAM.
`agamemnon doctor` reports separate inspection, MCU-build, fabric-build, and
hardware-transport capabilities.

Two larger templates replay known working designs rather than promising a
generic implementation:

- `--template mcu-fpga` replays the reviewed L48 public32 AHB map: ID32
  `0x4147414d` at +0, scratch16 at +4, counter3 at +8, and W1C1 at +c.
- `--template serv-blinky` replays one retained L48 SERV route. Fresh arbitrary
  SERV placement and wider direct-D placement are not supported yet.

The current checkout has a review gate on the public32 composition. If the
composer reports `candidate hash does not match reviewed artifact`, inspect the
change instead of simply updating the expected hash. See
[Landing a chip-database change](docs/LANDING_A_CHIPDB_CHANGE.md).

## Programming and recovery

SWD/DAP is the simplest transport on a stock board and supports volatile
MCU/fabric loads. The USB CDC uploader is convenient after its loader has been
installed. The UART0 mask ROM is independent of main flash; the checked-in Pico
bridge implements that protocol, but the current five-wire Pico-to-L48 setup
still needs its final target-side bench run. See
[Programming](docs/PROGRAMMING.md) for the details.

## Documentation

| Read | For |
|---|---|
| [Status](docs/STATUS.md) | current support, open defects, and test state |
| [Vendor parity](docs/VENDOR_PARITY.md) | the 105-design campaign and its results |
| [Installation](docs/INSTALLATION.md) | host tools, bundles, and drivers |
| [Usage](docs/USAGE.md) | command reference and build behavior |
| [Projects](docs/PROJECTS.md) | manifests and replay templates |
| [Programming](docs/PROGRAMMING.md) | DAP, USB, UART, and persistent writes |
| [Examples](examples/README.md) | runnable RTL and firmware |
| [MCU SDK](sdk/README.md) | open HAL coverage |
| [MCU HAL reference](docs/HAL_MCU_REFERENCE.md) | MCU registers and drivers |
| [FPGA HAL reference](docs/HAL_FPGA_REFERENCE.md) | fabric resources and configuration fields |
| [Architecture](docs/ARCHITECTURE.md) | synthesis, routing, bitgen, and verification |
| [Routing admission](docs/ROUTING_ADMISSION.md) | which routing edges are allowed and why |
| [MCU/fabric boundary](docs/MCU_AHB_REGISTER_BANK.md) | AHB compositions and remaining gaps |
| [Hardware validation](docs/HARDWARE_VALIDATION.md) | board-observed results, including failures |
| [Roadmap](ROADMAP.md) | unfinished work |
| [Documentation issues](docs/DOCUMENTATION_ISSUES.md) | contradictions and prose that needs reconciliation |
| [Notices](NOTICE.md) | licensing and recovered-data provenance |

The detailed [documentation index](docs/AG32_OVERVIEW.md#documentation-map)
links the remaining bitstream, clock, pin-routing, peripheral and research
records.

## Contributing and support

New hardware evidence is especially valuable. Read
[CONTRIBUTING.md](CONTRIBUTING.md) before submitting code or qualification
records. Use [SUPPORT.md](SUPPORT.md) for unexpected behavior and
[SECURITY.md](SECURITY.md) for security reports. User-visible changes are in
[CHANGELOG.md](CHANGELOG.md).

## The name

AGaMEMnon: “AG” plus something about memory. It was named before Nolan's
*Odyssey* came out.
