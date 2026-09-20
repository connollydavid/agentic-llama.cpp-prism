# Plan: agentic-llama.cpp-prism — host setup + AVX/AVX2 ternary kernels for 4GiB CUDA targets

## Objective

Turn the empty `C:\Users\david\Development\agentic-llama.cpp-prism` into an **agentic project** per `connollydavid/host`, then execute its first milestone set: make **Ternary-Bonsai-2-27B (PTQ1_0, 5.95 GB)** run well on a **4GiB Turing CUDA GPU + i9-10885H** split, using **calx-mill** to model the optimal layer split and per-µarch kernel strategy, developed in a new fork **`slartibardfast/llama.cpp-prism-avx2`**. Stretch goal: the SIMD ladder reaches down to **AVX-only CPUs (i7-2600K, Sandy Bridge)** and **i7-4670K (Haswell)**, with **Agner Fog's CPU tables** imported as the pinned reference corpus and calx-mill substrate coverage spanning **all AVX + AVX2 CPU generations**.

Research established the exact gap: the Prism fork's CPU backend already runs both ternary types correctly, with the Hadamard FWHT runtime and all hybrid-attention ops CPU-complete — but **PTQ1_0 has only a scalar generic vec_dot on every architecture**, **PQ2_0's only x86 SIMD requires AVX-VNNI/AVX512-VNNI** (Comet Lake and everything pre-Alder-Lake/Zen4 has neither), and the weight-repack fast path (batch-1 decode) exists only for PQ2_0 under AVX512-VNNI. The port is SIMD kernel work in `ggml/src/ggml-cpu/arch/x86/quants.c` and `repack.cpp`.

**All work runs from WSL2.** Locked decisions: gh authenticated as slartibardfast (create fork + push); Turing 4GiB primary GPU target; host repo on the Windows side (`/mnt/c/Users/david/Development/agentic-llama.cpp-prism`) with software object stores in WSL2 ext4 via `.host-software` `store=` lines.

## Phase 0 — WSL2 environment

1. `wsl.exe -l -v`; inside the distro check CPU flags, WSL CUDA (`nvidia-smi`), `git cmake ninja gcc g++ gh rustc cargo`, gh auth as slartibardfast.
2. Install if missing: `gh` (+ auth login if needed), `rustup` (calx-mill + host tools), Linux CUDA toolkit (nvcc, for `GGML_CUDA=ON` builds), python3.
3. Download `host-lifecycle` v0.54.2 + `host-lint` v0.18.1 **linux-amd64** into WSL `~/.local/bin`.

## Phase 1 — Host adoption

From WSL, in the repo directory:

1. `host-lifecycle classify .` → case **a**; mode **Shallow**.
2. `host-lifecycle adopt --at . --purpose "AVX/AVX2 ternary kernels for 4GiB CUDA" --name agentic-llama.cpp-prism` — scaffolds `cast/ plan/ call/`, stamps `.host` at template revision **b917d4d**, seeds `LEXICON`/`MEMORY.md`, initial commit.
3. Copy from host-template @ b917d4d: `AGENTS.md` (the manual), `CLAUDE.md` (pointer), `STRUCTURE.md`, `README.md`, `UPGRADING.md`, `lifecycle.manifest`, `link-skills.sh`, `.gitignore`, `.host-lintignore`, `prose.yml` workflow. Append project specifics: WSL2-execution rule, store layout, build/test commands, pin discipline, Agner Fog reference corpus.
4. Submodules at pins: `host-template` @ b917d4d, `tools/host-lint` @ 0eeabc2, `tools/host-lifecycle` @ 6f9c626, `tools/allium` @ 4870af9, `tools/specula` @ ffb2442. `.gitignore` += `/software/`.

## Phase 2 — Software embedding

1. `gh repo fork PrismML-Eng/llama.cpp --clone=false`, rename to **`llama.cpp-prism-avx2`**; pin **9a9394a8** (verify default branch at fork time).
2. Branch `bonsai-split` in `slartibardfast/calx-mill`; pin **a2e53dcf**.
3. `.host-software`: `llama-cpp-prism` (canonical = default branch; `worktree = avx2-port <sha> store=/home/david/stores/llama.cpp-prism-avx2 host=linux`), `calx-mill` (canonical = default; `worktree = bonsai-split <sha> store=/home/david/stores/calx-mill host=linux`), plus reference stanzas for gating tools (host-lint `hooks = pre-commit`, locally built artifact).
4. `software --materialize .`, build host-lint, `software --install-hooks .`, `bootstrap`/`--verify-setup`, `validate plan/ call/`, `prose .`, `refs --gate .`, `software --check`.
5. `call/` decisions: 0000 adoption, 0001 fork-embed, 0002 verification scoping (kernel correctness via the fork's own suites: `test-quantize-fns`, `test-backend-ops`, `test-ptq1_0-element-map`; no `.allium` initially), 0003 store layout. `plan/PLAN.md` indexes milestones 0001–0006.

## Phase 3 — plan/0001: Agner Fog corpus + calx-mill substrate & split model

1. **Reference corpus**: fetch Agner Fog's optimization manuals (`instruction_tables.pdf`, `microarchitecture.pdf` from agner.org/optimize), pin by SHA256 under gitignored `refs/agner/`, and commit an **extracted, per-row-cited data table** (instruction throughput/latency/ports + pipeline widths for Sandy Bridge, Ivy Bridge, Haswell, Broadwell, Skylake…Comet Lake, Zen 1–4) — cited to the pinned PDFs per the host refs discipline ("a foreign citation names its repository"). His CC BY-NC-ND license means: reference + extracts with provenance, not wholesale vendoring into git.
2. On `bonsai-split` (new `examples/bonsai-split` bin over calx-mill's f64 API; ns units; manual ternary byte accounting — built-in dtype table has no ternary entry, and `project()` u32 overflows at 5.95e9): substrates **for all AVX + AVX2 generations** — Sandy Bridge (i7-2600K: 256-bit FP, no FMA, no 256-bit integer ops; ROB 168; DDR3-1333 ≈ 21 GB/s), Haswell (i7-4670K: 2×256-bit FMA; ROB 192; DDR3-1600 ≈ 25.6 GB/s), Comet Lake (**i9-10885H, primary**: AVX2+FMA, ROB 224, DDR4-2933 ≈ 46.9 GB/s), Zen 1/2/3/4, plus **TU117-class Turing 4GiB** (per-SM = proven `tu102_sm()`; 16 SMs; 160–192 GB/s), local RTX 6000 TU102 for anchoring (calx-mill's op table was measured on an RTX 6000 rig), PCIe Gen3 x16 leg. Pipe rates per substrate cite Agner Fog table rows.
3. Model per-token byte streams exactly (PTQ1_0 = 28 B/128 w; PQ2_0 = 34 B/128 w; activations; full-attn KV vs constant linear-attention state), sweep the 64-layer GPU/CPU split, compose with `Phase`/`overlapped_contended`, output: predicted tok/s per split, bottleneck naming, and the **kernel divergence matrix** across the AVX/AVX2/VNNI classes.
4. Anchors: stock-build `llama-bench` on the local machine (full-GPU TG128; CPU-only `-ngl 0` = today's scalar "before" number), registered via `calx-mill gate`. Download the 5.95 GB PTQ1_0 GGUF.

## Phase 4 — plan/0002 (vec_dot kernels) + plan/0003 (repack path), branch `avx2-port`

- **0002**: AVX2+FMA `ggml_vec_dot_ptq1_0_q8_0` (vectorized base-3 trit decode `(b*3^n & 0xFF)*3 >> 8` mirroring the CUDA `ptq1_0_trit` mapping, respecting the non-positional 24B/2B staging; maddubs/madd accumulation, per-block q8-sum subtraction, fp16 group scale); plain-AVX2 branch for `ggml_vec_dot_pq2_0_q8_0`; remove x86 `arch-fallback.h` aliases; extend the element-map test to the new kernel.
- **0003**: `block_ptq1_0x4` + repack + generic gemv/gemm (4×8 q8_0) in `repack.cpp`; **AVX2** specializations (not just AVX512-VNNI) in `arch/x86/repack.cpp`; AVX2 variant for PQ2_0 repack. Batch-1-decode fast path.
- Benchmarks: `llama-bench -ngl 0` before/after in WSL; variant dispatch verified per x86 class (force-select where the local CPU is AVX512-class); calx-mill gate against 0001 predictions.

## Phase 5 — plan/0004: 4GiB end-to-end validation

Emulate the 4GiB budget on the local 24 GB card (`-ngl` from the 0001 optimum, matching context); TG/PP measured vs prediction; mixed CPU+GPU smoke run of the PTQ1_0 model with the Hadamard runtime active; run scripts + Bonsai-demo-style README; push `avx2-port` and `bonsai-split`.

## Phase 6 — plan/0006 (stretch): AVX-only (Sandy/Ivy Bridge) kernels

i7-2600K class: AVX1 has no 256-bit integer ops and no FMA — vec_dot via 128-bit SSSE3 `maddubs` + 256-bit FP accumulation without FMA (mul+add), no repack path (or a 128-bit variant if modeling says it pays). Feature detection grows the ladder: SSE2 → **AVX (new)** → AVX2+FMA → AVX-VNNI → AVX512-VNNI, dispatched by the backend's existing runtime CPU detection. Calx-mill substrates from Phase 3 already predict where this class lands (bandwidth-bound decode ≈ free; prompt processing suffers).

## Verification gates

- Host: `validate plan/ call/`, `software --check` (no HAZARD), `prose .` zero tropes, `refs --gate .`, hooks active.
- Software: fork's `ctest` suites green in WSL; `cargo test` green in calx-mill (Kani untouched); llama-bench deltas recorded; every plan claim carries a mechanical `verify:` line.

## Risks / fallbacks

- WSL2 CUDA toolkit install is large (~3 GB); CPU-only builds proceed in parallel.
- `store=`/symlink semantics of host-lifecycle v0.54.2 confirmed against `software --check` at runtime; fallback: in-repo worktrees on /mnt/c (slower builds, same governance).
- Local dev CPU may be AVX512-class: AVX2/AVX paths verified via forced-variant builds.
- Agner Fog license (CC BY-NC-ND): extracts cited per row, PDFs pinned-and-gitignored, never vendored wholesale.
- If gh auth is stale, pause at fork creation for `gh auth login`.