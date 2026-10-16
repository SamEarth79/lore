# LORE-STORY-001-004: Graph schema and deterministic base seed

## Description

As the team, I want `lore_base` built deterministically from committed data and a committed extraction cache, so that deploys never depend on a live LLM and every visitor starts from the same graph.

## Acceptance criteria

1. `graph/schema.py` creates the `:Entity` fulltext index on `name, alias_text` and range indexes on `Entity.id`, `Source.id`, `Fact.id`, `Fact.subject_id`.
2. `scripts/seed_base.py` builds `lore_base` from `data/synthetic/` and the committed `data/cache/extract/` with zero LLM calls (verified by running with `LLM_DISABLED=1`).
3. Seeding twice produces identical node/edge counts and identical fact ids.
4. A seed check asserts every Fact has ≥1 `SOURCED_FROM` edge (gather business rule 1) and fails the seed otherwise.
5. The shared `FRESHNESS` Cypher fragment computes freshness at read time; after seeding, `GET`-equivalent queries show the expected suspect facts from the gold file and the frozen ledger-batch fact as trusted.
6. Seeding records `lore_base` memory usage and node count into `lore_registry` `Meta`; implementation notes state the chosen `MAX_SANDBOXES` and `maxmemory` derived from them.
7. The API at startup creates indexes and seeds `lore_base` if missing (idempotent).
8. Timestamps are integer epoch seconds; `lore_base` holds a frozen `Meta {key:'clock'}`.

## Wireframe

none

## Requirements implemented

- gather.md: Business rules #1, #3
- gather.md: Security & reliability #9
- gather.md: Technical constraints #1

## Depends on

- LORE-STORY-001-003

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
