# LORE-STORY-001-011: GitHub real-repo slice in the full loop

## Description

As a judge, I want Lore to run on real public data from FalkorDB/FalkorDB, so that the staleness signal is shown to work beyond hand-made data.

## Acceptance criteria

1. `scripts/fetch_github.py` (uses `GITHUB_TOKEN` locally only, never deployed) snapshots ~100 recent issues/PRs and their authors to `data/github/snapshot.json`, storing only usernames, public titles/body excerpts, URLs, states and timestamps — no emails.
2. The snapshot maps into the graph: maintainers → `Person {origin:'github', is_persona:false}`; FalkorDB components → `System`; issues/PRs → `Ticket {origin:'github'}`; extraction results are committed to the global cache and merged by `seed_base.py`.
3. Merged PRs and closed issues become change events tagged by aspect through the same extraction pipeline; at least one real-slice fact is suspect after seeding.
4. Knower questions for real-slice facts go to "Repo Steward (simulated)" with `suggested_real_knower` set; real people never receive inbox items or appear as confirming/correcting.
5. Source URLs are validated as http/https before storage; excerpts are short with a link back to GitHub.
6. Tests: no email field anywhere in the snapshot; a real-slice question routes to the steward; the real-repo suggestion chip question returns an answer with GitHub sources.

## Wireframe

none

## Requirements implemented

- gather.md: Scope: real-data slice
- gather.md: Security & reliability #1
- gather.md: Key decisions: synthetic + real repo

## Depends on

- LORE-STORY-001-004
- LORE-STORY-001-009

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
