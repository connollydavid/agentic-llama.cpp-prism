# The gating tool builds from a local rust toolchain, not a container lane

- Status: accepted
- Date: 2026-09-24
- Scope: the reproducibility lane (`software --verify-build`) for the
  `host-lint` Where-room component; why it is waived rather than container-
  reproduced, and why the fix surfaced in the CI gate.

## Context and Problem Statement

`reproducible-build.yml` runs `software --verify-build` on every push. The
`llama-cpp-prism` and `calx-mill` components record no artifact, so the lane
skips them; `host-lint` records an artifact (`target/release/host-lint`,
sha256 `5ebd3607…`) but no `build` recipe, so the lane DRIFTs: "has an
artifact but no build recipe to reproduce it". The gate cannot be green.

`host-lint` is the gating tool, embedded as a Where-room component purely so
its store carries the binary that `--install-hooks` deploys (call/0003). It is
the same repository pinned as the `tools/host-lint` submodule; its true build
and release are the tool's own upstream CI, not this project's. Its local
toolchain here is recorded as `rustup 1.95.0 (local, x86_64-unknown-linux-gnu;
call/0003)` — a machine-local build, not a digest-pinned container image.

## Decision

- Record `repro-waiver = call/0007` on the `host-lint` stanza: the artifact is
  a local gating build deployed from the tool's worktree, and its factorial
  reproducibility is attested by host-lint's own pinned release CI (`rust:
  1.95.0` container in the reference workflow), not by a second container
  rebuild staged in this host. `--verify-build` then prints `WAIVED` and the
  lane is green.
- The two port components remain skip-eligible (no artifact recorded): their
  reproducible-build disipline is owed only once a deployable artifact is cut
  from `avx2-port` / `bonsai-split` in a release.

## Consequences

- `reproducible-build.yml` passes: llama-cpp-prism and calx-mill skip, host-lint
  is WAIVED.
- The waiver is a case decision (migrated/reference tooling), never available
  to greenfield software; if a kernel release later cuts a deployable artifact
  from `avx2-port`, that component must record `build`/`toolchain` and earn its
  hash rather than inherit a waiver.
- The concrete trigger that surfaced this was the re-materialization re-run of
  the gate on a fresh checkout; the fix is a recipe+decision change, committed
  with the same CI run that proves it.