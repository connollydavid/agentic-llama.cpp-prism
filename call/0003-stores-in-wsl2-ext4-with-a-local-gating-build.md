# Object stores in WSL2 ext4, host repo on the Windows mount, gating tool built locally

- Status: accepted
- Date: 2026-09-20
- Scope: the physical layout of this host: where the repository, the software
  bare stores, the build worktrees, and the reference material live, and how
  the gating tool (host-lint) is built without a container repro lane. Covers
  the `store=` lines in `.host-software` and the `toolchain`/`artifact` keys of
  the `host-lint` stanza.

## Context and Problem Statement

All execution for this project happens from WSL2 (Ubuntu-22.04): CUDA builds,
cargo, ctest, llama-bench. The host repository sits on the Windows filesystem
at `/mnt/c/Users/david/Development/agentic-llama.cpp-prism` so the desktop
agent session edits governance files natively. The 9p mount is far too slow for
CUDA or Rust builds, so software worktrees must not be checked out there. The
reference-host recipe builds the gating tool inside a digest-pinned musl Docker
image; this machine has no such lane wired, but has the exact Rust version the
methodology pins (1.95.0) as a local rustup toolchain.

## Decision

- The two port components carry `store=` worktree lines into
  `/home/david/stores/` (ext4): `llama.cpp-prism-avx2` for the `avx2-port`
  worktree, `calx-mill` for the `bonsai-split` worktree. The tool materializes
  the stores there and symlinks the in-tree handles; builds run inside the
  worktrees, never across the mount.
- Canonical reference worktrees (`master`, `main`) stay in-tree on the Windows
  side; they are read-only anchors, not build trees.
- The `host-lint` stanza records `artifact` (the built binary and its SHA256)
  with `toolchain = rustup 1.95.0 (local, x86_64-unknown-linux-gnu)`. The
  container reproducible-build lane is not wired; the artifact hash still
  gates `software --check` and `--install-hooks` verification.

## Consequences

- Build performance matches native Linux; governance files stay native to the
  agent session; `software --check` still verifies every worktree at its pin.
- `software --verify-build` on host-lint would need a `build` command to be
  meaningful in this environment; until a container lane exists, the artifact
  hash plus the pinned Rust version is the accepted evidence.
- Anything that must be built should live under `/home/david/stores/`; adding
  a new build on the Windows side is a mistake, not a convenience.
