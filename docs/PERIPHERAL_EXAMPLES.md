# MCU and FPGA peripheral examples

The AG32 has two different kinds of “peripheral”:

- **hard MCU peripherals** such as UART, SPI, I²C, timers, USB and ADCs;
- **soft FPGA peripherals** written in Verilog and built from LUTs/flip-flops.

There is one AG32-specific wrinkle: most hard digital peripheral signals still
need the FPGA fabric to route them to package pins. Configuring UART0 in firmware
therefore does not by itself put UART0 TX on any particular lead.

## Example overview

| Function | MCU side | FPGA side | Current state |
|---|---|---|---|
| Timer | CLINT/MTIME and basic-timer examples | `timer_tick.v` | Machine-timer interrupt tested on hardware; soft timer simulates |
| GPIO | LED walkers | `gpio_walker.v` | Board LED mapping partly tested; soft example simulates |
| PWM | GPTIMER API | `pwm4.v` | Soft four-channel PWM simulates |
| UART | 5 hard UARTs + polling HAL | `uart_tx.v` | UART0/1/2 TX have tested routes; UART0 duplex retained; wider RX coverage open |
| SPI | SPI0/SPI1 polling HAL | `spi_master.v` | TX works in tested cases; fresh typed MISO currently broken/blocked |
| I²C | I²C0/I²C1 polling HAL | `i2c_writer.v` | Tested address/write/repeated-start/read transactions on both controllers |
| CAN | hard CAN 2.0 | none | Registers exercised; no useful wire-level CAN test yet |
| USB | hard FS/OTG + PHY | not a plain fabric peripheral | Flash-resident CDC uploader works |
| Watchdog | APB watchdog driver | n/a | Snapshot + supervised reset tested |
| RTC | RTC driver | n/a | Registers/config tested; timekeeping still needs a low-speed clock test |
| DMA | memory DMA example | normal RTL datapaths | Four-word SRAM copy tested |
| CRC | hard CRC | normal RTL | CRC-32/MPEG-2 known-answer tested |
| External AHB | MCU accesses fabric window | constant/register-bank slaves | Several retained read/write/register-bank examples work |
| Ethernet | hard MAC | none here | Needs PHY/clock/pin work |
| ADC/DAC/comparator | hard analog blocks | not synthesizable from LUT RTL | Small bench subset works through vendor analog wrapper; open bitgen does not emit it yet |

For the exact current hardware boundaries, see [STATUS.md](STATUS.md).

## Build the MCU examples

Windows:

```powershell
./examples/riscv_mcu/build.ps1
```

Linux/macOS:

```sh
sh examples/riscv_mcu/build.sh
```

Some useful outputs:

| Image | Load address | What it does |
|---|---:|---|
| `timer_led_walk_flash.bin` | `0x80000000` | CLINT-timed LED walk |
| `timer_led_walk_usb_app.bin` | `0x80010000` | same program for resident USB uploader |
| `basic_timer_led_walk_flash.bin` | `0x80000000` | polls hard TIMER0 |
| `hard_peripheral_inventory.bin` | `0x20000000` | reads the SDK's peripheral map without enabling devices |
| `crc_self_test.bin` | `0x20000000` | hard CRC known-answer test |
| `watchdog_snapshot.bin` | `0x20000000` | read-only watchdog state |
| `watchdog_supervised.bin` | `0x20000000` | intentionally triggers a watchdog reset |
| `rtc_count.bin` | `0x20000000` | RTC configuration/counter probe |

Example SRAM inventory run:

```powershell
agamemnon sram .tmp/riscv_mcu/hard_peripheral_inventory.bin --words 5 --sleep 100
```

The inventory intentionally does not initialize every peripheral. Doing that
blindly would be a terrible board test: watchdog resets the MCU, I²C needs
pull-ups, CAN needs a transceiver, Ethernet needs a PHY, and several signals
share package routing resources.

### GPIO warning

“Blink every GPIO” is not a useful test on this board. Pins overlap USB,
oscillators, debug, boot UART, CAN, I²C and other attached hardware.

The examples that say “all LEDs” mean the four board LEDs, not every package
GPIO. See [MCU_PIN_ROUTING.md](MCU_PIN_ROUTING.md) for how MCU signals reach
package pins.

## Simulate the FPGA examples

Windows:

```powershell
$env:AGAMEMNON_OSS = "C:/path/to/oss-cad-suite"
./examples/peripherals/fpga/simulate.ps1
```

Linux/macOS:

```sh
sh examples/peripherals/fpga/simulate.sh
```

`peripheral_showcase.v` instantiates the small soft timer, LED walker, four PWM
channels, UART TX, SPI master and I²C writer together.

`showcase_top.v` exposes only the four commonly used L48 fabric LED pads and is
a much safer physical build than guessing package pins for every protocol.

Example build:

```powershell
agamemnon build examples/peripherals/fpga/showcase_all.v --uarch `
  --pcf examples/peripherals/fpga/showcase_L48.pcf `
  -o .tmp/peripheral_showcase.bin
```

The soft protocol blocks are deliberately small examples:

- UART: transmit only;
- SPI: mode 0, one byte per start;
- I²C: one-byte single-master writer, drive-low enables for open-drain IO,
  external pull-ups required.

They are examples for using the fabric, not attempts to replace complete
production protocol IP.

## USB is a hard peripheral

The USB connector is attached to the AG32's dedicated USB PHY/controller. It is
not an ordinary pair of FPGA-routable GPIOs.

The tested CDC uploader uses the hard USB controller with TinyUSB. The FPGA can
still provide buffers or custom MCU-visible logic behind USB firmware, but a
plain Verilog USB transceiver cannot simply be routed to D+/D- through the L48
GPIO fabric.

See [USB_CDC_UPLOADER.md](USB_CDC_UPLOADER.md).

## Analog blocks

ADC, DAC and comparator registers can be exercised from the MCU once the analog
hard macro is present. The current bench tests use the vendor `analog_ip`
wrapper because AGaMEMnon does not yet emit that macro itself.

See [ANALOG_FABRIC_BOUNDARY.md](ANALOG_FABRIC_BOUNDARY.md).

## Vendor references

- [AGM AG32 product page](https://www.agm-micro.com/products.aspx?id=3113&p=37)
- [AG32 MCU Reference Manual](https://www.agm-micro.com/upload/userfiles/files/AG32%20MCU%20Reference%20Manual%2820250515%E4%BF%AE%E8%AE%A2%E7%89%88%EF%BC%89.pdf)
- [AG32 MCU datasheet](https://www.agm-micro.com/upload/userfiles/files/AG32_DATASHEET_202303.pdf)
- [AGRV2K programmable-logic manual](https://www.agm-micro.com/upload/userfiles/files/AGRV2K_Rev2_0.pdf)

The external AGM PlatformIO SDK is used as a cross-check; its code is not copied
into this repository.