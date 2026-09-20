# Verify the ternary kernels through the fork's own test suites, not a new allium spec

- Status: accepted
- Date: 2026-09-20
- Scope: the verification lane for the SIMD kernel work in the llama-cpp-prism
  component: which suites gate correctness, and why no `.allium` behavioural
  spec is authored at adoption time. The ladder's rules for other components
  are unchanged.

## Context and Problem Statement

The methodology makes the allium lane mandatory once a `.allium` spec exists,
and authoring one is the default path for new behaviour. But the fork already
ships dense, cheap, directly-on-point correctness suites for exactly the code
we are changing: `test-quantize-fns` (both ternary types against reference with
a 0.01 error bound), `test-backend-ops` (mul_mat/get_rows for both types on
every backend), `test-ptq1_0-element-map` (the CUDA trit mapping vs the CPU
codec over 20,000 random blocks, which is the exact hazard a SIMD decode rewrite
risks), and `test-ptq1_0-cuda-dot`. Performance claims are gated by
`llama-bench` before/after runs recorded in the plan milestone, and by the
calx-mill anchor registry (`calx-mill gate`), not by prose.

## Decision

Kernel correctness is gated by the fork's existing suites, run in WSL2 from the
`avx2-port` worktree (`ctest --test-dir build`), plus `cargo test` in calx-mill
for the modeling side. No `.allium` spec is authored for the kernel work now;
the lanes rule keeps its force: if a `.allium` appears later (for example to
specify the runtime CPU-feature dispatch contract), its CI lane and obligations
manifest arrive with it.

## Consequences

- No duplicated specification of behaviour the fork's suites already pin down.
- The obligation "the AVX2 vec_dot matches the scalar reference" is discharged
  by a test that already exists and runs in CI-identical form locally.
- If someone later changes the element map or the codec, the suites catch it
  the same way this decision relies on today.
