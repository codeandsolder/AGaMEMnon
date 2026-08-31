# Programming the AG32

AGaMEMnon currently has three programming paths:

- **CMSIS-DAP/SWD + OpenOCD** drives the on-chip flash controller directly.
- **UART0 mask ROM through a Raspberry Pi Pico 2** is independent of main
  flash. The ROM protocol is understood, but the current Pico-to-L48 five-wire
  bench setup still needs its final target-side hardware run.
- **USB CDC uploader** is a TinyUSB application installed in main flash. It is
  convenient once installed, but it is not a ROM recovery mechanism.

For new fabric work, use volatile SRAM first. It is faster to recover from and
does not touch flash. [STATUS.md](STATUS.md) lists the currently known fabric
failures.

| Transport | Untouched stock board | Recovery capable | Hardware modification |
|---|---|---|---|
| DAP/SWD | Yes, with AGaMEMnon's OpenOCD build | Yes | No |
| USB CDC | No; uploader must first be installed | No | No |
| UART mask ROM/Pico | ROM supports it | ROM mechanism yes; current Pico bench setup pending | Current L48 board/harness requires added wiring |

The USB uploader is neither mask-ROM USB boot nor USB DFU class. See
[USB_CDC_UPLOADER.md](USB_CDC_UPLOADER.md) for its source corrections, backup
hashes and bench transcript.

An untouched stock board cannot upload over USB: the loader is absent from
factory flash on the board used for testing. Install it once through SWD or the
UART0 ROM, then use the right-hand target USB-C connector for later transfers.
The tested standalone loader returns to itself after reset. A persistently
booted uploaded application needs a separate, non-overlapping layout;
`0x80008000` must not be used because the existing compressed fabric begins at
`0x80008100`.

None of the three AGaMEMnon CLI transports uses the vendor `agrv` flash driver.
The USB client implements the loader protocol directly in Python;
`agrv32flash` is kept as an independent comparison tool.

For MCU binary format, linker addresses, startup requirements, and runnable
SRAM/native-flash/USB examples, see
[RISCV_MCU_PROGRAMMING.md](RISCV_MCU_PROGRAMMING.md).

Hardware commands need an OpenOCD build with AGM's RISC-V-over-ADIv5-DAP target
option, `target create riscv -dap`. Stock upstream, xPack, and OSS CAD Suite
binaries do not contain that extension.

```bash
agamemnon install-openocd
agamemnon doctor --probe-dap
```

`AGAMEMNON_OOCD_CFG` and `AGAMEMNON_OOCD_SCRIPTS` override the packaged target
configuration and OpenOCD script directory. `AGAMEMNON_OPENOCD` is an explicit
executable override.

DAP writes are not retried. If OpenOCD fails during an SRAM/program session,
identify the target again before doing anything else; after a failed write its
state may be partly changed. Flash commands require a backup for recovery.

The tested OpenOCD build is official parent `a17c5f5a`, Gerrit 9590 patchset 2
(`9aa0f976`), plus AGaMEMnon's ADIv5-config repair (`f96d840a`). The Windows
artifact has been tested for probe, halt, register and SRAM access, SRAM
firmware execution, full flash backup, sector program/readback/restore,
full-device hash restoration, and reset recovery. The machine-readable record
is [`evidence/openocd-windows-ag32.json`](evidence/openocd-windows-ag32.json).

## Commands

| Command | Behavior |
|---|---|
| `agamemnon probe` | reads `DEVICE_ID`; no persistent write |
| `agamemnon probe --transport usb [--port PORT]` | identifies the resident CDC uploader and AG32 |
| `agamemnon probe --transport uart [--port PORT]` | resets through Pico into ROM and identifies AG32 |
| `agamemnon sram FW --fabric FABRIC` | loads fabric and firmware into SRAM and runs them |
| `agamemnon fcb-restream FW IMAGE...` | prepares a one-firmware/many-image restream plan; `--execute-sram` runs it |
| `agamemnon hil-campaign WORKLIST --root ROOT` | prepares an ordered HIL work list |
| `agamemnon backup FILE` | reads the complete 256-KiB main flash |
| `agamemnon flash FILE --addr ADDR --backup FILE` | backs up flash, erases touched sectors, programs, reads back, and verifies |
| `agamemnon backup FILE --transport usb` | reads all flash through USB CDC |
| `agamemnon flash FILE --addr ADDR --backup FILE --transport usb` | preserves sectors and verifies through USB CDC |
| `agamemnon go ADDR --transport usb` | launches a separately linked application |
| `agamemnon image ...` | plans or writes MCU and uncompressed fabric regions |
| `agamemnon image ... --write-options` | opt-in, unsupported option-byte pointer operation |
| `agamemnon uart-probe [--port PORT]` | resets into the serial ROM and reads the device ID |
| `agamemnon uart-backup FILE [--port PORT]` | reads all 256 KiB through UART0 |
| `agamemnon uart-flash FILE --addr ADDR --backup FILE [--port PORT]` | preserves sectors, writes, verifies, and resets into flash |
| `agamemnon uart-reset [--port PORT]` | selects normal boot and resets the target |

Native mask-ROM USB boot and USB DFU are not implemented. The separate
flash-resident CDC uploader has been tested on the LQFP-48 board.

## Pico 2 UART programmer

Flash the checked-in bridge firmware first (the port shown here is an example):

```powershell
arduino-cli compile --fqbn rp2040:rp2040:rpipico2 pico/ag32_uart_programmer
arduino-cli upload -p COM6 --fqbn rp2040:rp2040:rpipico2 pico/ag32_uart_programmer
```

For the LQFP-48 AG32 board, add these wires to the existing Pico harness:

| Pico 2 | Direction | AG32 LQFP-48 signal | Package pin |
|---|---:|---|---:|
| GP20 / UART1 TX | -> | UART0_RX / `PIN_31` | 31 |
| GP21 / UART1 RX | <- | UART0_TX / `PIN_30` | 30 |
| GP22 | -> | BOOT0 | 44 |
| GP26 | open-drain -> | NRST | 7 |
| GP27 | open-drain -> | `PIN_20` / BOOT1 | 20 |
| GND | -- | GND | any board ground |

Cross TX and RX as shown. Both sides are 3.3-V logic. Keep the AG32 board on
its normal supply and the Pico on USB; **do not connect Pico VBUS or 3V3 to the
AG32 power rail**. The Pico drives BOOT1 low only while reset is being latched,
then releases GP27 so target firmware can use `PIN_20`. NRST is only driven low
and otherwise released.

The strap pins may have passive pull resistors, but they must not be hard-tied
or actively driven by another device. Remove a hard BOOT0/BOOT1 strap before
connecting the Pico. A 1-kohm series resistor in each Pico control lead
(GP22/GP26/GP27) is useful for contention protection.

The board's AGM DAP-Link adapter is also wired to UART0. **Disconnect or mux the
DAP-Link TX-to-AG32-UART0_RX path before connecting Pico GP20.** Two push-pull
TX outputs must not be connected together. DAP-Link RX may remain connected to
AG32 UART0_TX because it is only another input. If the board has no removable
UART jumper, open its TX solder bridge or add a 3.3-V UART mux; a series
resistor alone is not a reliable bus selector.

Install the host dependency and check the link:

```bash
python -m pip install -e ".[programming]"
agamemnon uart-probe --port COM6
```

The AG32 ROM uses UART0 at 460800 baud, 8 data bits, no parity, and one stop
bit. The Pico owns that target-side baud rate; the USB CDC baud selected by the
host does not matter. If exactly one Pico is connected, `--port` can be omitted.

The bridge firmware and host protocol have software tests, and the Pico on the
bench has been flashed and USB-smoke-tested. The remaining gap is the complete
five-wire Pico-to-AG32 target-side run. In other words: the ROM mechanism is
known, but this particular recovery setup is not yet fully hardware-tested.

The source trail, ROM-protocol findings, current bench transcript, and proposed
programming interposer are in [UART_BOOTLOADER.md](UART_BOOTLOADER.md).

## Volatile SRAM execution

```bash
agamemnon sram firmware.bin --fabric design.bin --words 10
```

The command places:

```text
0x20000000  firmware
0x20001000  result words
0x20002000  99,944-byte uncompressed fabric image
```

The firmware calls `ag32_fcb_config()`, performs the test or application work,
and stores optional observations at `0x20001000`. SRAM execution does not touch
flash and is the preferred development path for new fabric images.

### FCB restream instrument

`qualification/fcb_restream_probe.c` is an SRAM-resident mailbox loop over the
same `ag32_fcb_config()` AUTO stream. It lets the host load the firmware once
and stage successive full images at `0x20002000`. Requests carry the image
length and a sequence number; malformed requests are rejected before FCB
access. An FCB error latches until MCU reset.

The command is desk-only by default:

```bash
agamemnon fcb-restream fcb_restream_probe.bin first.bin second.bin
```

It checks that each image is 99,944 bytes and prints a SHA-256 plan with one
firmware load and no flash writes. `--execute-sram` runs the sequence through
DAP.

One A/B/A sequence has been tested on L48: the retained constant AHB endpoint
(`0x4147414d`), the same routed image with its shared HRDATA-one LUT changed to
constant zero (`0x00000000`), then the retained endpoint again. All three
configurations returned `FCB_STAT_OK`; direct AHB reads were A/B/A and the board
was reset afterwards. See `qualification/fcb_restream_evidence.jsonl`.

This demonstrates repeated configuration of that setup. It does not yet cover
arbitrary hot reconfiguration, state migration, continuity-sensitive outputs,
or unattended use.

## Main-flash programming

Create a complete backup before every write:

```bash
agamemnon backup full-flash.bin
agamemnon flash payload.bin --addr 0x80020000 --backup full-flash.bin
```

Main flash occupies `0x80000000..0x8003ffff`. `flash` erases every 4-KiB sector
spanned by the payload, programs through the controller at `0x40001000`, reads
the region back, and compares it byte-for-byte. A readback failure, truncation,
or mismatch exits nonzero. Before a DAP, USB, or UART mutation/execute command,
the host also reads the device ID and requires `0x40200001`.

Option bytes occupy a separate region at `0x81000000` and are not modified by
`flash`.

The UART equivalent also requires a backup and preserves complete touched
sectors:

```bash
agamemnon uart-flash payload.bin --addr 0x80020000 \
  --backup pre-uart-write.bin --port COM6
```

It reads the whole main flash to the backup file, overlays the payload on the
saved sector images, erases only touched 4-KiB pages, writes those pages, and
compares the complete readback. After a clean comparison it lowers BOOT0 and
resets into flash. A failed write leaves BOOT0 high in serial-ROM recovery.

## Existing compressed boot layout

On the tested board, option bytes select a compressed fabric image at
`0x80008100` and a decompressor at `0x80007000`. Replace the image without
changing the option pointers:

```bash
agamemnon backup full-flash.bin
agamemnon flash design.bin.comp --addr 0x80008100 --backup full-flash.bin
```

The decompressor and fabric image can share a 4-KiB erase sector. Preserve the
whole affected sector; erasing only the visible image fragment can destroy part
of the decompressor.

Fabric configuration from flash occurs on power-on. A debugger warm reset does
not rerun the complete fabric boot sequence.

## Image planning

```bash
agamemnon image --fabric design.bin --mcu firmware.bin
agamemnon image --fabric design.bin --mcu firmware.bin \
  --flash --backup full-flash.bin
```

Without `--flash`, `image` prints a write plan. With `--flash`, it writes the
MCU and uncompressed fabric regions through the same verified main-flash path.
Those writes do not change the boot ROM's fabric pointer. `--flash` requires a
complete `--backup` before erase begins.

`--write-options` also attempts to write the uncompressed-config pointer at
`0x81000030`. This is experimental. It requires an explicit flag, `--flash`,
the main-flash backup, and a separate `--option-backup` containing all 128
option bytes. Do not use it as the normal boot-layout path.

## Recovery

- Keep the complete 256-KiB backup off-board.
- Restore it with
  `agamemnon flash full-flash.bin --addr 0x80000000 --backup pre-restore.bin`.
- SWD works independently of main-flash contents.
- Once the Pico UART setup is fully bench-tested, the ROM path is intended to
  provide the same flash-independent recovery with
  `agamemnon uart-flash full-flash.bin --addr 0x80000000 --backup pre-restore.bin`.
- `uart-reset` drives BOOT0 low and pulses NRST without writing flash.

See [flashboot/FLASH_LAYOUT.md](flashboot/FLASH_LAYOUT.md) for the memory map
and [flashboot/flash_controller.md](flashboot/flash_controller.md) for the
controller register sequence.
