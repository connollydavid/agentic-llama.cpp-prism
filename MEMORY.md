# MEMORY.md

## Session Log

- Project purpose: AVX/AVX2 ternary kernels for 4GiB CUDA targets: Bonsai 2 27B on llama.cpp-prism-avx2, split-modeled with calx-mill

### 2026-09-20 — host adopted, software embedded

- Adopted the methodology at template b917d4dc81da; stamp baseline advanced to RENAME-repro-waiver; the REFS chain (3 entries) stays pending per call/0005 (stale verifies grep the pinned submodule CLAUDE.md).
- The fork: the slartibardfast account already carried an August ggml-org mirror named llama.cpp; one-fork-per-network forced repurposing it. Renamed to llama.cpp-prism-avx2, old master preserved at backup/pre-prism-master (db231ec0d1af), master forced to the known-good release pin 9a9394a895b9 (prism-b10709; upstream master has since moved to 5ea87dda). Port branch avx2-port; calx-mill branch bonsai-split; both pushed at the pins recorded in .host-software.
- Layout: host repo on /mnt/c (Windows side, native agent access), software stores in /home/david/stores (WSL2 ext4) via store= lines (call/0003). Everything executes from WSL2 Ubuntu-22.04.
- Gates at adoption: validate plan/ call/ ok; software --check no hazards; refs --gate resolves everything; prose 0 flags.
- Local rig: 2x Quadro RTX 6000 (TU102) + Xeon W-2140B (Skylake-X: AVX2 + AVX512, no VNNI). The new AVX2 kernels run natively here because the CPU lacks every VNNI class the existing ternary kernels require. Target machine: i9-10885H (Comet Lake, AVX2 only) + 4 GiB Turing (TU117 class).
- gh holds two accounts: connollydavid (active) and slartibardfast; switch to slartibardfast for fork pushes.

### 2026-09-20 — kernels and repack landed; anchors and the packing verdict

- plan/0002 complete: AVX2 vec_dot for PTQ1_0 (reference-staging decode) and the plain-AVX2 branch for PQ2_0; test-quantize-fns green both types; x86 fallback alias retired.
- plan/0003 complete except the optimized gemm (owed work, generic path ships): block_ptq1_0x4 = PQ2_0 slot layout, shared AVX2 gemv; three real-model defects fixed (repack buffer 21 percent undersized; the prism.hadamard rotation matrices crashed the loader in the repack buffer, now stored verbatim with generic-path op approval; first gemm cut regressed prefill below scalar).
- Measured CPU-only (W-2140B, 16t): tg32 0.13 -> 1.81 tok/s (13.9x). PQ2_0 GPU anchor: tg128 41.90 tok/s (268 GB/s decode-effective) vs PTQ1_0 31.84 (180 GB/s): Turing decode is instruction-bound, PQ2_0 unpacks cheaper, yet PTQ1_0+repack still wins the 4 GiB + i9-10885H split (6.55 vs 6.21 predicted) because 28-byte blocks fit 37 layers against 31.
- The calx-mill worktree file lost two patches once (found reverted to the last commit with a clean tree; cause unknown, re-applied and verified on disk before building). Lesson recorded: verify patch application with grep before cargo.

### 2026-09-20 — the silent-commit trap

The host-lint pre-commit hook (shared into every software worktree through the store .bare/hooks) blocks commits on confirmed naming tells and returns rc 1 silently alongside its warning lines. With git commit -q under set -e, two kernel commits vanished without an error line: the work was staged, the commit never existed, and a later push shipped the empty branch. Rules from this: echo the commit hash after every commit in a software worktree; run the hook directly for its true rc when a commit surprises you. The confirmed tell was the word "stage" in my kernel comments (the codec traversal vocabulary); reworded to "group". Upstream comment lines (Q2:, Q8:) lint as warnings only and pass.

### 2026-09-20 — validation numbers and the honest gap

Final measurements on the anchor rig (avx2-port @ 7750dd8, fixed binaries): CPU-only pp512 184.96 vs 174.58 scalar (prefill parity restored by the unrolled vec_dot); CPU decode 0.27 vec_dot / 0.41 repack vs 0.13 scalar; mixed 4 GiB emulation ngl37 tg128 1.01 / pp512 253, and ngl 32 decodes faster at 1.59 (Turing per-layer decode overheads mean the optimum sits below the byte-budget maximum). The sweep's 6.5 tok/s target assumed a bandwidth-bound CPU leg; measured batch-1 CPU decode runs 2.3 GB/s effective, so the leg is per-op-overhead-bound — the exact blind spot in calx-mill LACUNAE. Owed work, by payoff: decode-step op profiling, the optimized AVX2 gemm, then re-run the split curve. Earlier 1.81 and 8.98 tok/s numbers were artifacts (a dead-code-eliminated loop after a dropped hsum); treat only the 7750dd8 numbers as real.
