# MCU peripheral and pin routing

The AG32 does not have a normal STM32-style alternate-function matrix where
firmware picks “UART on pin 12” from a mux register. Most hard-peripheral
signals still pass through the FPGA fabric before they reach package pins.

`GPIOAFSEL` only chooses who controls a GPIO line:

- `0`: software GPIO (`DATA` / `DIR`);
- `1`: hardware/peripheral control.

It does **not** select UART/SPI/I²C/CAN, and it does not determine which package
pin a peripheral reaches. That part comes from the loaded fabric image.

This means a peripheral driver and a pin route are separate things: firmware can
configure UART0 correctly while the fabric routes UART0 nowhere useful.

## Current L48 routes

The useful tested routes are summarized below. Detailed hashes, captures and
bench records live in [HARDWARE_VALIDATION.md](HARDWARE_VALIDATION.md) and the
files under `qualification/`.

### GPIO / plain fabric IO

| Route | Current result |
|---|---|
| `GPIO4.1 -> PIN_34 -> LED1` | Works on the reference board |
| GPIO4 <-> fabric bridge | Tested with a four-bit inverted loopback |
| GPIO5 data/OE/input | Two output-data/OE lanes and one return input lane tested |
| `PIN_25..PIN_28` fabric outputs | All four work on L48 |
| `PIN_10..PIN_19` fabric outputs | All ten have tested exact output routes; fresh `--uarch` routing still has a PIN_15 gap |
| `PIN_25` OE/readback | Several static and dynamic OE cases work |
| fabric inputs on `PIN_10`, `11`, `12`, `15`, `19` | Retained exact examples work, but new PIN_10/PIN_12 input compositions have also failed |

The last row is important: input routing is not generally solved just because
some older exact routes work. `VP-AGM-008` tracks the newer input failures.

### UART

- UART0 application full duplex works on `PIN_30` / `PIN_31`.
- UART0, UART1 and UART2 TX have working campaign routes.
- UART0 RX works in retained exact routes.
- UART3/4 TX and broad RX coverage are still open.
- The mask-ROM UART uses UART0, but the current Pico recovery harness has its own
  unfinished target-side qualification work; see [UART_BOOTLOADER.md](UART_BOOTLOADER.md).

### I²C

I²C0 and I²C1 have working L48 open-drain routes. The main tested transaction is
an address `0x55` write/read exchange on `PIN_11`/`PIN_15`, with an RP2350 acting
as the external slave.

I²C needs external pull-ups; routing the signals alone is not enough.

### SPI

SPI0 and SPI1 TX work in tested configurations.

SPI MISO is the awkward part: newer typed SPI0/SPI1 MISO compositions returned
all ones even though the external slave and vendor controls worked. Those typed
MISO paths are currently blocked (`VP-AGM-008`).

An older exact SPI0 receive image still works as a retained example, but it is
not evidence that arbitrary fresh MISO routing works.

### CAN / Ethernet / USB

- CAN and Ethernet do not yet have package-route qualification.
- CAN additionally needs a transceiver.
- Ethernet needs the PHY and clocks.
- USB D+/D- use the hard USB PHY and are not ordinary fabric GPIO signals.

## Practical rule

Do not write firmware APIs that pretend the pin mapping is fixed in silicon.
This is the wrong abstraction:

```text
set_uart_pin(UART0, PIN_12)
```

A board definition should instead pair:

- a specific part/package/board;
- the fabric configuration that creates the route;
- the firmware configuration for the peripheral.

If you load an unknown fabric image, assume any previous hard-peripheral pin
mapping may have disappeared.

## Firmware and board guidelines

1. Peripheral drivers should configure the controller registers, not claim a
   package pin by themselves.
2. Board definitions may name a peripheral pin when they also identify the
   matching fabric route/profile.
3. Keep a recovery/debug path available when experimenting with routing.
4. Plain GPIO examples should clear `AFSEL` and only drive known board nets.
5. I²C requires open-drain + pull-ups; CAN requires a transceiver; Ethernet
   requires its PHY.
6. Do not assume the same package pin number means the same tested route on a
   different package or board.

## Adding a route

For a new physical route, record:

- chip/package/board;
- fabric source and image hash;
- signal source/sink and direction;
- pin and electrical setup;
- firmware used;
- what was actually observed.

Test the route by itself before burying it in a large design. Once it works, add
the result to the normal qualification records.