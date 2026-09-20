# plan/0002, AVX2 vec_dot kernels for the ternary types

## Why

The fork's CPU backend runs PTQ1_0 through a scalar generic vec_dot on every
architecture (the arch-fallback comments say so outright), and PQ2_0's only
x86 SIMD path requires AVX-VNNI or AVX512-VNNI. The primary target CPU
(i9-10885H, Comet Lake) has plain AVX2+FMA and nothing newer, so both ternary
types decode at reference speed on the exact machine the port is for. The
same kernels serve every pre-VNNI part: Haswell through Comet Lake on the
Intel side, Zen 1 through Zen 3 on the AMD side.

## Scope

- `ggml_vec_dot_ptq1_0_q8_0` for AVX2+FMA in `ggml/src/ggml-cpu/arch/x86/quants.c`:
  vectorized base-3 trit decode mirroring the CUDA `ptq1_0_trit` mapping,
  the non-positional 24-byte/2-byte staging order respected, maddubs/madd
  accumulation (no dpbusd without VNNI), per-block Q8 sum subtraction, FP16
  group scale.
- A plain-AVX2 branch for `ggml_vec_dot_pq2_0_q8_0` beside the existing VNNI
  path, mirroring the upstream Q2_0 idiom.
- Remove the x86 `arch-fallback.h` aliases for the ternary vec_dots; declare
  the new symbols in `quants.h`.
- Extend the element-map test to cover the AVX2 decode order against the CPU
  codec.
- Out of scope: the weight-repack path (plan/0003), AVX-only Sandy/Ivy
  kernels (plan/0006).

## Results so far (2026-09-20)

Both kernels landed on `avx2-port` (commit "cpu: AVX2 vec_dot kernels for
PTQ1_0 and PQ2_0"): the PTQ1_0 AVX2 decode follows the reference staging
exactly (16/8/2-element groups packed per q8_0 sub-block through
packus/permute 0xD8), and the PQ2_0 plain-AVX2 branch mirrors the VNNI path
with saturation-safe maddubs. The x86 arch-fallback alias retired.
`test-quantize-fns` green on both types in the WSL2 CPU build (this machine,
AVX2+AVX512 with no VNNI class, executes exactly the new branches).
`test-backend-ops -b CPU` and the before/after bench: running, numbers land
here when they complete.

## Verification

The fork's own suites gate correctness (call/0002): `test-quantize-fns` and
`test-backend-ops` green on the CPU backend in the WSL2 build; the
element-map test green. Speed is a plan claim measured by `llama-bench
-ngl 0` before and after on this machine, with the command lines recorded
here, and checked against the plan/0001 divergence matrix.

## Build sequence

### avx2-ptq1-0-vec-dot {#avx2-ptq1-0-vec-dot}

Implement and wire the PTQ1_0 AVX2 kernel; keep the generic scalar path as
the fallback for non-AVX2 x86.

- verify: `./build/bin/test-backend-ops -b CPU` and `test-quantize-fns` green;
  the element-map test extended and green
- depends: none

### avx2-pq2-0-branch {#avx2-pq2-0-branch}

Add the AVX2 (no-VNNI) branch to the PQ2_0 vec_dot under the existing
feature guards.

- verify: `test-backend-ops -b CPU` green on PQ2_0 tensors;
  `llama-bench -ngl 0 -m <pq2_0-model-or-converted>` if a PQ2_0 file is at
  hand, else the backend-ops timings
- depends: none

### bench-before-after {#bench-before-after}

Record `llama-bench -ngl 0` TG128 and PP512 on this machine before the change
(the plan/0001 anchor run) and after, and file both tables here.

- verify: the before/after table names the binary, the model file, and the
  exact command line for each number
- depends: #avx2-ptq1-0-vec-dot, #avx2-pq2-0-branch
