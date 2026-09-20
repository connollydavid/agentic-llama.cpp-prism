# plan/0001, substrates for the SIMD ladder and the 4 GiB split model

## Why

Ternary-Bonsai-2-27B in PTQ1_0 needs 5.95 GB of weights read per generated
token. A 4 GiB card cannot hold them, so the model must split across the GPU
and the host CPU, and the host leg runs on CPUs (i9-10885H primary; i7-2600K
and i7-4670K stretch; Zen 1-4 as divergence classes) whose SIMD support stops
anywhere from AVX to AVX-512. Where the layers go, and which kernel family
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
constants beside them. No changes to the core, the proofs, or the shipped
substrates.

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
