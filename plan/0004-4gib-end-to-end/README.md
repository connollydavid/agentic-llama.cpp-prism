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
- Out of scope: kernel changes (landed by 0002/0003), moving the fork pin.

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
