# plan/0007, Turing behavior with the GTX 1650's limits in view

## Why

The 4 GiB target class sharpens to a GTX 1650: TU117 with 14 SMs, 128-bit
GDDR5 at 128 GB/s, no tensor cores, a 75 W power envelope. The measured
Turing anchor (RTX 6000, 72 SMs) says PTQ1_0 batch-1 decode runs about
2.5 GB/s per SM (180 GB/s card-level against 609 DRAM), so the 1650's GPU
leg projects to roughly 35 GB/s effective against its own 128 GB/s: the
decode kernel, not memory, is the wall. Raising per-SM decode to about
9.1 GB/s (128/14) would saturate the card's bandwidth; the RTX 6000 shares
the sm_75 kernel paths and serves as the proxy bench.

## Scope

- calx-mill: a GTX 1650 GPU entry with the exact axes (14 SMs, 128 GB/s,
  decode rates scaled from the measured anchor) and a re-run of the split
  sweep for both packings; the packing verdict may flip when the GPU leg
  is instruction-bound.
- The decode curve on the anchor rig: tg128 at ngl 64, 48, 37, 32, 24, 16,
  8 to separate per-byte instruction cost from per-layer overhead (the
  plan/0004 puzzle: ngl 32 beat ngl 37).
- The CUDA PTQ1_0 MMVQ decode kernel (vecdotq.cuh): reduce the per-block
  instruction count of the base-3 trit decode; verify with the CUDA suites
  and measure tg128 full-offload before and after.
- Out of scope: CPU-side work (plan/0002 and plan/0003 carried it), the
  AVX-only stretch (plan/0006).

## Verification

The CUDA suites green on both ternary types (test-quantize-fns,
test-backend-ops with CUDA); the anchor-rig tg128 full-offload number moves
measurably above the 31.84 tok/s baseline if the kernel work lands; the
sweep table for the 1650 lands here with the packing and split it
recommends, and the projection for the 1650 is stated against its 128 GB/s.

## Build sequence

### model-1650-axes {#model-1650-axes}

Exact 1650 axes in the sweep; re-run both packings; record the
recommendation.

- verify: the sweep prints a 1650 row and the packing comparison; the
  projected GPU-leg rate is compared against 128 GB/s
- depends: none

### decode-curve {#decode-curve}

tg128 across ngl values on the anchor rig with repack enabled; fit the
per-byte and per-layer costs.

- verify: the curve table lands in this README with command lines
- depends: none

### cuda-decode-kernel {#cuda-decode-kernel}

Instruction-count reduction in the PTQ1_0 vec_dot multi kernel; suites and
before/after tg128 on the anchor rig.

- verify: suites green; the before/after number lands here
- depends: none
