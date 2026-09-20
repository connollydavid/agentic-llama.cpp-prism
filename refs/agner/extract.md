# Agner Fog extract: the rows this project's substrates cite

Source manuals, pinned by SHA256 in `agner.sums` (downloaded 2026-09-20 from
agner.org/optimize; CC BY-NC-ND 3.0 DK — cited, never vendored beyond these
working copies, which are gitignored):

- `microarchitecture.pdf` = micro.txt line references below
- `instruction_tables.pdf` = instr.txt line references below

The text layer of the pinned PDFs drops some Intel-vector rows from the
per-microarchitecture instruction lists; where a row could not be lifted
mechanically the extract says so and names the derivation instead.

## Pipeline axes per microarchitecture (drives each `Substrate`)

| µarch | ROB | RS | physical vector regs | issue/rename width | micro.txt |
|---|---|---|---|---|---|
| Sandy Bridge | 168 | 54 | 144 | 4 µops/cycle | 6989-6992 |
| Ivy Bridge | 168 (as SNB) | 54 | 144 | 4 | 6990 ("as Sandy Bridge" chapter basis) |
| Haswell | 192 | 60 | 168 | 4 | 6989-6992, 7858-7860 |
| Broadwell | 192 (as HSW) | 60 | 168 | 4 | 7858 context (BDW chapter reuses HSW resources) |
| Skylake (client; Comet Lake is this core) | 224 | 97 | 168 | 4 | 8509-8512 |
| Ice Lake | 352 | 160 | 224 | 5 (µop-fused, front end 6-wide decode) | 9140-9143 |
| Zen 1 | 192 µops in flight | schedulers | 168 (AGU file) | 6 dispatch | 13115 |
| Zen 2 | 224 µops in flight | schedulers | 168 | 6 dispatch | 13115 |
| Zen 3 | 256 | schedulers | 192 | 6 dispatch | 13659 |
| Zen 4 | 320 | schedulers | 224 | 8 int + 6 FP dispatch | 14014 |

## SIMD capability classes (drives the kernel ladder and feature detection)

| µarch | 256-bit integer (AVX2) | 256-bit FMA | VNNI | maddubs 256-bit |
|---|---|---|---|---|
| Sandy/Ivy Bridge | no | no (separate mul + add) | no | n/a (128-bit only) |
| Haswell/Broadwell/Skylake/Comet Lake | yes | yes, 2 per cycle, ports 0+1 | no | 2 vector ALUs; derived 0.5 c reciprocal on the integer multiply-add class (instr.txt Intel rows for this mnemonic do not lift; ports per micro.txt 7838-7851, 8100, 8423) |
| Zen 1 | yes (cracked to 2×128) | 1 per cycle effective | no | VPMADDUBSW y,y,r/m: 2 µops, 4 c latency, 2 c reciprocal, P0 (instr.txt 5311) |
| Zen 2 | yes native | 2 per cycle | no | as Zen 1 rates (not liftable from the pinned text; derived) |
| Zen 3 | yes native | 2 per cycle | no | derived from Zen 2 rates |
| Zen 4 / Alder Lake+ | yes | 2 per cycle | AVX-VNNI (dpbusd) | dpbusd replaces maddubs; not needed by this port |

## Instruction rows lifted verbatim (Zen 1 section, instr.txt)

```
VPMADDWD    y,y,r/m   2   3   2   P0     (instr.txt 5309)
VPMADDUBSW  y,y,r/m   2   4   2   P0     (instr.txt 5311)
VPSHUFB     y,r/m     2   1   1   P12    (instr.txt 5218)
```

## Permutes (staging-order cost for the PTQ1_0 non-positional layout)

```
VPERMD y,y,y  1 µop  1 c  p5   (Haswell, instr.txt in 14717-15614; Skylake, in 16522-18599)
VPERMD y,y,m  1 µop  2 c  p5 p23
```

## Memory subsystem numbers used by the model (datasheet-derived, not Agner)

| platform | theoretical peak DRAM | note |
|---|---|---|
| i7-2600K, DDR3-1333 dual-channel | 21.3 GB/s | Intel Desktop 2nd Gen datasheet |
| i7-4670K, DDR3-1600 dual-channel | 25.6 GB/s | Intel Desktop 4th Gen datasheet |
| i9-10885H, DDR4-2933 dual-channel | 46.9 GB/s | Intel 10th Gen H datasheet |
| Xeon W-2140B (local), DDR4-2666 quad-channel | 85.3 GB/s | measured STREAM share applied in the sweep |
| TU117-class Turing, 128-bit GDDR5/GDDR6 | 96-192 GB/s by SKU | NVIDIA TU117 datasheet class |
| Quadro RTX 6000 (local, TU102), 384-bit GDDR6 | 672 GB/s peak | calx-mill telemetry fixture cites 609 GB/s measured (tests/fixtures) |

Effective (STREAM-class) fractions are applied in the sweep driver, not in
this table; the calx-mill TU102 table's measured 5.82 B/clk/SM DRAM budget is
the anchor method (its fixtures README documents the rig: T5820 + 2x RTX
6000, the same GPU as this machine).
