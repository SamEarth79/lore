# LORE-STORY-001-003: Synthetic company dataset and gold labels

## Description

As the team, I want a synthetic company written backwards from the demo script, plus hand-checked gold labels, so that the self-healing loop is demonstrable and extraction quality is measurable.

## Acceptance criteria

1. `data/synthetic/` contains entities (≥8 people incl. personas Priya Shah, Dana Lee, Marco Ruiz, Sam Okafor and the simulated Repo Steward; ≥4 teams; ≥8 systems incl. payments-api, auth-v1, auth-v2, checkout-web, ledger-batch, billing-api; ≥2 projects incl. Project Atlas; ≥3 decisions incl. picking VendorY) with ids matching `^[a-z]+:[a-z0-9-]{1,64}$`.
2. Ownership, membership, WORKS_ON and DEPENDS_ON edges exist so that payments-api has ≥2 dependents with distinct owners (for the notify demo) and auth-v1 has ≥2 dependents.
3. ≥40 sources (docs, tickets, chat threads) with authors and timestamps; ≥1 old doc about the frozen system ledger-batch with no later events; ≥1 recent chat about payments-api limits followed by a later contradicting event.
4. A timeline file of ≥6 scripted events (incl. the three ingest presets: ownership handover of payments-api to Platform, VendorY limit raised to 500 rps, auth-v1 deprecation), each with expected subject, aspect and structural type.
5. Seeded data yields 8–12 suspect facts after full timeline replay (not a mostly-red board), and the frozen-vs-churning contrast pair is identifiable by id.
6. `data/gold/` holds ~50 hand-checked expected facts (subject_id, aspect, predicate, normalised object) and expected change events per source, plus expected suspect facts after replay.
7. A validation script checks referential integrity (every author, subject and dependency id exists) and passes.

## Wireframe

none

## Requirements implemented

- gather.md: In scope: synthetic company dataset with timestamped change events and gold set
- gather.md: Freshness board contrast (alternate flows)
- gather.md: Business rule #3

## Depends on

- LORE-STORY-001-001

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
