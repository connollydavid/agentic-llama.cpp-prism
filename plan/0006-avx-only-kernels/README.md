# plan/0006, AVX-only kernels for Sandy Bridge and Ivy Bridge (stretch)

## Why

The stretch targets (i7-2600K, i7-4670K era machines) include Sandy/Ivy
Bridge parts with AVX but no AVX2: no 256-bit integer ops, no FMA, no VNNI.
The plan/0001 substrate set models them; if the model says the CPU leg is
bandwidth-bound anyway (the likely verdict for decode), the win is in
prompt processing and in simply having a path faster than the scalar
fallback.

## Scope

- An AVX-level vec_dot for PTQ1_0 and PQ2_0: 128-bit SSSE3/SSE4.1 integer
  maddubs for the trit/code decode and dot, 256-bit AVX FP for the
  accumulation tail, mul+add instead of FMA.
- Feature-detection placement in the x86 dispatch order (SSE2, AVX, AVX2,
  AVX-VNNI, AVX512-VNNI), verified on this machine by forcing the AVX class.
- Out of scope unless the model says otherwise: an AVX repack layout (the
  plan/0001 divergence matrix decides whether 128-bit repack pays on this
  class at all).

## Verification

The suites from call/0002 green with the AVX class forced; a before/after
`llama-bench` table for the local machine running the forced-AVX binary; the
plan/0001 prediction for the SNB class compared against the measurement.

## Build sequence

### avx-vec-dot {#avx-vec-dot}

The 128-bit integer + 256-bit FP kernels for both types, wired under the
AVX (not AVX2) guard.

- verify: `test-backend-ops -b CPU` and `test-quantize-fns` green with the
  AVX variant forced
- depends: none

### forced-class-bench {#forced-class-bench}

Measure the forced-AVX binary locally and file the table; compare against
the plan/0001 SNB-class prediction.

- verify: the table names the forcing mechanism, binary, and command lines
- depends: #avx-vec-dot
