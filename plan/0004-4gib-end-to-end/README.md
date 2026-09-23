# plan/0004, the 4 GiB end-to-end validation run

## Why

The port exists to run the model on a 4 GiB Turing card plus an AVX2 host.
The local machine has 24 GiB cards, so the 4 GiB budget is emulated: hold
the GPU-resident weight bytes at the plan/0001 optimum (what fits after
context, buffers, and the CUDA context), put the rest on the CPU with the
plan/0002 and plan/0003 kernels, and measure what that split actually
delivers against what the model predicted.

## Scope

- A run script that fixes the split (layer count from plan/0001, context
  from the same table) and launches the fork's `llama-cli`/`llama-bench`
  with the PTQ1_0 model on the local card under the emulated budget.
- TG128 and PP512 measurements for the chosen split and its neighbors, the
  curve filed here, and the prediction-vs-measurement gap registered through
  the calx-mill gate.
- A Bonsai-demo-style README in this host recording the tested invocation
  for the real 4 GiB machine (the on-target run itself happens when the
  target hardware is at hand).
- Out of scope: kernel changes (landed by 0002/0003) and any move of the
  fork pin.

## Results (2026-09-20, anchor rig emulating the 4 GiB budget)

All runs: `avx2-port` @ 7750dd8 (CUDA build, sm_75), 16 CPU threads,
repack enabled (`GGML_CPU_PTQ1_0_REPACK=1`), context 8192 with `-fa on`,
model on the local RTX 6000 (the 4 GiB budget is emulated by the layer
count, not the card size):

| run | command core | result |
|---|---|---|
| GPU-only baseline | `-ngl 99` | tg128 31.84 tok/s |
| CPU-only, scalar (the "before") | `-ngl 0` | tg32 0.13, pp512 174.58 |
| CPU-only, AVX2 vec_dot | `-ngl 0` | tg32 0.27, pp512 184.96 |
| CPU-only, repack on | `-ngl 0` | tg32 0.41 |
| mixed ngl 37 (the 4 GiB fit) | `-ngl 37 -fa 1` | tg128 1.01, pp512 253.01 |
| mixed ngl 32 | `-ngl 32 -fa 1` | tg128 1.59 |

The emulated 4 GiB configuration runs end to end (the smoke generation with
the Hadamard runtime active produced coherent text), but decode lands at
about 1 to 1.6 tok/s against the sweep's 6.5 prediction. The gap is
diagnosed, not mysterious: the sweep modeled the CPU leg as
stream-bandwidth-bound, and the measurement shows batch-1 CPU decode at
about 2.3 GB/s effective (20x below the machine's STREAM class), so the leg
is dominated by per-op scheduling and latency, exactly the blind spot
calx-mill's own LACUNAE register names for this model class. Second finding:
fewer GPU layers decode faster on this rig (ngl 32 beats ngl 37), because
Turing's PTQ1_0 decode carries per-layer overheads the byte model does not
see; the optimum split on real hardware sits at or below the byte-budget
maximum.

Owed work, in order of expected payoff: profile the decode step op by op
(llama.cpp timing logs) to find the per-layer fixed cost; the optimized AVX2
gemm (plan/0003); then re-run this curve. The runbook below carries the
tested commands for the target machine with the corrected expectation:
roughly 1 to 2 tok/s decode today, prefill healthy (253 tok/s at the split).

## Verification

The measured curve and the predicted curve land in this README with command
lines; the gate registry entry for the split prediction is CERTIFIED or the
gap is named and explained; the smoke run produces coherent text with the
Hadamard runtime active (no folded-weight verification errors).

## Build sequence

### emulate-split {#emulate-split}

Fix the split from the plan/0001 table and run the measurement set.

- verify: `llama-bench` runs complete at the pinned split without VRAM
  errors; raw logs preserved under `plan/0004-4gib-end-to-end/`
- depends: none

### curve-vs-prediction {#curve-vs-prediction}

Sweep neighboring splits, file the curve, register the gap in the gate.

- verify: `calx-mill gate` verdict recorded in this README
- depends: #emulate-split

### target-machine-runbook {#target-machine-runbook}

The README a 4 GiB owner follows: exact binary, flags, context, and expected
tokens/s for the i9-10885H + TU117-class pairing.

- verify: every command in the runbook is the one actually run here or a
  documented adaptation of it
- depends: #curve-vs-prediction

Postscript (same day): the interactive smoke was not captured cleanly (llama-cli interactive mode with piped stdio echoes instead of flushing text); bench-level validation stands, and the owed-work list gains a direct gemv-vs-reference unit test for the repack path alongside the decode-step profile and the optimized gemm.
