# Current status

AGaMEMnon works for a growing but still fairly narrow set of designs on the
AG32VF303CCT6/LQFP-48. Small logic, several IO routes, a useful slice of the
MCU/fabric AHB interface, some BRAM/carry cases, and several hard peripherals
have been demonstrated on hardware. Wider designs still often fail to route,
and some images that look completely fine in software are wrong on silicon.

This page is the current list of what works, what is broken, and what is only
partly understood. The generated [FPGA parity ledger](FPGA_PARITY_LEDGER.md)
and [claim-policy ledger](CLAIM_POLICY_LEDGER.md) contain the machine-readable
detail.

## Test state

Current full test run:

```text
1457 passed, 49 skipped, 0 failed
```

The reviewed `public32` composition still reproduces its existing route and
image hashes. If that hash changes after a chip-database edit, inspect the
actual route/configuration change before updating it; the procedure is in
[LANDING_A_CHIPDB_CHANGE.md](LANDING_A_CHIPDB_CHANGE.md).

The hardware campaign found 13 AGaMEMnon images that built cleanly but failed
their tests on silicon. The exact known bad images are blocked by hash, and
seven recurring bad logical graphs are also blocked before routing. All 13
recorded failures are covered by those guards; their underlying causes are
still open.

## 105-design vendor/open campaign

| Result | Designs |
|---|---:|
| Vendor and AGaMEMnon both matched the test | 25 |
| Vendor reference failed | 10 |
| Vendor result was unstable | 2 |
| AGaMEMnon did not route | 52 |
| AGaMEMnon built but failed on hardware | 13 |
| Test harness incomplete | 3 |
| **Total** | **105** |

Six of 51 paired user/structural forms worked in both forms: SPI0 TX, SPI1 TX,
I²C0, I²C1, UART1 TX, and UART2 TX. These were hand-written development tests,
not a random or hidden benchmark set. See [Vendor parity](VENDOR_PARITY.md) for
the individual experiments.

## Build flow

```text
Verilog
  -> Yosys technology mapping
  -> generated AGRV2K device database
  -> nextpnr
  -> AGaMEMnon bitgen
  -> SRAM image + compressed flash image
```

`agamemnon build --uarch` uses the normal tiered routing graph. It may use mux
encodings known from other physical positions and records those routes in
`<output>.confidence.json`. `--release-strict` only allows routes with a direct
witness at that position. See [Routing admission](ROUTING_ADMISSION.md) if you
care about that distinction.

## Fabric and MCU/fabric support

| Surface | What currently works | Known gaps / failures |
|---|---|---|
| LUT4 / local logic | Several small Boolean, shift, add/subtract, fanout and handshake designs | FSM, rotate, feedback and dense-state tests have known failures; arbitrary RTL is not there yet |
| Flip-flops / state | Small counters, LFSRs, selected direct-D placements, retained known-good designs | Reset/update failures and a five-region state failure; generic state placement is not reliable yet |
| General routing | Large recovered routing/selector tables; many small and medium routes | 52 campaign designs did not route; some topology and special feeders are still missing |
| Dedicated carry | Short same-tile chains, one 33-site X20 corridor, one inter-tile seam | Other columns/seams, branching and large carry designs |
| External AHB slave | 32-bit constant endpoint, byte/16-bit banks, local-interrupt commands, reviewed `public32` map | Fresh wider banks, broad burst behavior, hard reset, alternate bus clocks, arbitrary placement, AHB master/DMA |
| Fabric local interrupts | One four-cause composition, causes 16–19, with mask/ack/set and synchronous reset | Generic pending banks, hard reset, alternate clocks, asynchronous sources |
| Physical outputs | Several exact L48 top/left-edge routes; campaign output tests on PIN_12/PIN_16 | Arbitrary routes, other packages, broad electrical modes |
| Physical inputs | Several old exact L48 input routes work | New PIN_10/PIN_12 held-input tests stayed low (`VP-AGM-008`); generic ingress is not reliable |
| Bidirectional / OE | PIN_25–PIN_28 OE corridors and exact I²C0/I²C1 open-drain routes | General direction switching, broad simultaneous readback, electrical/PVT characterization |
| BRAM | Retained X13Y4 read/write routes and several exact profiles | New initialized x1/x18 Port-A reads returned zero (`VP-AGM-006`); the affected profiles are blocked |
| PLL frequency | With 8 MHz HSE, 43 requested SYSCLK rates from 4–248 MHz measured and locked | Phase/duty/other outputs, other HSEs, arbitrary distribution; two 12/16 MHz-HSE profiles are byte-exact but untested on the 8 MHz board |
| Clock reach / regions | A local matched PLL/shift test works | One five-region registered design stayed at zero despite a correct routed model (`VP-AGM-007`) |
| Timing model | Useful conservative local estimates | No complete clock-skew/IO/BRAM/PLL/package/PVT model; not sign-off quality |
| Packages | L48 bond map is cross-checked and is the main hardware target; maps exist for L64/L100/Q32 | Current docs disagree about exactly how far non-L48 pad-free builds are allowed; see [Documentation issues](DOCUMENTATION_ISSUES.md) |
| Analog boundary | Some MCU ADC/DAC/comparator register and loopback work | Public bitgen does not emit the vendor analog macro; external analog modes are largely unexplored |
| Programming | DAP SRAM loads, L48 flash backup/program/verify, installed USB CDC uploader | Pico/UART ROM software exists, but the current five-wire target setup still needs its final bench run |

### A couple of old results that need context

The old claim that HRESP reliably becomes an MCU load/store access fault turned
out to be wrong. The response signal was active, but the MCU did not report the
expected trap because the response phase slipped into the following transfer.
HRESP is therefore not used as a deterministic exception mechanism on this
setup.

The former 14-entry "dead edge" routing list was also wrong as an intrinsic
edge classification. All 14 edges conduct in isolated tests. The original
congested design still failed, so the problem appears to be composition/context
rather than fourteen physically dead wires.

For IO, keep two odd results in mind:

- one retained PIN_12 inversion route works, while newer held-input PIN_12 and
  PIN_10 designs fail; do not assume all input routes behave the same;
- PIN_25 OE has working local and externally controlled examples, but other
  OE/readback routes still need characterization.

## Hard peripherals

Hard peripherals live on the MCU side, but getting their signals to pins still
depends on the fabric routes.

| Peripheral | Working examples | Missing / broken |
|---|---|---|
| UART | UART0 internal loopback; retained UART0 PIN_30/PIN_31 duplex; campaign UART0/1/2 TX on PIN_10 at 9,600/38,400/115,200 baud | UART3/4 TX; campaign RX for UART0–4; broader framing, flow control, interrupt/DMA, other pads/packages, clock/PVT work |
| SPI | SPI0/SPI1 TX, mode 3, MSB first, active-low CS, 1–4-byte transfers on exact L48 routes | New typed SPI0/SPI1 MISO routes return `0xffffffff` (`VP-AGM-008`) and are blocked; generic RX/duplex, other modes, DMA, other pads/packages |
| I²C | I²C0/I²C1 transaction to address `0x55`: write `2A A6`, repeated START, read `5A C3 7E`; I²C0 also has a 500 µs stretch test | 10-bit addressing, multimaster/arbitration, arbitrary lengths, broader stretching, DMA/interrupts, electrical margins |
| CRC | CRC-32/MPEG-2 known-answer test for `123456789` | Other modes |
| DMA | One DMAC0 four-word SRAM copy | Peripheral-linked/chained/broader DMA |
| Watchdog | Disabled-state snapshot and supervised warm reset | Broader modes |
| CLINT timer | One machine-timer interrupt with `mcause=0x80000007` | Full timing/interrupt behavior |
| RTC | Configuration/readback | Timekeeping; no tested low-speed clock yet |
| CAN | Register/config/transmit-state observations | No transceiver-backed protocol test yet |
| USB | Flash-resident CDC device uploader | Host/OTG and ROM USB recovery |
| Ethernet | Register-level work | MAC/PHY operation; no PHY fixture yet |

## Open hardware failures

| ID | Area | What happened |
|---|---|---|
| `VP-AGM-001` | MCU ALU feedback | Software models agree, hardware diverges in a read-data feedback design |
| `VP-AGM-003` | FSM update | Vendor follows the model; AGaMEMnon hardware loses one next-state bit at step 16 |
| `VP-AGM-004` | Rotate/reset | Correcting the known selector did not fix a four-bit rotate design |
| `VP-AGM-005` | One-bit add/sub reset | Ordinary and explicit-carry versions start with the same wrong reset state |
| `VP-AGM-006` | BRAM read | Initialized x1 and x18 designs read zero; affected profiles are blocked |
| `VP-AGM-007` | Clock/state reach | Five far-apart registers remain zero although routed simulation is correct |
| `VP-AGM-008` | Physical input / SPI MISO | PIN_10/PIN_12 inputs stay low; SPI0/SPI1 MISO stays high in the tested designs |
| `VP-AGM-009` | 256-bit state | Routes on attempt 13, matches simulation, then diverges on hardware at transaction 2 |

`VP-AGM-002` (UART0 TX/PIN_10 selector) and `VP-AGM-010` (SPI TX lane packing)
were fixed for the tested cases.

## Routability and wider designs

Routing is currently the biggest practical limitation: 52 of the 105 campaign
designs produced no usable image. Some equivalent user/structural rewrites have
very different placement/routing success.

Current wide-design landmarks:

- the old X13Y12 ingress problem has working solutions;
- fresh `regbank16` still does not produce an image;
- `addsub16` reaches the density policy but exposes placement differences;
- a 256-bit state design needed 13 attempts to route and was then wrong on
  hardware; its structural version did not route.

The work here is better placement/routing and figuring out the hardware failure
modes, not pinning one-off routes until benchmarks pass.

## SERV

`serv-blinky` is a retained known-good route and is useful as a large integration
example. Fresh arbitrary SERV placement/routing has not been demonstrated, and
the retained image should not be read as general RV32I, BRAM, or CPU-scale
support.

For the investigation record, see
[AF_EXE_REVERSE_ENGINEERING.md](AF_EXE_REVERSE_ENGINEERING.md),
[CONDUCTION_REFRAME_STATUS.md](CONDUCTION_REFRAME_STATUS.md), and
[HARDWARE_VALIDATION.md](HARDWARE_VALIDATION.md). For unfinished work, see the
[top-level roadmap](../ROADMAP.md).
