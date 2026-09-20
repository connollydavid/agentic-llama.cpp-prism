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
