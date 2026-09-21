# plan/0008, eliminating the decode host cost fully

## Why

The evidence chain through plan/0007: full-offload decode spends about
1.6 ms of a 31.4 ms token in GPU kernels; CUDA graph replay is active; the
remaining ~28 ms is host-side work around a 2242-node graph rebuilt and
walked every token. The mixed split inherits the same disease on its CPU
side (~35 ms per layer, ~2.4 GB/s effective). Until this cost is eliminated,
no kernel work moves the decode number; after it, the 1650 pairing's
9.89 tok/s ceiling (plan/0001, occupancy-corrected) becomes reachable.

## Scope

The decode path only: llama.cpp graph build, scheduler split, per-node
processing, and the mixed-split CPU side. No new kernels beyond the folds
that remove nodes; no CUDA-graph-capture dependence (the standing
preference is code, not capture).

## Verification

Each task lands with a before/after tg number on the anchor rig (command
lines in this README) and, for anything touching numerics, a perplexity
equivalence run fold-on vs fold-off. The program closes when full-offload
decode exceeds 60 tok/s (2x today) or the remaining host cost is attributed
to upstream scheduler scope with the numbers to show it, whichever comes
first.

## Build sequence

### host-sample-profile {#host-sample-profile}

Sample the host during steady decode: gdb batch backtraces of the running
process (full offload, then the mixed split), 20 samples spaced a second
apart, tallied by frame. Cross-check the gaps against the nsys timeline
already captured. Deliverable: a ranked list naming where the ~28 ms lives
(build, sched split, per-node dispatch, CPU-backend barriers, logits and
sampling path).

- verify: the tally table lands in this README with the sampling commands
- depends: none

### elide-identity-reshapes {#elide-identity-reshapes}

llama_mul_mat_hadamard emits reshape_2d before and reshape_4d after the
transform even when the shapes already match (the decode case). Elide both
when the reshape is the identity, and elide the leading cont when the
tensor is already contiguous. Census value: about 2 nodes per transform
across ~390 transforms per token.

- verify: census node count drops accordingly; tg128 full offload moves;
  perplexity unchanged
- depends: none

### fix-what-the-profile-names {#fix-what-the-profile-names}

Ranked candidates, decided by the profile outcome:

1. graph reuse not engaging: verify llm_graph_input::can_reuse and the
   context's graph-reuse path for the decode shape; fix the rejection
   cause if it is fork-side
2. scheduler split cost per token: the split walks and re-hashes the graph
   each call; if the profile names it, cache splits for repeated identical
   graphs (fork-side) or reduce split passes
3. per-node dispatch on the CPU backend for logits and sampling ops:
   move the tail ops into fewer nodes or onto the GPU backend wholesale
4. the hadamard permute chains (five nodes each, grouped-V paths): fold
   the permute into the transform kernel's indexing, as the signs fold did

- verify: the chosen fix lands with its own before/after numbers here
- depends: #host-sample-profile

### mixed-split-cpu-leg {#mixed-split-cpu-leg}

The same medicine measured on the mixed split (ngl 37, physical-core
threads, repack on): profile the CPU side the same way, then apply
whichever of the above applies to the CPU backend's per-op barriers and
the repack gemv dispatch path.

- verify: mixed tg at ngl 37 exceeds 4 tok/s on the anchor rig or the
  remaining gap is attributed with numbers
- depends: #fix-what-the-profile-names

### re-baseline-the-model {#re-baseline-the-model}

Fold the measured host costs back into the calx-mill sweep as the
per-op overhead term it lacked (the LACUNAE-named blind spot), and restate
the 1650 and big-Turing projections against the improved binaries.

- verify: the sweep table for both packings lands here with the
  corrected expectation for the target machine
- depends: #mixed-split-cpu-leg
