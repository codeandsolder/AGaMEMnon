# AG32 configuration surface

A useful way to organize the AGRV2K bitstream is to split it into three areas:
logic, routing, and everything around the fabric.

This is mainly a reverse-engineering map. For what AGaMEMnon can actually build
and run today, see [STATUS.md](STATUS.md).

## 1. LUT contents

The LUT truth-table bits are understood:

- 33,792 physical LUT INIT positions;
- mapped by `physmap.init_bit_pos`;
- unconfigured default is zero;
- bit positions and polarity are known.

This part is in good shape.

## 2. Routing and cell interconnect

This is the large selector/mux area containing families such as:

- RMUX
- IMUX
- OMUX
- CTRLMUX
- SEAMMUX
- BBMUXS
- IOMUX

The from-scratch base generator knows where these bits live and what their reset
values are. The main `0xFF` region of the old vendor canvas belongs here, not to
LUT INIT as an earlier theory claimed.

`agamemnon/chipdb/logictile_config_template.csv` contains the promoted
LogicTile map. Roughly 26% of the selector bit-lines currently have semantic
names; the remaining ~74% have known positions/reset values but not yet known
individual functions. There are also 15 `XXXX` spare bits whose purpose is
unknown.

So there are two different notions of “decoded” here:

- **byte/position complete:** enough information exists to regenerate the
  design-neutral body exactly;
- **semantic complete:** every individual bit has a known function.

The first is essentially done for the base image. The second is not.

## 3. Clocks, IO, BRAM and hard-block interfaces

Everything that is not ordinary LUT/routing configuration lands in this bucket:

- clock and PLL setup;
- IO electrical settings and OE;
- BRAM modes and ports;
- MCU/fabric boundary configuration;
- other hard-block interfaces.

Coverage varies a lot by subsystem. Some PLL fields are well understood; BRAM
and physical input behavior still have major holes.

## Current map

| Surface | State |
|---|---|
| LUT INIT | Positions and function decoded |
| Routing/cell interconnect | Positions/reset values regenerated; many individual selector functions still unnamed |
| Clock/PLL | Useful subset decoded and generated |
| IO electrical/OE/banks | Partial |
| BRAM modes/ports | Partial |
| MCU/hard-block interfaces | Partial |
| Hard MMIO peripherals | Partial firmware/register coverage |

The old “vendor canvas dependency” problem is solved: AGaMEMnon can generate the
design-neutral base image from public data. The interesting remaining work is
understanding what all of the still-unnamed fields do and correctly configuring
them for real designs.

More detail on the base image is in
[FABRIC_DEFAULT_CANVAS.md](FABRIC_DEFAULT_CANVAS.md).