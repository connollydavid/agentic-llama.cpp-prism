# plan/0009, Bonsai-4B fast decode, the CUDA path first

## Why

plan/0008's corrected model settles the interactive question for the 27B
pairing: about 10 tok/s at the bandwidth ceiling, 3.07 tok/s expected today,
with the mixed-split CPU leg per-op overhead as the wall. The interactive
model for the target machine is therefore Ternary-Bonsai-4B (Qwen3-4B base,
36 plain-attention layers, PQ2_0 about 1.06 GB): it fully offloads into the
4 GiB card, and the mixed split, the CPU leg, and the per-op overhead disease
all leave the critical path. The decision recorded in the 2026-09-24 session:
a mega-kernel decode path for the 4B, CUDA first, the generic Vulkan runtime
as its own milestone later. Both target cards are sm_75, so the anchor rig's
RTX 6000 is the direct proxy, scaled by the sweep's SM and bandwidth axes.
Estimate against the 8.3 ms bandwidth floor: 90 to 120 tok/s on the 1650.

## Scope

- The 4B decode path on sm_75 CUDA: fused kernels (per-layer fusion or one
  cooperative launch, decided by measurement), PQ2_0 slot layout, weights
  resident, batch-1 decode.
- Anchor-rig validation of everything (same shader model as TU117).
- calx-mill gains the 4B model constants; the 1650 projection prints from
  the corrected sweep.
- Out of scope: the Vulkan runtime and the P630 LUT kernel (their own
  milestone once CUDA lands); the 27B program (plan/0008 closes on its
  attribution branch); any draft-dual-use work beyond the tokenizer check,
  whose outcome decides a later milestone.

## Verification

Suites green on the touched paths (test-quantize-fns, test-backend-ops,
test-ptq1_0-element-map, test-ptq1_0-cuda-dot); every kernel change lands
with before and after tg128 and pp512 on the anchor rig and its command
lines here; anything touching numerics carries a perplexity parity run
(the plan/0008 corpus method). The milestone closes when the anchor-rig 4B
decode number is attributed between GPU work and the dispatch floor, the
mega kernel's own number is measured against it, and the 1650 projection
is restated against the measured rate.

## Build sequence

### fetch-and-hash {#fetch-model}

Download Ternary-Bonsai-4B-PQ2_0.gguf, verify size and sha256 against the
Hugging Face tree, and check the tokenizer match against the 27B file
(vocab size and token sampling): the outcome decides whether the 4B can
moonlight as the 27B's draft head in a later milestone.

- verify: the hash, the size, and the vocab comparison land in this README
- depends: none

### baseline-4b {#baseline-4b}

As-is fork benches on the anchor rig: full offload on one GPU (the sm_75
proxy for the 1650), pp512 and tg128. The models are considered separately:
the 4B is a full-offload GPU program with no mixed split and no CPU leg, so
no CPU-only point is taken here.

- verify: the table lands here with command lines, beside the sweep's
  prediction for the same points
- depends: #fetch-model

### attribute-4b-token {#attribute-4b-token}

Where the as-is 4B token goes: kernel census and GPU-busy share per token
by the plan/0008 layered method, shortened. This decides the mega kernel's
payoff estimate before any kernel is written, the lesson plan/0008 paid for.

- verify: the tally lands here with the sampling commands and names the
  removable share
- depends: #baseline-4b

### mega-kernel-prototype {#mega-kernel-prototype}

The CUDA fast path, shape decided by the attribution: fused PQ2_0 decode
with the attention and MLP chain of the 36-layer architecture, weights
resident, batch 1.

- verify: before and after tg128 on the anchor rig with command lines;
  suites green; perplexity parity
- depends: #attribute-4b-token

### project-1650 {#project-1650}

calx-mill gains the 4B constants (the Qwen3-4B tensor table, provenance the
way plan/0001 recorded the 27B's), and the sweep prints the 1650 projection
against the measured anchor rate.

- verify: the sweep table lands here with the corrected 1650 expectation
- depends: #mega-kernel-prototype

## Results (2026-09-24, anchor rig)

### fetch-and-hash

`Ternary-Bonsai-4B-PQ2_0.gguf`: 1,074,969,344 bytes, sha256
`829abec7eb92f5bf464762be7c9e8a45d777c714543a1474fc90cee20e698beb`
(self-recorded anchor; the HF tree API publishes no per-file sha256, the
plan/0001 precedent).

Vocab comparison, both headers parsed field by field: the 4B carries the
Qwen3 tokenizer (151,669 tokens, ChatML specials at 151643); the 27B carries
a 248,320-token Qwen3.8-generation vocab. No match, so the 4B cannot draft
for the 27B: the dual-use question closes negative, and any draft milestone
belongs to the 27B's own line (the purpose-built DFlash2 head stays its only
path). The models are separate programs from here.

Shape constants for project-1650: qwen3 arch, 36 layers, d_model 2560, ffn
9728, 32 query and 8 KV heads at 128, context 32768 (yarn over 8192, factor
4, rope base 5e6), 1.0195 GB of weights on card.

### baseline-4b

Full offload on one RTX 6000 (`CUDA_VISIBLE_DEVICES=0`, `-ngl 99 -p 512 -n
128`, build-release at the pin's content):

| run | result |
|---|---|
| pp512 | 4669.06 |
| tg128 | 194.07 ± 0.42 |

Decode streams 1.0195 GB per token at 194 GB/s decode-effective, 72 percent
of the 27B's PQ2_0 rate on the same card (268 GB/s). The gap is the
per-token dispatch floor across 36 layers of smaller matmuls, exactly the
mega kernel's target: at the 27B's rate the same card would run about 263
tok/s, and the bandwidth floor sits at 1.0195 GB over 609 GB/s, about 590.

### attribute-4b-token (2026-09-24, the plan/0008 method, shortened)

nsys with node-level graph tracing, one RTX 6000, context 8192, traced
decode at 144 tok/s (6.94 ms per token; the node tracer's overhead is
host-side):

| layer | per token | share |
|---|---|---|
| GPU kernel execution | 6.11 ms (693 kernels per token, 87 percent GPU busy) | 88 percent |
| cudaGraphLaunch submission, GPU idle | 0.92 ms (median 914 us across 8,137 launches) | 13 percent |
| matmul kernels (mul_mat_vec_q, three variants, 217 per token) | 3.43 ms | 49 percent |
| flash attention family at 8k fill | 1.49 ms | 21 percent |
| activation quantization (quantize_q8_1, 218 launches) | 0.49 ms | 7 percent |
| the norm family (rms_norm and fused rope variants, 145 launches) | 0.68 ms | 10 percent |

Flag A/Bs at the 8k operating point (`-p 8064 -n 128`): the as-is defaults
are already optimal. `tg128 193.80` as-is, `193.54` with `-nopo 1` (null at
full offload, as expected: no CPU layers), `173.90` with `-fa off` (flash
attention on by default is worth 20 tok/s). No configuration lever remains;
everything past 194 is kernel work.

### The 70 tok/s gate on the 1650

The intermediate goal (agreed 2026-09-24): bank the 70 tok/s band on the
1650 before the full mega kernel. Expressing it in anchor-rig units, since
both cards are sm_75: 70 tok/s on 14 SMs is 5.1 GB/s per SM; clock-matched
to the 6000 that is an anchor aggregate of about 400 GB/s, and the measured
concentration behavior from plan/0007 (the same warp count on fewer SMs
runs each SM faster) softens that to the prerequisite the sweep can check:
**anchor tg128 at 8k context at or above 280 tok/s, the 27B's PQ2_0
efficiency class (3.9 GB/s per SM) on the same card.** The census ladder to
it, each step independent and measurable: kill the per-token graph launch
(-0.92 ms, persistent or fused decode), fuse activation quantization into
the matmul kernel (-0.49 ms and 218 nodes), fuse the norm family (-0.4 ms
and 145 nodes); the ladder ends near 3.3 ms per token, about 300 tok/s
anchor, which projects to 70 to 100 tok/s on the 1650 under plan/0007's
concentration model. The real number comes from the card when at hand.
