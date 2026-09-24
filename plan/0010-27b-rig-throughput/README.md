# plan/0010, the 27B at rig scale, dual Turing throughput

Status: closed 2026-09-25, every task landed or refused with numbers; the
standing levers for the next milestone are named in the close section
below.

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

## Results (2026-09-24, the production shape and the split-buffer gap)

**The production config this rig already serves** (llama-server.service, the
neighbor deployment): `Qwen3.8-27B-W4A16-AR16.gguf` (a 4-bit AutoRound
file, not the ternary), `-sm tensor -fa on -ctk f16 -ctv f16 --ctx-size
262144 --parallel 1 --kv-unified --cache-ram 65536 --spec-type none
--jinja --no-mmap`. Three consequences for this milestone:

- The serving shape to beat is 262144 f16 KV sessions; the
  batch-the-server sweep runs at that total KV budget, slots trading
  context depth (each slot of N costs 262144/N of the 16.8 GiB the full
  f16 cache occupies across the pair).
- Tensor split demonstrably works on this hardware and driver: the
  neighbor build, upstream llama.cpp at db231ec0d, runs `-sm tensor`
  daily.
- Production runs speculation off (`--spec-type none`), so the draft
  lever is greenfield against this rig's own serving.

**The split-buffer gap localized.** The fork's CUDA backend has no
split-buffer implementation at all (the model-side check at
src/llama-model.cpp:1097 throws "does not support split buffers" because
the device's split buffer type yields nothing), and the fork's base
(prism-b10709) predates the upstream CUDA split-buffer work that
db231ec0d carries. The fix for overlap-the-pair is a bounded backport of
the upstream implementation into the fork; then the parallel split modes
load for the ternary types and the pair computes simultaneously (the
matmul halves are independent), against the measured sequential pair at
28.26 and the single card at 28.95.

**Pipeline parallelism was already on.** It auto-enables at full offload
with layer split (llama-context.cpp:533), so the 28.26 measurement
already had it. The plan/0008 nsys timeline's strictly alternating device
busy explains why it cannot help a layer split: each layer depends on the
previous one, so only the boundary copies overlap. True device overlap
requires the parallel split modes, that is, the backport above.

### The first measured round (2026-09-24, PQ2_0 on the pair)

Files: `Ternary-Bonsai-2-27B-PQ2_0.gguf` 7,206,168,928 bytes, sha256
`3907dc1658db1f78…`; `Bonsai-2-27B-DFlash2-Q8_0.gguf` 2,056,415,104
bytes, sha256 `9dd11c8adb910058…` (self-recorded anchors, the plan/0001
precedent).

Baselines: full offload on the pair, layer split, pp512 699.14, tg128
39.93 ± 0.02 (the WSL2 record's class, its 41.90 within the OS delta).

The server at the production shape (262144 f16 unified KV, one slot):
prefill 626 t/s at 16.8k fill, decode 36.25 t/s, and the graphs-reused
counter finally visible in a log: 158 of 160. At two slots the decodes
never co-scheduled: slot 1's 16.8k prefill starved slot 0's decode to
4.93 t/s (26.99 clean after), and both prefills slowed to the 440 to 500
t/s class. The server-level parallelism question is scheduling, not
hardware.

The hardware answer, `llama-batched-bench` (4k fill, 128 tokens, `-npl
1,2,4,8`):

| batch | aggregate tg | per-sequence |
|---|---|---|
| 1 | 39.01 | 39.0 |
| 2 | 59.97 | 30.0 |
| 4 | 71.29 | 17.8 |
| 8 | 81.76 | 10.2 |

Two conclusions. The per-pass weight stream amortizes exactly as
predicted: 2.1x aggregate at eight slots, prefill flat at 689 t/s at
every batch. And the limiter at batch 8 is the fork's generic batched
matmul for M greater than 1 (plan/0003's owed gemm, now with a rig-scale
payoff): the batched forward costs 97.8 ms against 25.6 ms single-stream,
far above the KV-arithmetic floor, so an optimized batched PQ2_0 matmul
is the lever that pushes the aggregate past 100. The draft-with-dflash2
A/B is queued for the next window (the head and the PQ2_0 target it was
calibrated against are both on disk and hashed); overlap-the-pair's
backport is scoped by the split-buffer finding above.

Ops notes of the round: port 8090 belongs to another user's service (the
sweep's instant 501s), a bare `wait` in a script that backgrounds a
server waits on the server too (the timing lines never ran; the servers'
own slot logs carried the numbers), and `pkill -f` matched this shell's
own command text again, the recorded 2026-09-21 trap.

### draft-with-dflash2: the head refuses this fork; n-gram measured neutral

The published head does not load: the fork's dflash loader expects 81
tensors and the DFlash2 file carries 58 (`done_getting_tensors: wrong
number of tensors; expected 81, got 58`, the loader at
src/models/dflash.cpp). The vendor's card says drafters for newer
releases need their one-time conversion plus fork patches, shipped as a
prebuilt CUDA 13.3 sm120 runtime and a build-runtime.sh against newer
mainline. Landing it here is a bounded port of those patches, scoped the
same way as the split-buffer backport; until then the dflash lever is
refused with the load error on the record.

The fork's in-tree n-gram drafting, measured on the same payload
(`--spec-type ngram-map-k --spec-draft-n-max 4`): 39.60 tok/s decode with
3 of 256 tokens accepted and the eval time unchanged at 25.25 ms per
token, against the 39.93 no-draft baseline. Neutral on fresh-generation
text, as the method's nature predicts; it stays a free option for
repetitive workloads.

### rig-kernel-ladder: re-aimed by measurement

The single-stream ladder steps (launch kill, quantize fusion, norm
fusion) are refused for this milestone in favor of the lever the
amortization table ranks above them: the batched M greater than 1 matmul
for PQ2_0, whose measured cost is the whole gap between the 81.76 tok/s
aggregate at batch 8 and the KV floor (the batched forward at 97.8 ms
against 25.6 single stream, plan/0003's owed gemm at rig scale). The
kernel budget that justified the ladder at single-stream scale belongs,
at serving scale, to the batched path; the ladder itself remains recorded
in plan/0009 for the 1650 program where single stream is the regime.

### Milestone close

Every task is landed or refused with numbers on the record: the
amortization table (batch-the-server, landed), the split-buffer refusal
with the load error and the backport scope (overlap-the-pair, refused),
the dflash load refusal plus the n-gram measurement (draft-with-dflash2,
refused with data), and the measured re-aim of the kernel ladder. The
standing throughput levers for the next milestone: the batched PQ2_0
matmul, the split-buffer backport, the dflash runtime patches, and the
server's decode co-phasing.
