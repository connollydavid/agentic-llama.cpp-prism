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

## Results (2026-09-21, anchor rig; avx2-port through 2379e0d, calx-mill through 15ab018)

The nsys profile settles where the decode time lives: for the full-offload
31.4 ms token, all GPU kernels together account for about 1.6 ms. The GPU is
idle roughly 95 percent of decode; CUDA graph replay is already active
(USE_GRAPHS=1, warmup completes), so the cost is the host building,
scheduling, and walking a 2242-node decode graph every token (census: 573
reshape + 355 view + 49 permute + 25 transpose layout nodes, 366 matmuls,
235 sign multiplies, 102 rms norms, 100 cpy, 100 get_rows).

Answers this yields, in the order the questions were asked:

Fusion: not required for Turing kernel efficiency (kernels are 5 percent of
the token), but graph-level fusion is the right lever against the host-side
95 percent, because it removes nodes rather than kernels. The fork already
fuses where it pays on the device (fwht_signed, gate_up MMVQ, GDN snapshot);
the sign-multiply fold below is the same medicine at graph level.

How much of the 30 ms can move into kernels: nearly all of the real work
already runs on the device; what must shrink is the node count. Landed:
the Hadamard sign vector now rides in the mul_mat hint params and the
builders emit it folded (235 nodes removed per token, plus their reshape
traffic); measured full-offload tg128 31.84 -> 33.11 tok/s and perplexity
identical on both backends. The remaining ~900 layout nodes (permute chains
and reshape pairs around each transform) are the next fold, and beyond that
the graph rebuild itself is upstream-scheduler scope.

Thread count: t8 (physical cores) beats t16 by 24 percent on the mixed split
(2.08 vs 1.68 tok/s at ngl 37); hyperthreads only added barrier overhead.

Decode curve (t16, repack on): ngl 16/32/37/48 = 1.01/1.46/1.68/2.32 tok/s,
monotone; the earlier ngl-32-beats-37 reading was day-to-day noise.

Bifurcation, both Turing classes, one file each:

| class | representation | measured / projected |
|---|---|---|
| big Turing (TU102-class, VRAM-rich) | PQ2_0 (slot layout) file | tg128 41.90 tok/s full offload (against 33.11 with the fold on PTQ1_0); the dequant-to-F16 cuBLAS route loses (per-call dequant adds 26 GB of traffic) |
| small Turing (GTX 1650: 14 SMs, 128 GB/s, 4 GiB) | PTQ1_0 dense, repack on | occupancy-corrected sweep: 9.89 tok/s ceiling at ngl 37 with the i9-10885H; the 1650 runs the same warps at ~39 per SM where the 72-SM card starves at 2.5 GB/s per SM |

The occupancy correction matters: decode-effective bandwidth is
warp-count-limited (ne01/32 warps per matmul), not SM-proportional, so the
1650's GPU leg runs near its 128 GB/s while the big card idles at 30 percent
of DRAM. The mixed-split reality on the anchor rig (2.08 tok/s at ngl 37,
t8, fold and repack on) remains CPU-leg-bound at about 2.4 GB/s effective
per-op overhead, the same gap plan/0004 recorded; the node-count work above
is what moves it.
