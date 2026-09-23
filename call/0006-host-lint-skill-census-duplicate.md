# The host-lint skill census counts its embedded copy and its submodule copy twice

- Status: accepted
- Date: 2026-09-23
- Scope: the setup-gate skill census on this host, where the gating tool is both a
  referenced submodule and an embedded Where-room component. The copy that
  reached this host re-materialized partially (stores without hooks, skills, or
  the gating artifact), and running the completeness gate surfaced a census
  collision inside host-lifecycle@0.54.2, not in this project's recipe.

## Context and Problem Statement

`host-lifecycle software --verify-setup` enumerates every skill a materialized
worktree or an initialized submodule offers, then requires each to resolve from
`.claude/skills/<name>`. Two legitimate, distinct sources here both offer a
skill named `host-lint`:

- the referenced tool submodule `tools/host-lint` (wire-the-tools; every
  verification tool is a pinned submodule), and
- the embedded Where-room component `software/host-lint/main` (call/0003: the
  gating tool is embedded so its store carries the build `--install-hooks`
  deploys).

Both trees ship the same root `SKILL.md`. A single symlink under
`.claude/skills/host-lint` can canonicalize to only one of the two real paths,
so exactly one of the two requirements HAZARDs whichever way the link points,
and `host-lifecycle bootstrap` reports `conflict` and "could not link every
skill" on the same tree it just linked.

## Decision

- Recorded upstream, where the census should dedupe a skill that a submodule
  and a recipe component both offer and keep a single canonical link home:
  [connollydavid/host-lifecycle#28](https://github.com/connollydavid/host-lifecycle/issues/28).
- Locally, the skill symlink stays at the census-first home the tool itself
  prefers. The bootstrap linker sorts role-neutral sources and links the
  embedded component's `software/host-lint/main` copy where a conflict keeps
  the submodule one unresolved; `link-skills.sh` (the template's project script)
  conversely links only `tools/*` and names the submodule copy. Either choice
  leaves the other requirement open, so the concrete target does not matter for
  the verdict. L set the link to the workflow that runs on this host
  (`software --verify-setup`), which canonicalizes to `software/host-lint/main`.
- The single residual `host-lint — skill linked is missing` gap is a known,
  tool-carried census bug against this project's sanctioned layout, not a
  missing hook or build artifact. It is tracked in the upstream issue; when
  host-lifecycle dedupes, the setup gate closes without project action.
- The record layer stays clean: this decision, the upstream issue, and a
  MEMORY.md entry are how the residual is auditable from a fresh session.

## Consequences

- `software --verify-setup` reports one HAZARD (`host-lint — skill linked`)
  attributable to the tool, with every artifact, hook, and materialized source
  genuinely present.
- `software --verify-build` remains a separate, unmet reprover for the gating
  tool: the host-lint stanza carries an `artifact` but no `build` recipe, which
  that lane DRIFTs on. Building the binary in a local toolchain and installing
  the verified binary is the attested path here (call/0003, "a local gating
  build"), not the container re-derivation.
- Nothing in the four gates (`validate`, `reconcile`, `refs --gate`) is weakened.