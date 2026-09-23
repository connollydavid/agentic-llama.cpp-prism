# Embed the Prism fork as slartibardfast/llama.cpp-prism-avx2, pinned at the known-good release

- Status: accepted
- Date: 2026-09-20
- Scope: the Where-room component for the llama.cpp port work: which repository
  carries the AVX/AVX2 kernels, at which pin, and how the pre-existing fork under
  the same account was repurposed. Covers `llama-cpp-prism` in `.host-software`.

## Context and Problem Statement

The ternary GGUF types (PTQ1_0, PQ2_0) and the Hadamard activation runtime only
exist in PrismML-Eng/llama.cpp; stock llama.cpp cannot load the files. The port
work needs a fork we can push to. GitHub allows one fork per network per
account, and the `slartibardfast` account already carried a fork named
`llama.cpp`, an August 2026 mirror of upstream ggml-org/llama.cpp used to
track contributor PR branches (`0cc4m/*` refs), diverged from PrismML-Eng
master by 96 fork-only commits.

## Decision

Repurpose the existing fork rather than attempt a second one:

- Rename `slartibardfast/llama.cpp` to `slartibardfast/llama.cpp-prism-avx2`.
- Preserve the old master line at branch `backup/pre-prism-master`
  (db231ec0d1af7466e6b10e5e3e3b4d7fffcd69f6) before repointing, so the
  rename costs nothing.
- Force master to PrismML-Eng commit 9a9394a895b96003ca842a6041cb28ac49a108f7,
  the `prism-b10709-9a9394a` release the Bonsai-demo repo pins as known-good,
  not the moving upstream master (5ea87dda at adoption time).
- Pin `.host-software` `llama-cpp-prism` at 9a9394a; the port branch is
  `avx2-port`, created at the same commit.
- The old `0cc4m/*` PR-tracking refs stay untouched in the fork.

## Consequences

- The pin is a release anchor: upstream PrismML-Eng movement does not shift the
  ground under the port. Moving to a newer upstream pin is a deliberate
  decision recorded here.
- The user's PR-tracking fork changed name and master; the backup branch keeps
  the old line recoverable, and the PR refs are unaffected.
- Pushes to the fork require the `slartibardfast` GitHub account (present in
  `gh auth` as a secondary account).
