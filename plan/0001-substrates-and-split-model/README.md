# plan/0001, substrates for the SIMD ladder and the 4 GiB split model

## Why

Ternary-Bonsai-2-27B in PTQ1_0 needs 5.95 GB of weights read per generated
token. A 4 GiB card cannot hold them, so the model must split across the GPU
and the host CPU, and the host leg runs on CPUs (i9-10885H primary; i7-2600K
and i7-4670K stretch; Zen 1-4 as divergence classes) whose SIMD support stops
at one class on the AVX to AVX-512 ladder. Where the layers go, and which kernel family
matters on which CPU, is a bandwidth-and-issue arithmetic question: exactly
what calx-mill projects. Deciding it by measurement-backed modeling instead
of intuition is the point of this host.

## Scope

- Agner Fog's instruction and microarchitecture tables, pinned by SHA256
  under `refs/agner/`, with the rows this project uses extracted into a
  committed, per-row-cited table.
- calx-mill substrates on the `bonsai-split` worktree: Comet Lake
  (i9-10885H), Haswell (i7-4670K), Sandy Bridge (i7-2600K), Zen 1-4, the
  TU117-class Turing 4 GiB part, and the local RTX 6000 (TU102) anchor,
  each a data function citing its Agner Fog rows.
- `examples/bonsai-split.rs`: exact ternary byte accounting (PTQ1_0 28 B per
  128 weights, PQ2_0 34 B per 128 weights), the 64-layer GPU/CPU split sweep
  with PCIe activation traffic, serial and overlapped composition, and the
  per-microarchitecture kernel divergence matrix.
- Local anchors on this machine (2x RTX 6000, Xeon W-2140B): a stock-build
  `llama-bench` run at full offload and at `-ngl 0`, registered in the gate
  registry, so every prediction carries a measured anchor.
- Out of scope: kernel implementation (plan/0002 and plan/0003), the
  end-to-end 4 GiB emulation run (plan/0004).

## Results (2026-09-20, sweep at examples/bonsai-split.rs @ bonsai-split f1b9de0)

Model facts from the shipped GGUF tensor table (header parsed 2026-09-20):
arch qwen35, 64 blocks, d_model 5120, FFN 17408, full attention every 4th
layer (16 full-attn at 81.47 MB, 48 linear at 84.91 MB), output head 278.1 MB
read per token at stream time, and the embedding reads one row per token;
5.936 GB total.

Anchor (local rig, stock pin build, `./build/bin/llama-bench -m
/home/david/models/Ternary-Bonsai-2-27B-PTQ1_0.gguf -ngl 99 -p 512 -n 128
-fa 1`): **tg128 31.84 tok/s, pp512 460.51 tok/s**. That tg number is the
load-bearing finding: 5.66 GB in 31.5 ms is 180 GB/s, 0.30 of the card's 609
GB/s measured DRAM, so PTQ1_0 batch-1 decode on Turing is
instruction/launch-bound, not bandwidth-bound. The sweep's GPU axis is
therefore a decode-effective rate anchored to this measurement (TU117 scales
the instruction side by SM count, 16/72).

Primary target (TU117-class 4 GiB, i9-10885H, context 8192): best feasible
split ngl 37, predicted 5.35 tok/s with the vec_dot kernel alone, 6.55 with
the repack path.

Divergence matrix (best tok/s, vec_dot kernel vs repacked):

| CPU | vec_dot | repacked | gain |
|---|---|---|---|
| i9-10885H (CML) | 5.35 | 6.55 | 1.22x |
| i7-4670K (HSW) | 4.33 | 4.70 | 1.08x |
| i7-2600K (SNB, AVX) | 2.62 | 4.06 | 1.55x |
| Zen 1 1700-class | 4.80 | 6.25 | 1.30x |
| Zen 2 3700X-class | 6.72 | 6.72 | 1.00x |
| Zen 3 5800X-class | 6.80 | 6.80 | 1.00x |
| Xeon W-2140B (local) | 6.24 | 10.99 | 1.76x |

Kernel priority handed to plan/0002 and plan/0003: the vec_dot lands first
(correctness and every non-repack path), the repack path follows as the
compute-heavy decode win on CML/SNB/SKX (Pipe-bound legs there), while Zen
2/3 sit at their memory bound already and prefill everywhere favors repack.
The vec_dot budget is the landed kernel's design-based 90 uops per 28-byte
block; the repack steady state is 8. The pure-GPU prediction at the anchored
rate is 31.82 tok/s against the 31.84 measured.

Packing comparison (both files benched on the same rig): PQ2_0 decodes
Turing at 41.90 tok/s (268 GB/s effective, 0.44 of DRAM) against PTQ1_0's
31.84 (180 GB/s, 0.30) and prefills 715.9 against 460.5, so the cheaper
unpacking wins both phases per byte on this generation. The sweep still
picks PTQ1_0 for the 4 GiB target: the 28-byte block fits 37 layers in the
budget against PQ2_0's 31, and PTQ1_0 with the repacked path predicts
6.55 tok/s to PQ2_0's 6.21 (both files carry the AVX2 CPU kernels of
plan/0002). The packing verdict holds the model card's guidance: PTQ1_0
wherever memory is tightest.

## Verification

- `cargo test` green in the `bonsai-split` worktree, including the shipped
  three-substrate falsifier (the new substrates are data; no proof changes).
- `calx-mill gate` reports the registered anchors within tolerance or names
  the extrapolation honestly.
- The sweep prints, for each candidate split, the predicted tokens/s, the
  bottleneck resource, and the bytes moved per substrate; the chosen split is
  the argmin and the table lands in this README with the command lines that
  produced it.

## Build sequence

### pin-agner-corpus {#pin-agner-corpus}

Fetch `instruction_tables.pdf` and `microarchitecture.pdf` from
agner.org/optimize, store under `refs/agner/` (gitignored), record SHA256
pins in `refs/agner/PINS.md`, and extract the rows used by the substrate
definitions into `refs/agner/extract.md` with per-row page citations. The
manuals are CC BY-NC-ND: reference and cite, never vendor wholesale.

- verify: `sha256sum -c refs/agner/agner.sums` passes; every substrate pipe
  rate in `add-cpu-substrates` cites an extract row or a measured anchor
- depends: none

### add-cpu-substrates {#add-cpu-substrates}

One data function per CPU in the calx-mill substrate set, following the
shipped `avx512_core()` shape, plus the device-level constants (SM count,
effective DRAM bytes/ns from STREAM-class measurement or datasheet) as named
constants beside them. The core, the proofs, and the shipped substrates stay
as they are.

- verify: `cargo test` green in the worktree; each substrate's axes trace to
  an Agner Fog extract row or a cited datasheet number
- depends: #pin-agner-corpus

### write-bonsai-split {#write-bonsai-split}

The sweep driver as `examples/bonsai-split.rs`: byte volumes computed from
the block sizes (never the built-in dtype table, which has no ternary entry),
f64 phases in nanoseconds, serial and overlapped composition, and output that
names the binding resource per split.

- verify: `cargo run --release --example bonsai-split` prints the sweep table
  for the primary target and the divergence matrix; hand-check one row's
  arithmetic against the model's `Phase` math
- depends: #add-cpu-substrates

### anchor-local {#anchor-local}

Build the fork unmodified (CPU and CUDA) in the `avx2-port` worktree at the
pin, download the PTQ1_0 GGUF, and measure `llama-bench` TG128 at full
offload and `-ngl 0` on this machine. Register both numbers in the anchors
registry. The CPU number is the before-picture for plan/0002 and plan/0003.

- verify: `calx-mill gate --registry <registry> --anchor-id ...` returns
  CERTIFIED or PROVISIONAL for the measured anchors; the bench command lines
  land in this README
- depends: none

### choose-split {#choose-split}

Run the sweep against the anchor-calibrated model, pick the split for the
4 GiB budget (weights in VRAM after context and buffers), and record the
predicted tokens/s for the target machine and the kernel priority list that
plan/0002 and plan/0003 implement.

- verify: the chosen split row traces to a sweep invocation recorded here;
  the priority list names, per SIMD class, which kernel family the model
  predicts to matter
- depends: #write-bonsai-split, #anchor-local
