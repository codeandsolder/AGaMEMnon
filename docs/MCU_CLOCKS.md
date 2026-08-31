# MCU and fabric clocks

There are several clock domains on the AG32, and the current open SDK does not
fully characterize the MCU clock tree yet.

## Current answer

| Clock | What we currently know |
|---|---|
| MCU `SYSCLK` | SDK inherits the existing clock state; it does not switch HSI/HSE/PLL itself |
| MTIME | measured **14.08 MHz** in one SRAM-loaded, PLL-unconfigured L48 setup |
| UART0 reference | measured/back-solved at about **14.47 MHz** in the same kind of setup |
| SPI0 reference | absolute frequency **unknown**; documented power-of-two divider behavior works |
| Fabric `SYSCLK` | selected by `build --freq` / `[fabric].freq`; many HSE=8 points from 4-248 MHz have been measured |
| External-AHB `bus_clock` | measured at **one state transition per MTIME tick** in the tested topology; absolute rate not settled |

The important practical rule is: **do not use the 248 MHz CPU maximum as if it
were the current peripheral clock.** That mistake produced a real UART baud-rate
bug.

Also keep MCU and fabric clocks separate. `agamemnon build --freq 25` changes
the emitted **fabric** clock profile and timing target; it does not re-clock the
RISC-V core to 25 MHz.

## Vendor-documented MCU clock tree

The AG32 MCU reference manual documents:

| Source / clock | Vendor range or rule |
|---|---|
| HSI | reset source; 10-40 MHz RC, 20 MHz typical |
| HSE crystal/resonator | 4-24 MHz |
| HSE bypass | up to 100 MHz |
| PLL input | 4-50 MHz |
| PLL output | 2-300 MHz electrical range |
| RISC-V CPU | 248 MHz maximum |
| external/fabric clock | fourth system-clock source |
| PBUS/APB | `SYSCLK / (PBUS_DIV + 1)`, divisor 1-16 |
| flash SPI clock | `SYSCLK / (SCLK_DIV + 1)`, divisor 1-16 |
| USB | manual requires 60 MHz PLL output |

The L48 reference board has an 8 MHz HSE.

## What the SDK does today

AGaMEMnon startup/examples generally inherit whatever MCU clock state existed
before entry. The open HAL can read the documented source-select and divider
fields, but it intentionally does **not** provide a runtime clock-switch API yet.

`ag32_pbus_hz(sysclk_hz)` only divides a frequency supplied by the caller. It
does not discover the current SYSCLK. Passing `248000000` because that is the
part maximum simply gives a wrong peripheral-rate calculation if the chip is
actually running much slower.

## The UART bug that exposed this

On 2026-08-14 an SRAM-loaded test called:

```text
ag32_pbus_hz(248000000)
ag32_uart_init(UART0, pbus, 9600)
```

The resulting UART was about **560 baud**, not 9600.

Back-solving from the programmed PL011 divisors and measured bit period put the
UART0 reference near **14.47 MHz**.

After switching the example to the measured UART reference, the same setup
successfully exchanged data at nominal:

- 9600 baud;
- 38400 baud;
- 115200 baud.

A later full-duplex test transferred 4096 bytes in each direction at all three
rates, and 38400 baud also worked with 7E1, 8E1, 8O1 and 8N2 framing.

## Measurements on the SRAM-loaded L48 board

| Domain | Result | Notes |
|---|---:|---|
| MTIME | **14.08 MHz** | measured repeatedly against host time |
| UART0 reference | **~14.47 MHz** | inferred from measured baud + programmed PL011 divisors |
| SPI0 reference | **unknown** | relative divider behavior tested; no trustworthy absolute calibration yet |

MTIME and UART0 are within about 3%, which is compatible with a common system/APB
clock in that setup. That is not enough to claim every peripheral has the same
reference clock.

### The bad SPI estimate

An earlier SPI experiment appeared to imply a roughly 258 MHz reference. That
conclusion was wrong: the initialization sequence wrote `CTRL.SOFT_RESET` and
then lost the following divider write, so the intended divisor was never
latched.

After fixing the reset sequence, the documented power-of-two divisors 2 through
256 read back correctly and produced monotonically longer transfer times. This
confirms the divider behavior, but the absolute SPI0 source frequency is still
unknown.

## Clock helpers

`ag32_sysctl.h` exposes the readable parts of the clock tree:

| Helper | Meaning |
|---|---|
| `ag32_sysclk_source()` | live HSI/HSE/PLL/EXT source select |
| `ag32_clk_hse_ready()` | HSE ready flag |
| `ag32_clk_pll_ready()` | PLL ready flag |
| `ag32_pbus_divider()` | live PBUS divisor |
| `ag32_mtime_divider()` | live MTIME divisor |
| `ag32_uart_ref_hz_measured()` | current L48 UART0 bench measurement (~14.47 MHz) |
| `ag32_sysclk_hz(&sources)` | resolve source select using caller-supplied source frequencies |
| `ag32_pbus_hz_actual(&sources)` | apply the live PBUS divider to that resolved source |
| `ag32_pbus_hz(sysclk_hz)` | legacy helper: divide a caller-supplied SYSCLK |

The chip can tell software which source and divider are selected. It cannot tell
software the absolute frequency of an external crystal or untrimmed RC source,
so `ag32_clk_sources_t` supplies those values. Missing values return 0 rather
than inventing a frequency.

`AG32_MTIME_HZ_MEASURED` and `AG32_UART_REF_HZ_MEASURED` are bench measurements,
not datasheet constants. The old SPI reset-clock constants are retained only for
historical compatibility and are not current calibration values.

Some examples such as `i2c_probe.c` and `can_selftest.c` currently borrow the
UART measurement as an explicit assumption until those peripheral references
are measured independently.

## One FCB side effect

`ag32_fcb_config()` clears the MCU clock source-select/HSE/PLL enable bits before
streaming a fabric image (historically `CLK_CNTL &= ~0x27`), and the SDK does
not switch them back afterward.

This is another reason application firmware should inspect the live clock state
instead of assuming a datasheet maximum or a previous bootloader setting.

## Runtime clock switching

The vendor manual describes the basic safe order:

1. start from HSI;
2. enable the desired HSE/PLL source;
3. wait for its ready flag;
4. set safe flash dividers before increasing SYSCLK;
5. switch SYSCLK only after the new source is ready;
6. never disable the currently selected source;
7. switch back to HSI before disabling HSE/PLL.

The register descriptions expose the source-select, ready bits and dividers, but
the open SDK has not yet qualified a complete transition sequence on hardware.
There is therefore no `set_sysclk()`-style API yet.

Before adding one, useful tests include:

- register snapshots before/after transitions;
- measured HSI/HSE/PLL frequencies;
- flash execution while switching up/down;
- UART/timer/PBUS measurements at each point;
- USB operation at its required clock;
- timeout/fallback behavior if a source never becomes ready.

## Fabric clock

The fabric clock is a separate system. `--freq` or `[fabric].freq` chooses the
PLL/configuration profile emitted into the fabric bitstream and the nextpnr
timing target.

The current reference-board table has **43 measured HSE=8 points from 4 to
248 MHz**, plus two byte-exact profiles using 12/16 MHz HSE inputs that have not
been exercised on the 8 MHz board.

These measurements show the requested PLL output rate, but not perfect clock
reach to every placement. `VP-AGM-007` is a counterexample: a five-region state
design routed and simulated correctly but stayed at zero on the chip. Clock
regions, seams, skew and far-site distribution still need work.

See [STATUS.md](STATUS.md) for the current accepted fabric profiles.

## External-AHB bus clock

The MCU/fabric External-AHB boundary has its own `bus_clock` signal. In the
tested default topology it aliases `sys_gck`.

A direct-D counter/LFSR experiment correlated fabric state against MTIME over
three runs and found exactly **one state transition per MTIME tick**.

Older documentation called this a “10 MHz bus clock” by assuming MTIME was at
the vendor-nominal 10 MHz HSI value. Since MTIME later measured 14.08 MHz in a
similar SRAM-loaded setup, that absolute label is not reliable.

The result to keep is the measured **1:1 bus-clock/MTIME ratio**. The absolute
frequency needs another run that records `CLK_CNTL`, `PBUS_DIVIDER` and
`MTIME_PSC` at the same time.

A separate GPIO-fed synchronous-reset test also works for the retained AHB
state examples. Hard `MCU_RESETN`, unrestricted direct-D placement and broader
clock routing remain open.

Primary source: [AG32 MCU Reference Manual, 2025-05-15 revision](https://www.agm-micro.com/upload/userfiles/files/AG32%20MCU%20Reference%20Manual%2820250515%E4%BF%AE%E8%AE%A2%E7%89%88%EF%BC%89.pdf).