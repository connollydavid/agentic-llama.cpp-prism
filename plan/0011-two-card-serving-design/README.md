# plan/0011, the two-card serving design from measured numbers

## Why

The operator's decisions, locked before the measurements: the rig serves
PQ2_0, the context window stays f16, and the two cards run as independent
instances (the weights and the full 262144 window fit one card, proven by
demonstration). The design had to come from real numbers: TTFT, ITL with
its tail, aggregate throughput, and fill dependence, measured on the
streaming path an actual client sees. The overnight battery of 2026-09-25
(together with a corrected recovery window for the three phases a script
bug cost) supplies them.

## Scope

- The serving topology for the 27B PQ2_0 on the two-card rig: instances,
  slots, context budget, KV type, and the latency and throughput budget
  per session class.
- The measurements that ground it: the ITL/TTFT fill curve to the full
  window, cross-talk between instances, the two-instance aggregate, the
  single-card batched table, the starvation behavior, and the packing
  perplexity pair.
- Out of scope: kernel work (the batched matmul stays the recorded next
  lever), the 1650 side quest (plan/0009), the 4B.

## Verification

Every number in the design tables traces to a logged measurement (the
battery's results file preserved under the run directory), each phase
runs with its command recorded, and the production llama-server is
restored after every window. The design closes when its tables stand on
measurements and its follow-up list names what is still open.

## Build sequence

### measure-battery {#measure-battery}

The overnight battery: fill curve, cross-talk, aggregate, batched table,
starvation curve, perplexity pair.

- verify: the results land in this README with the probe and server
  commands
- depends: none

### design-tables {#design-tables}

The design itself: topology, per-class latency and throughput budgets,
and the operating guidance the numbers dictate.

- verify: the tables and guidance land here, every entry measured
- depends: #measure-battery

### unified-kv-ab {#unified-kv-ab}

The one confounded observation, resolved: the starvation reproduction
differed from the starved overnight run in both `--kv-unified` and
context size. A clean A/B (same context, unified on and off, decode
behind a foreign prefill) decides whether the unified KV pool is what
serializes slots in this fork.

- verify: the A/B table lands here with commands
- depends: #measure-battery

## Results (2026-09-25, one RTX 6000 per instance unless named)

The battery: one full-window instance per card (PQ2_0, `-ngl 99 -fa on
-ctk f16 -ctv f16 -c 262144 --parallel 1`), a streaming probe (SSE chunk
timestamps; TTFT plus ITL mean, p95, p99, max), fills 4k to 261k with
two repetitions, and the recovery phases rerun after the overnight
script's bug (instance A was never released from card 0, which starved
phases four to six of VRAM; the corrected runs supply them). The
production llama-server was restored after every window.

### The fill curve (the design's spine)

| fill | TTFT fresh | TTFT cached | ITL mean | ITL p95 | ITL p99 | max |
|---|---|---|---|---|---|---|
| 4k | 6.55 s | 0.18 s | 26.9 ms | 55.5 | 82.7 | 82.7 |
| 16k | 18.97 s | 0.21 s | 28.1 ms | 58.2 | 86.2 | 90.5 |
| 32k | 29.35 s | 0.24 s | 29.8 ms | 61.3 | 91.4 | 91.8 |
| 65k | 80.12 s | 0.30 s | 33.5 ms | 68.6 | 102.2 | 103.0 |
| 131k | 217.21 s | 0.42 s | 40.7 ms | 82.6 | 106.4 | 123.8 |
| 196k | 311.08 s | 0.56 s | 48.2 ms | 97.2 | 123.1 | 145.3 |
| 261k | 391.13 s | 0.67 s | 55.5 ms | 111.9 | 149.4 | 166.4 |

Readings: ITL mean climbs 0.11 ms per 1k of fill, exactly the KV term
(64 KiB per token at about 580 GB/s effective); at the full window the
instance decodes at about 18 tok/s. The tail is p95 about 2x and p99
about 3x the mean at every fill (the probe's chunks coalesce at the
faster cadences, so mean, p95, p99, and max are the trustworthy columns).
Fresh TTFT runs 625 to 1115 tok/s of prefill (peaking mid-length, 668
tok/s at the full window); cached TTFT, the slot's own prompt reuse, is
0.2 to 0.7 s at every fill, which is the session-continuation experience.

### Independence and aggregate

Cross-talk: card 0's ITL is byte-identical solo and while card 1 decodes
beside it (28.1 ms mean, p99 86 ms both). The aggregate run: both
instances at 16k fill stream 128 tokens each in 3.6 to 3.9 s, about 66
to 71 tok/s combined single-stream, with each instance's ITL unchanged
at 28.2 ms. Two cards are two servers.

### The single-card batched table (4k fill)

| batch | aggregate tg | per-sequence |
|---|---|---|
| 1 | 36.91 | 36.9 |
| 2 | 59.79 | 29.9 |
| 4 | 71.76 | 17.9 |
| 8 | 82.01 | 10.3 |

Identical to the pair's layer-split table within noise (its 39.01 to
81.76), which closes the loop: the card is not the limiter, the batched
M greater than 1 matmul is, and one card carries the whole serving
curve. Two instances therefore deliver about 164 tok/s aggregate at
batch 8.

### Starvation and the unified-KV observation

The clean reproduction (no `--kv-unified`, context 65536, two slots,
one decoding behind the other's 16.8k prefill) shows no starvation at
any `-ub`: ITL holds at the batch-2 cadence (36.2 ms mean at every of
512, 128, 64). The overnight run that starved (4.93 t/s) differed in
both `--kv-unified` and context size; the confound is named and the
clean A/B is the follow-up task. Design guidance stands regardless: a
single-card instance has no reason to run unified KV, and without it the
scheduler co-phases decode behind prefill at the batched cadence.

### The packing gate

Perplexity on the 8-chunk corpus: PTQ1_0 4.9591, PQ2_0 4.9584. The
packings are quality-equivalent; the rig serves PQ2_0 for its measured
decode advantage with no quality bill.

## The design

Topology: two llama-server instances, one per card via
`CUDA_VISIBLE_DEVICES`, each `PQ2_0 -ngl 99 -fa on -ctk f16 -ctv f16 -c
262144` without `--kv-unified`. Session classes:

- Latency-critical sessions: one per instance (`--parallel 1`). The
  budget is the fill curve above: ITL 27 ms rising to 56 ms at the full
  window, p99 about 3x mean, cached re-entry under a second. Fresh
  long-context TTFT is minutes (6.5 minutes at the full window); the
  prompt cache is the mechanism that keeps sessions interactive, and
  `--cache-ram` extends it across restarts the way production already
  runs.
- Throughput sessions: batch within an instance (`--parallel 8`): 82
  tok/s per instance, about 164 across the pair, at a 98 ms per-token
  cadence everyone in the batch shares.
- The card boundary is the isolation boundary: sessions on different
  cards never see each other (measured, byte-identical ITL); sessions
  sharing an instance share the batch cadence.

The standing kernel lever is unchanged and now sole: the batched M
greater than 1 matmul (one card already equals the pair at every batch,
so nothing else buys rig-scale throughput). The open follow-up is the
unified-KV A/B, and the production-migration question (replacing the
W4A16 service with these instances) belongs to the neighbor project's
call, with the numbers above as the offer.
