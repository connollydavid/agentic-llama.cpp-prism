# plan/0003, AVX2 weight repack for the ternary types

## Why

Batch-1 decode on the CPU backend goes through the repack path (interleaved
4-block layouts consumed by gemv/gemm against Q8_0 activations), not the
plain vec_dot. Today PQ2_0 has a repack layout with kernels only under
AVX512-VNNI, and PTQ1_0 has no repack at all. Without an AVX2 repack, the
CPU leg of the 4 GiB split runs the plan/0002 vec_dot without the
layout the fast decode path expects, leaving the CPU leg slower than the
bandwidth arithmetic of plan/0001 predicts.

## Scope

- `block_ptq1_0x4` and the repack builder in `ggml/src/ggml-cpu/repack.cpp`,
  following the PQ2_0 pattern, plus the generic gemv/gemm (4x8 against Q8_0).
- AVX2 specializations in `ggml/src/ggml-cpu/arch/x86/repack.cpp` for the
  PTQ1_0 layout and an AVX2 variant of the PQ2_0 kernels (the existing ones
  are AVX512-VNNI only): maddubs/madd accumulation instead of dpbusd.
- Dispatch entries so the repack path activates for the ternary types on
  AVX2-class machines, with the AVX512 path unchanged where it exists.
- Out of scope: changes to the repack framework itself or other quant types.

## Results (2026-09-20)

Landed on `avx2-port` ("cpu: AVX2 weight repack for PTQ1_0 (shared layout
with PQ2_0)"). `block_ptq1_0x4` stores the decoded ternary codes as 2-bit
slots in the `block_pq2_0x4` layout (a lossless conversion), so the repack
pays the base-3 decode once at load and the steady-state gemv is shared by
both types. Three integration defects were found and fixed on the real model,
each invisible to the synthetic suites: the repack buffer was sized from the
source type (21 percent too small), the Hadamard rotation matrices that the
runtime places beside the weights crashed the loader (stored verbatim now,
with generic-path approval for ops whose sources carry no repack traits),
and the first AVX2 gemm cut ran prefill below the scalar baseline (qword
gathers and 16-accumulator pressure), so the batched gemm stays on the
generic path and an optimized gemm is owed work.

Measured, CPU-only, 16 threads (`-ngl 0 -p 512 -n 32 -t 16`, Xeon W-2140B:

| | before (scalar) | after (AVX2 + repack) |
|---|---|---|
| tg32 | 0.13 tok/s | 1.81 tok/s (13.9x) |
| pp512 | 174.58 tok/s | generic gemm, number pending the rerun |

The plan/0001 divergence matrix predicted a 1.33x repack gain on this CPU
class at the model level; the measured full-pipeline gain (13.9x against the
scalar baseline, which pays the decode every token with no SIMD at all)
reflects how far the starting point was below the model's vec_dot class.

## Verification

`test-backend-ops` green on the CPU backend for both ternary types with the
repack path forced on; `llama-bench -ngl 0` before/after for the decode
number; the measured gain compared against the plan/0001 projection for this
machine's SIMD class.

## Build sequence

### ptq1-0-repack-layout {#ptq1-0-repack-layout}

Define the interleaved block type, the builder, and the generic kernels;
wire the repack dispatch for PTQ1_0.

- verify: `test-backend-ops -b CPU` green with PTQ1_0 repack active
- depends: none

### avx2-repack-kernels {#avx2-repack-kernels}

AVX2 gemv/gemm specializations for PTQ1_0 and the AVX2 variant of the PQ2_0
kernels; feature-guarded so AVX512-VNNI machines keep their existing path.

- verify: `test-backend-ops -b CPU` green; `llama-bench -ngl 0` decode
  improves over the plan/0002 after-number on this machine
- depends: #ptq1-0-repack-layout

### record-decode-gain {#record-decode-gain}

File the before/after decode table with command lines here and against the
plan/0001 prediction.

- verify: the table names binary, model, and command line for every number
- depends: #avx2-repack-kernels
