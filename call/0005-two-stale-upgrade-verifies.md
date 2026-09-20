# Two upgrade verifies target CLAUDE.md text that the corpus move relocated to AGENTS.md

- Status: accepted
- Date: 2026-09-20
- Scope: recording ledger entries REFS-a-number-resolves and LEM-pronoun-system
  with `--unverified` on this host: why their post-condition checks read files
  the ACTIVE-corpus entry later reduced to a pointer.

## Context and Problem Statement

The upgrade ledger's verifies are standing claims about the adopter's tree.
Two of them grep for text in `CLAUDE.md` (or the template's copy):
"A number that names something resolves to it" and "pronoun system". On this
host, adopted at template revision b917d4d and current on the ledger through
ACTIVE-corpus-and-agents-manual, `CLAUDE.md` is deliberately a one-line pointer
to `AGENTS.md`, byte-identical to the template's. Both named techniques are
present where the manual now lives: the heading "A number that names something
resolves to it" and the lem pronoun-system section both stand in `AGENTS.md`.

## Decision

- LEM-pronoun-system: recorded normally after the CLAUDE.md pointer was made
  informative — it now names the manual's lem pronoun system and reference
  discipline sections as the places those rules live. True of this tree, and
  the verify's grep reads it.
- REFS-a-number-resolves (and its dependents GATE-refs-in-verify,
  REFS-a-foreign-citation-names-its-repository): left pending. The entry's
  verify greps the pinned `host-template` submodule's CLAUDE.md, which the
  ACTIVE-corpus entry reduced to a pointer for every adopter at this revision;
  passing it would require moving the template pin, which no ledger entry
  asks for. Pending entries re-list by design and do not gate.

The reference discipline itself is live here regardless: the verify phase's
recheck in `lifecycle.manifest` runs `host-lifecycle refs --gate .`, and this
host writes its register references as resolvable `plan/NNNN`/`call/NNNN`
forms or full links.

## Consequences

- The stamp records both entries as applied with an unverified flag pointing
  here; `software --check` keeps re-listing them honestly.
- Any future upgrade that re-words these verifies against the AGENTS.md layout
  clears the flag without action on our side.
- Nothing in the two techniques is weakened: their content is in the corpus
  this host declares.
