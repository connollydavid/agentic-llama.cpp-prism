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

## Results (2026-09-24, anchor rig on native Arch, avx2-port @ cd9012a9)

Same silicon as every earlier record (Xeon W-2140B, 2x RTX 6000) now booted
into native Arch rather than WSL2, governor performance at 3.9 GHz. The
Release and RelWithDebInfo trees were configured before the accessor fix was
committed, so they report build id 2379e0d71 while carrying the pin's content;
the LTO tree reports cd9012a9d.

Baselines, `GGML_CPU_PTQ1_0_REPACK=1`:

| run | command core | result |
|---|---|---|
| full offload | `-ngl 99` | pp512 400.00, tg128 28.25 |
| mixed, the 4 GiB fit | `-ngl 37 -fa 1 -t 8` | pp512 294.84, tg128 3.30 |

Against the WSL2 records: full-offload decode slowed (33.11 to 28.25) while
the mixed split improved (2.08 to 3.30). The profile below says the
full-offload move is not host regression; the token simply is GPU work.

### Scheduler telemetry (GGML_SCHED_DEBUG=1, llama-cli --single-turn -v)

- Full offload: about 3.6 splits per compute, a fixed shape (CPU head,
  CUDA0 through layer 31, CUDA1, CPU tail). The model is split across both
  cards at the layer boundary and the devices run sequentially.
- Mixed at ngl 37: 4,014 split headers over 23 logged computes, average
  about 175 per compute and the deepest about 451. The assignment oscillates
  CPU and CUDA inside every block because small GDN-side tensors stay
  host-resident, and every oscillation pays a split handoff with input
  copies. That is the mixed-split CPU leg's disease, measured.
- Graph reuse engages: the traced window records 99 graph captures against
  24,267 graph launches. The rebuild-per-token theory is retired for steady
  decode, and with it candidate 1 of fix-what-the-profile-names (can_reuse)
  closes by measurement.

### The layered tally (host-sample-profile)

Commands as run (model at `/home/dconnolly/models`, decode via
`llama-cli ... -p "Write a long detailed essay..." -n 100000 --ignore-eos
--single-turn`):

```
perf record -F 99 --call-graph dwarf -p <pid> -- sleep 15
for i in $(seq 1 20); do sudo gdb -p <pid> -batch -ex 'thread 24' \
  -ex 'bt 12'; sleep 1; done
nsys profile -t cuda --cuda-graph-trace=node --sample=none \
  -o /tmp/0008-nsys-full --force-overwrite true ./build-reldbg/bin/llama-cli ...
```

| layer | evidence | where the token goes (35.4 ms at 28.25 tok/s) |
|---|---|---|
| host thread cycles | perf, 1,509 samples: 94.93% of cycles sit under `cudaStreamSynchronize`, called from `llama_context::synchronize` by way of `server_context_impl::decode`; 5.11% under `clock_gettime` | the host thread spins in the driver's sync wait; it is not doing host work |
| gdb tally | 20 backtraces of the decode thread 1 s apart: 18 in `cudaStreamSynchronize`, 1 in `post_decode`, 1 in `graph_compute` | same reading, wall-clock weighted |
| GPU occupancy | nsys node-level: 15,009,251 kernel instances in a 305.5 s span; device busy 145.5 s plus 144.5 s; a sampled 100 ms window shows each device 54.2% busy with no overlap | about 95% of the wall has a kernel executing on one device or the other |
| kernel budget | `mul_mat_vec_q` variants: 127 per token at 165-183 us, about 29 ms; the remaining ~1,700 kernels per token average 3.7 us, about 6.4 ms | matmuls dominate; the rest is a dispatch floor of tiny kernels |
| launch cost | `cudaGraphLaunch` 24,267 calls, avg 992 us, median 1,395 us | 1-1.4 ms per token of host launch time, hidden inside the 5% gap budget |
| gaps | both devices idle about 5% of wall combined | the entire removable host budget |

Verdict: the full-offload token is GPU-work-bound. The Why section's premise
(~1.6 ms of GPU kernels, ~28 ms of host cost around a 2242-node graph) was a
plan/0007 measurement artifact: that profile traced CUDA graphs as single
lumps, so the replay's kernels were absent from its kernel table. Traced at
node level, the 2242-node graph expands to about 1,840 kernel launches per
token and they occupy nearly the whole token. Eliminating every remaining
host cost caps at roughly 30 tok/s, far from the 60 tok/s close criterion;
the criterion's second branch is the one this milestone can close on, with
these numbers.

### LTO experiment

`build-lto` versus `build-release`, tg128 full offload: 28.26 against 28.25.
Null, as the attribution predicts: a GPU-bound token does not care about host
codegen.

### What the reframing does to the remaining tasks

- elide-identity-reshapes removes graph nodes, and node removal now pays
  through the GPU-side dispatch floor (the 6.4 ms of tiny kernels), not
  through host time. The census estimate (~2 nodes per transform across
  ~390 transforms) stands, but reshapes and views launch no kernels
  themselves, so the expected tg128 movement is small.
- fix-what-the-profile-names: candidate 1 (graph reuse) closed by
  measurement above. Candidates 2 (split caching) and 3 (tail ops) address
  the 5% gap budget at best. Candidate 4 (permute-chain folding) joins the
  elision in paying through the dispatch floor.
- The levers the profile does name, all outside this milestone's declared
  scope (no new kernels): matmul kernel efficiency (about 29 ms of the
  token, the plan/0007 instruction-bound analysis), overlapping the two
  sequential devices (the fork carries `pipeline_parallel` machinery,
  unused), and the mixed-split split-oscillation disease below.
- mixed-split-cpu-leg keeps its premise: at ngl 37 the ~175 splits per
  compute are real host-side cost on the CPU leg.

The milestone decision this leaves with the operator: close on the
attribution branch and re-cut the GPU-leg work as a new milestone, or
re-scope this one. The measured record here supports either.
