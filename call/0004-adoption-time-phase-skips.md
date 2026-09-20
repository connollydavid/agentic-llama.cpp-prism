# Adoption-time phase skips: no remap, no publish, no releases yet

- Status: accepted
- Date: 2026-09-20
- Scope: the lifecycle phase receipts recorded at adoption for this host:
  which phases are skipped at the start and why each skip is safe to make
  permanent in the record.

## Context and Problem Statement

The lifecycle is unconditional, so every phase owes a receipt; a `skip`
receipt must cite a decision. At adoption three phases genuinely do not apply
yet, and recording them as skips without a citation would be noise later.

## Decision

- `remap` skipped: this host was adopted into an empty directory; there are no
  ordinal-named files to rename, so there is no rename dictionary to apply.
- `publish` skipped: no mdBook site is wired (the template's `site.yml` was
  not copied); there is nothing to publish. Revisit if a site is added.
- `release` skipped for all three components (`llama-cpp-prism`, `calx-mill`,
  `host-lint`): no release has been cut; each will get its own receipt at the
  first real release.

## Consequences

- The receipt record shows the phases as deliberately skipped, not forgotten;
  the fail-safe re-listing of owed work is satisfied by this citation.
- When the first kernel release is cut from `avx2-port`, the release phase
  receipt supersedes the skip for that component.
