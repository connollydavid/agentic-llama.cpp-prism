# The identity-reshape elision never fires for this model, so it stays out of the fork

- Status: accepted
- Date: 2026-09-24
- Scope: the plan/0008 task elide-identity-reshapes; why it closes falsified
  with no code change in `avx2-port`.

## Context and Problem Statement

plan/0008's build sequence carried the task on this premise:
`llama_mul_mat_hadamard` emits a leading cont/reshape_2d and a trailing
reshape_4d even when they are the identity (the decode case), worth about two
nodes per transform across roughly 390 transforms per token. An elision
guarded on `contiguous && ne[0] == n && ne[2] == ne[3] == 1` was implemented in
`src/llama-impl.h` and built.

Measured: the decode graph is 4,382 nodes both before and after the change,
with 1,188 reshape and 738 view nodes unchanged (census via gdb, breaking at
`ggml_backend_sched_alloc_graph` and dumping with `ggml_graph_dump_dot`, a
method that counts view-class nodes the scheduler debug dump skips). The
identity case never fires anywhere in this model's graph: its hadamard sites
(the unified KV path, `src/llama-kv-cache.cpp`) always reshape `[5120, X]` to
`[1024, X times 5]` because the transform is block-diagonal over 1024-wide
blocks, so the leading reshape is structural at every batch size, and the
trailing reshape_4d restores the shape the callers need. The plan's premise
held for no call site of this model. The change was reverted: for this fork it
was dead code, and the methodology's least-code principle says leave it out.

## Decision

- Close plan/0008#elide-identity-reshapes as skip, falsified by measurement:
  the census cannot drop because no identity reshape exists to elide.
- The evidence is re-derivable: both dot dumps and their histograms, with the
  gdb command that produced them, recorded in plan/0008's Results.
- Any revisit that wants the remaining 1,926 view-class nodes gone must fold
  the reshape into transform-kernel indexing (the permute-chain fold plan/0008
  scoped out), not elide it at graph level.

## Consequences

- Positive: no speculative branch ships in the fork; the task closes with the
  premise's falsification on record instead of an unmeasured assumption.
- Negative: none for this model. A future model whose hadamard sites hand the
  builder a matching shape would need the elision rewritten against its actual
  call sites.
- The skip receipt for the task cites this decision.
