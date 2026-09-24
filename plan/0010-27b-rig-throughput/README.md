# plan/0010, the 27B at rig scale, dual Turing throughput

## Why

The primary system is the rig: two RTX 6000 cards, 24 GiB each, both
sm_75. plan/0008's attribution set the frame: full-offload decode is
GPU-work-bound (about 95 percent busy across the pair), the host ceiling
is about 30 tok/s, and the sequential layer split loses to a single card
(28.26 against 28.95) because the handoffs cost more than halved streams
gain. With 48 GiB total the 27B fits whole on one card, both packings fit
together, and the KV budget reaches serving-grade context (the neighbor
deployment runs 262144 on this class of memory). The levers the
attribution and the session's research named are device overlap, batched
serving, speculative drafting, and the kernel ladder at rig scale. The
goal, as set by the operator: unlimited speedups that apply at rig scale.

## Scope

- Full-offload decode throughput for the 27B on the rig: the four levers
  of the build sequence below.
- Serving context is a free variable of the rig (24 GiB per card admits
  large-context operation); measured at the points each lever names.
- Out of scope: the 1650 side quest (plan/0009, parked at its locked
  operating point), the 4B, and the Vulkan runtime.

## Verification

Each lever lands with before and after numbers: tg128 single stream with
its command line, plus aggregate tok/s for the serving levers, recorded
here; anything touching numerics carries the perplexity parity method;
suites stay green on touched paths. The milestone closes when every task
below is landed or refused with numbers on the record.

## Build sequence

### overlap-the-pair {#overlap-the-pair}

Why two cards lose to one: the layer split runs the pair sequentially
with per-boundary handoffs. Investigate the fork's pipeline-parallel
machinery (referenced in `llama_context::process_ubatch`), whether it is
reachable as a flag or needs wiring, and why the parallel split modes
fail at model load for these types (`-sm row`, `-sm tensor`).

- verify: before and after full-offload tg128 with command lines, or the
  refusal with the code path and the load error on the record
- depends: none

### batch-the-server {#batch-the-server}

The per-pass cost (the weights stream, the graph launch, the dispatch
floor) amortizes across slots. Run llama-server at full offload with
increasing `--parallel`, aggregate tok/s and single-stream latency side
by side, at a short and a serving-length context.

- verify: the aggregate table with its commands lands here
- depends: none

### draft-with-dflash2 {#draft-dflash2}

The ProCreations DFlash2 head (about 2 GB, purpose-built for this model,
reported acceptance near 50 percent) drafting for the 27B at full
offload, where the verify pass amortizes the 5.95 GB weight stream over
every accepted token.

- verify: acceptance and before and after tok/s recorded here
- depends: none

### rig-kernel-ladder {#rig-kernel-ladder}

The plan/0009 census ladder at 27B scale: kill the per-token graph launch
(about 1.9 ms across the pair), fuse activation quantization (215
`quantize_q8_1` launches per token), fuse the norm family; the
1,840-kernel graph shrinks stepwise and each step is measured.

- verify: per-change before and after tg128 with command lines; suites
  green; perplexity parity
- depends: none
