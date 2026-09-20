# agentic-llama.cpp-prism

A host for porting Prism ML's ternary GGUF types to the x86 SIMD ladder, so
[Ternary-Bonsai-2-27B](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf)
(PTQ1_0, 5.95 GB of weights) runs well on a CUDA GPU with only 4 GiB of VRAM
paired with an ordinary AVX-class CPU.

The model needs 5.95 GB of weights per token; a 4 GiB card cannot hold them.
The fork's CUDA and Metal kernels are fast, but its CPU backend runs PTQ1_0
through a scalar generic vec_dot and PQ2_0 only under AVX-VNNI, which the
i9-10885H target lacks. So the plan is: measure and model the GPU/CPU layer
split with [calx-mill](https://github.com/slartibardfast/calx-mill), then land
AVX2 kernels (vec_dot and the weight-repack path) in the fork
[slartibardfast/llama.cpp-prism-avx2](https://github.com/slartibardfast/llama.cpp-prism-avx2)
until the CPU leg stops being the bottleneck.

This repository is the *thought*: plans, decisions, personas, and the modeling
record. The *action* lives in the embedded software worktrees under
`software/`, materialized from `.host-software`. Read `AGENTS.md` for the
operating manual, `plan/PLAN.md` for the milestone index, and `call/` for the
decisions. Everything runs from WSL2; see the project-specifics section of
`AGENTS.md`.
