# The old vendor canvas: `fabric_default.bin`

`fabric_default.bin` used to be the starting point for every generated fabric
image. It is **not** the default anymore.

Since 2026-08-14 AGaMEMnon generates the design-neutral base image from public
data in `default_frame.py`. The generated preamble/body matches the decoded
vendor canvas byte-for-byte, then bitgen writes a fresh CRC.

The old file is still useful as:

- a decode/differential reference;
- a regression oracle for the from-scratch generator;
- an optional historical baseline selected with `AGAMEMNON_BASELINE`.

It is **not directly loadable** because its stored CRC is stale. The L48 FCB
rejects the untouched file with `STAT_ERR_CRC`.

## Current state in one table

| Area | Current status |
|---|---|
| 164-byte preamble | generated from public constants/profiles |
| 99,768-byte design-neutral body | generated from public tables, byte-identical to decoded canvas |
| CRC | regenerated every build |
| LUT default state | understood; unconfigured LUT INIT is zero |
| routing/seam reset area | positions/reset values known and generated |
| individual meaning of many selector bits | still incomplete; roughly 74% of those bit-lines remain unnamed |
| 15 `XXXX` spare bits | position/reset value known; function unknown |

The important distinction is that **we can reproduce every base-image byte
without knowing the semantic name of every bit**.

## File format

The shipped reference file is:

```text
agamemnon/chipdb/fabric_default.bin
size    2,839 bytes
sha256  6093e876041bab9f8d1f6058235713a6b8ced1024455070fe2b358e87915a041
```

It contains an eight-byte header followed by the AGM LZW stream:

```text
40 20 00 01   DEVICE_ID = 0x40200001
00 00 ff ff   max dictionary index = 0xffff
...           compressed payload
```

Decompression produces a 99,936-byte raw configuration image. See
[BITSTREAM_FORMAT.md](BITSTREAM_FORMAT.md) for the codec details.

The decoded raw image has SHA-256:

```text
717d6c672b215676ae74279d47835eaab7367e8d05b6cb0d7585727ba581c18f
```

## Raw image layout

```text
0x00000 .. 0x000a3   164 bytes    preamble
0x000a4 .. 0x1865b   99,768 bytes configuration body
0x1865c .. 0x1865f   4 bytes      CRC-32/BZIP2
```

Total: 99,936 bytes / 799,488 configuration bits.

The CRC covers the eight-byte file header plus raw bytes `[0:99932]`.

The canvas stores CRC `0xAD5B5DB9`, but recalculating the CRC over its own
contents gives `0x4B36B054`. That stale CRC is why the untouched canvas is
rejected by the FCB.

## Preamble

The 164-byte preamble is reconstructed rather than copied:

| Offset | Bytes | Region |
|---|---:|---|
| `[0x00:0x21]` | 33 | descriptor / idle records |
| `[0x21:0x40]` | 31 | global setup |
| `[0x40:0x71]` | 49 | clock distribution |
| `[0x71:0x7c]` | 11 | PLL descriptor |
| `[0x7c:0x9a]` | 30 | PLL source chain |
| `[0x9a:0xa4]` | 10 | trailer |

Clock/PLL regions are filled from the selected supported profile.

## The big `0xFF` region

An early theory said the large `0xFF` area in the canvas was unconfigured LUT
INIT stored in complemented form. That was wrong.

Direct comparison with all 33,792 LUT INIT positions showed that unconfigured
LUTs are actually zero in the canvas. Only configured border LUTs are set.

The large `0xFF` region belongs instead to routing/cell-interconnect selector
state: RMUX/IMUX/seam/control families at their all-ones reset polarity.

That region covers **28,570 body bytes**. Its positions and reset values are
represented by:

```text
agamemnon/chipdb/logictile_config_template.csv
```

The table is enough to regenerate the reset image even though many individual
bit-lines still lack semantic names.

## The final partial bytes

After filling the main selector/reset rectangles, 227 edge/partial bytes were
still missing from a byte-exact reconstruction.

Those are now generated from:

```text
agamemnon/chipdb/border_edge_partial_cells.csv
```

The table accounts for:

- 408 asserted bits associated with named LogicTile cells;
- 15 asserted `XXXX` spare bits whose positions are known but functions are not.

Adding those bytes took the generated body from 99.77% to 100% byte-identical to
the decoded canvas.

## How reconstruction improved

| Generator stage | Body byte match |
|---|---:|
| zeros + generated preamble/CRC | 70.33% |
| + named border config and framing | about 71% |
| + routing/seam reset fill | 99.77% |
| + edge/partial-cell fill | **100%** |

No byte from `fabric_default.bin` is copied by the normal generator today.

## Hardware check

The generated base was also tested on the L48 board. Two otherwise identical
99,944-byte SRAM images differed only in the final CRC:

| Image | `FCB_STAT` | Result |
|---|---|---|
| body + stale canvas CRC `0xAD5B5DB9` | `0x00000040` | rejected with `STAT_ERR_CRC` |
| body + recalculated CRC `0x4B36B054` | `0x000f0002` | configured |

The retained pack-regression designs also produce identical final output whether
they start from the generated base or the decoded reference canvas.

That result justified flipping the default to the from-scratch generator on
2026-08-14.

## Bit ownership

Bitgen tracks which stage owns each output bit with `bit_ownership.py`.
Enable the optional trace with:

```text
AGAMEMNON_OWNERSHIP_TRACE=<path>
```

A feature may only write inside its declared masks, and two active features may
not claim the same bit.

For a small four-bit carry-counter build, the ownership breakdown was:

```text
base/default state   544,323 bits   68.08%
cleared defaults     254,629 bits   31.85%
design-owned             536 bits    0.07%
```

The first category now means “left at the generated design-neutral base value,”
not “copied from the vendor file.”

## Build pipeline

The current bit generator roughly does:

```text
default_frame.build()
    -> clear design-dependent fields
    -> routing
    -> MCU edges
    -> logic
    -> clocks
    -> IO
    -> BRAM
    -> carry
    -> regenerate preamble
    -> regenerate CRC
    -> LZW encode
```

`AGAMEMNON_BASELINE=<file>` replaces the generated starting frame for historical
replay/debugging, but the later clear/overlay/preamble/CRC phases still run.

## What remains unknown

The base-image dependency itself is solved. The remaining reverse-engineering
question is semantic:

- many selector bit-lines have a known location and reset value but no name;
- the 15 `XXXX` spare bits still have no known function;
- subsystem-specific configuration around IO, BRAM and other hard blocks is
  only partly understood.

See [CONFIG_SURFACE_MAP.md](CONFIG_SURFACE_MAP.md) for the broader configuration
map and [STATUS.md](STATUS.md) for current build/hardware support.

The reproducible evidence for the base generator is in
`qualification/fabric_base_evidence.jsonl`; the old canvas hash remains pinned
in [NOTICE.md](../NOTICE.md) for as long as the reference file is shipped.