# Analog/fabric boundary

The analog blocks themselves have been exercised through a vendor `analog_ip`
wrapper, and the MCU-side register access works. AGaMEMnon does **not** yet emit
that analog macro itself.

So there are two separate pieces of work here:

1. MCU firmware can talk to ADC/DAC/comparator registers through a configured
   analog wrapper on L48.
2. The open FPGA flow currently exposes only a few read-only ADC-to-fabric
   routes.

## What has worked on hardware

The analog register window starts at `0x60000000`:

| Block | Offset |
|---|---:|
| ADC0 | `+0x0000` |
| ADC1 | `+0x1000` |
| ADC2 | `+0x2000` |
| DAC0 | `+0x3000` |
| DAC1 | `+0x4000` |
| CMP0 | `+0x5000` |

These blocks only appeared after loading a fabric image containing the vendor
analog wrapper.

On the L48 reference board the following worked:

| Block/path | Result |
|---|---|
| ADC0, ADC1, ADC2 | 12-bit one-shot conversion followed a DAC stimulus |
| DAC0, DAC1 | 10-bit output verified through ADC readback |
| CMP0 unit 1 | switched at the four internal VREF settings at the expected DAC codes |
| DAC0 -> ADC channel 4 | internal loopback worked on all three ADCs |
| DAC1 -> ADC channel 5 | internal loopback worked |
| MCU -> analog register window | register reads/writes worked from open firmware |

One DAC0 sweep produced ADC0 channel-4 values:

```text
DAC:   0  128  256  384  512  640  768  896  1023
ADC:   0  512 1024 1536 2054 2575 3085 3598 4095
```

Another run differed by a few ADC counts, as expected from an actual converter.
The useful result is that the response was monotonic, close to the expected 4:1
12-bit/10-bit slope, and saturated near full scale. The exact low bits are not
stable test constants.

CMP0 unit 1 flipped at DAC codes 94, 188, 281 and 373 for VREF/4, VREF/2,
3*VREF/4 and VREF. The vendor RTL predicts roughly 93, 186, 279 and 372.

The firmware and drivers are:

- `agamemnon/sdk/include/ag32_adc.h`
- `agamemnon/sdk/include/ag32_dac.h`
- `agamemnon/sdk/include/ag32_comparator.h`
- `examples/riscv_mcu/analog_probe.c`

## Things that did not work or are still unknown

### Comparator unit 2

Unit 2 can be enabled and its registers read back, but its output stayed high
for the full DAC sweep under both tested PSEL2 choices. Its input mux is not
understood yet.

### External ADC channels 0-3

These read `0xfff` in the current setup. An earlier explanation said those pads
were not bonded on L48; that was wrong. The pin data places `ADC_IN0..IN3` on
`PIN_10..PIN_13`, and those package pins definitely exist and work as digital
IO.

Plausible causes include:

- the analog mux was not enabled;
- the pads remained configured for digital IO;
- the board does not connect the expected analog signal;
- some reference/bias setup is missing.

For now the external channels are simply untested as useful analog inputs.

### Other analog modes

Not tested yet:

- comparator hysteresis and other comparator modes;
- ADC/DAC DMA;
- continuous scan;
- longer sequencer programs;
- broad external-channel behavior.

## Open-flow ADC routes

The strict open routing graph currently exposes three read-only ADC0 outputs:

- `AGRV2K_ADC0_DB0`
- `AGRV2K_ADC0_DB1`
- `AGRV2K_ADC0_EOC`

The vendor route data gives DB0, DB1 and EOC distinct raw source indices even
though its textual names are lossy. AGaMEMnon therefore gives each one a
private synthetic first-exit wire before joining the normal fabric graph.

The current smoke routes are small:

- DB0: 7 PIPs, 5 configurable fields;
- DB1: 7 PIPs, 5 configurable fields;
- EOC: 8 PIPs, 6 configurable fields.

Their route evidence is recorded in the corresponding
`qualification/analog_adc0_*_route_evidence.jsonl` files.

These routes only move hard-block output signals into the fabric. The open flow
does not yet instantiate/configure the ADC macro, start conversions, or manage
ownership between the MCU and fabric.

The other ten ADC0 data lanes and the fabric-to-ADC control paths are still
missing from the public route graph.