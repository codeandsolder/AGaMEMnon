# Installation and tool bundles

AGaMEMnon can be used at several levels. Python-only inspection needs very
little; building MCU firmware, building FPGA images, and programming hardware
need additional tools.

## Source install

Python-only inspection, project creation and offline verification work on
Windows, Linux and macOS:

```sh
git clone https://github.com/codeandsolder/AGaMEMnon
cd AGaMEMnon
python3 -m pip install -e ".[programming]"
agamemnon --version
agamemnon doctor --no-hardware
```

On Windows, use `python` instead of `python3` if that is your installed launcher.
Git LFS is not required.

## What each capability needs

| Capability | Additional dependency |
|---|---|
| Decode, encode, inspect, scaffold, offline verify | Python + Git only |
| Build MCU firmware | `riscv-none-elf-gcc` or compatible RISC-V GCC |
| Build FPGA fabric | Yosys + AGaMEMnon's AGRV2K nextpnr backend |
| Program through USB CDC | pyserial + uploader already installed on target |
| Program through SWD/DAP | CMSIS-DAP + AGaMEMnon OpenOCD |
| Recover through mask-ROM UART | Pico 2 bridge + documented board wiring |

`agamemnon doctor` reports these independently, so a missing FPGA toolchain does
not stop image inspection and a missing probe does not stop builds.

Useful forms:

```sh
agamemnon doctor --no-hardware
agamemnon doctor --json --no-hardware
agamemnon doctor --probe-dap
```

UART target probing resets into ROM, so it only runs when an explicit
`--uart-port` is supplied.

## FPGA toolchain

Yosys is usually taken from
[OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build/releases).
Point `AGAMEMNON_OSS` at its root, then build the pinned AGRV2K nextpnr backend:

```sh
export AGAMEMNON_OSS=/opt/oss-cad-suite
./agamemnon/engine/uarch/agrv2k/build.sh
export AGAMEMNON_UARCH_NEXTPNR="$PWD/third_party/nextpnr/build/nextpnr-generic"
```

On PowerShell:

```powershell
$env:AGAMEMNON_OSS = "C:\tools\oss-cad-suite"
$env:AGAMEMNON_UARCH_NEXTPNR = "$PWD\third_party\nextpnr\build\nextpnr-generic.exe"
$env:AGAMEMNON_UARCH_NEXTPNR_RUNTIME = "C:\path\to\matching\runtime"
```

Building nextpnr from source needs a C++ toolchain, CMake, Boost and Eigen.
`AGAMEMNON_UARCH_NEXTPNR_RUNTIME` is mainly useful on Windows when the matching
runtime DLLs must be kept separate from OSS CAD Suite's environment.

## MCU compiler

Release bundles use xPack's `riscv-none-elf-gcc`. Source installs also accept:

- `riscv-none-elf-gcc`;
- `riscv64-unknown-elf-gcc`;
- `RISCV_PREFIX`;
- PlatformIO's `toolchain-agrv` package.

## OpenOCD for SWD/DAP

AG32 SWD support needs AGM's `target create riscv -dap` extension. Stock
upstream/OSS CAD Suite OpenOCD does not have it.

Install the AGaMEMnon build with:

```sh
agamemnon install-openocd
agamemnon doctor --probe-dap
```

The installer verifies the archive hash, installs it under
`~/.agamemnon/tools/openocd/`, and records the active executable/scripts in
`current.json`. Normal `probe`, `backup`, `flash` and `doctor` commands find it
automatically.

The exact OpenOCD source/patch inputs are pinned in
[`tools/openocd/manifest.json`](../tools/openocd/manifest.json). Release
archives include the executable, required libraries, source, patches, licenses,
build recipe, hashes and SBOM.

Builds are produced for:

- Windows x64;
- Linux x64;
- macOS arm64;
- macOS x64.

The macOS arm64 build has been run through firmware execution and a
backup/program/restore flash cycle on the L48 reference board. The Intel macOS
archive is built/tested in CI but has not had a separate Intel-Mac hardware run.

## Driver notes

- CMSIS-DAP uses HID. On Windows, do not replace its HID driver with a generic
  libusb driver.
- The AG32 uploader and Pico bridge use USB CDC ACM. Windows 10/11 has a class
  driver; Linux normally exposes `/dev/ttyACM*`.
- If Linux serial access is denied, add your user to the distro's serial group
  (commonly `dialout`).
- VID:PID `cafe:4001` is the flash-resident AGaMEMnon uploader. A factory board
  does not have it until the uploader is installed.
- The mask-ROM UART path needs the wiring described in
  [UART_BOOTLOADER.md](UART_BOOTLOADER.md).

## Development install

For engine/toolchain work:

```sh
python -m pip install -e ".[programming]"
./agamemnon/engine/uarch/agrv2k/build.sh
agamemnon doctor
```

Dependency/source pins live in
[`tools/bundle/manifest.json`](../tools/bundle/manifest.json). Bundle assembly is
documented in [`tools/bundle/README.md`](../tools/bundle/README.md).

## SDK bundles

The intended release bundle contains AGaMEMnon, Yosys/OSS CAD Suite, the
matching AGRV2K nextpnr runtime and RISC-V GCC. OpenOCD remains a paired
installable component.

Local Windows and Linux bundle candidates have passed the offline smoke tests,
including CLI diagnostics, routed-fixture verification, MCU compilation and
FPGA+MCU builds. At the time of this document they are still pre-release until
hosted archives and SHA-256 sidecars are published and independently checked.

Once a release is published, the intended installers are:

```powershell
./tools/install.ps1 -Version VERSION
```

```sh
sh tools/install.sh VERSION
```

They verify the archive hash, install into a versioned directory, create an
isolated Python environment, use only wheels shipped in the bundle, activate the
included tools, and run `doctor --no-hardware`.

Windows paths with spaces and non-ASCII characters are supported. AGaMEMnon
stages the few native tools that dislike such paths into an ASCII-only cache.
If the default cache location is unsuitable, set `AGAMEMNON_ASCII_TOOL_CACHE`.

For what the installed tools can currently build successfully on hardware, see
[STATUS.md](STATUS.md).