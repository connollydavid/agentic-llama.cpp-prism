# Runbook: Ternary-Bonsai-2-27B PTQ1_0 on a 4 GiB Turing card with an AVX2 CPU

Every command below is the one actually run for validation on this host's
anchor rig (2x RTX 6000, Xeon W-2140B, WSL2), or a one-line adaptation of it
for the target machine (i9-10885H + TU117-class 4 GiB, PCIe Gen3 x16). The
binary is the `avx2-port` build of `slartibardfast/llama.cpp-prism-avx2`
(the fork of PrismML-Eng/llama.cpp; stock llama.cpp cannot load these files).

## Build (from WSL2 or Linux)

```bash
git clone -b avx2-port https://github.com/slartibardfast/llama.cpp-prism-avx2
cd llama.cpp-prism-avx2
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=75   # 75 = Turing;
        # drop -DGGML_CUDA=ON for a CPU-only build (macOS/Linux CPU)
cmake --build build -j
```

On the 4 GiB laptop the same commands apply; `-DCMAKE_CUDA_ARCHITECTURES=75`
covers TU117. The CPU side needs no flags: the AVX2 kernels are selected by
the build's runtime feature detection, and the PTQ1_0 weight repack engages
on any AVX2 machine.

## Model

```bash
curl -L -o model.gguf https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf/resolve/main/Ternary-Bonsai-2-27B-PTQ1_0.gguf
```

## Decode (the phase this port targets)

The plan/0001 sweep pins the split at ngl 37 for a 4 GiB card at context
8192 (weights ~3.1 GB after the CUDA context, compute buffers, the 537 MB
KV cache, and the linear-attention states):

```bash
./build/bin/llama-bench -m model.gguf -ngl 37 -p 512 -n 128 -fa 1 -t 8
```

Interactive use, with the card's sampling defaults from the model card:

```bash
./build/bin/llama-cli -m model.gguf -ngl 37 -fa on -c 8192 -t 8 \
    --temp 1.0 --top-p 0.95 --top-k 20 \
    -p "Explain quantum computing in simple terms." -n 256
```

Lower `-ngl` (fewer layers on the GPU) if the card reports out-of-memory at
a larger context; each layer is about 85 MB. Context costs roughly 64 KB of
VRAM per token across the sixteen full-attention layers.

## What to expect

Measured on the anchor rig emulating the 4 GiB budget (numbers and commands
in this milestone's README): decode between 1.0 and 2.1 tok/s depending on
split and thread count (ngl 37, repack on, physical-core threads: 2.08
tok/s), prefill around 250 tok/s at the split. Two operational rules from
plan/0007: use `-t <physical cores>` (t8 beat t16 by 24 percent on the mixed
split), and the Hadamard signs fold is on by default (LLAMA_HADAMARD_FOLD_SIGNS=0
reverts it). The plan/0001 sweep's decode ceiling for the 1650 + i9-10885H
is 9.89 tok/s once the CPU leg's per-op overhead closes; the gap and its
census live in plan/0007. Big-Turing cards with VRAM to spare should run
the PQ2_0 file instead (41.9 tok/s full offload measured), the packing
verdict the Turing bifurcation. CPU-only with the AVX2 kernels: pp512 184.96
against the 174.58 scalar baseline; decode 0.27 (vec_dot) to 0.41 (repack
on) against 0.13 before.
