# LORE-STORY-001-005: Tool catalogue and entity resolution

## Description

As the agent (and any MCP client), I want a fixed catalogue of parameterised read tools returning facts with freshness, paths and sources, so that every answer is grounded and explainable.

## Acceptance criteria

1. All 12 read tools in architecture.md exist in one registry with Pydantic argument models and return the envelope `{rows, facts, path, sources}`; each fact carries server-computed `freshness` using the documented precedence.
2. Every read uses `ro_query` with a timeout, bounded variable-length hops and an explicit `LIMIT`; no tool exposes a graph/sandbox argument; labels/relationship types come from fixed enums.
3. `entity_id` arguments are validated against `^[a-z]+:[a-z0-9-]{1,64}$` and must exist, else a typed error.
4. `resolve_entity` escapes RediSearch syntax (no leading `-` negation; hyphens only between word chars), falls back to `CONTAINS` matching when the escaped query is empty or fulltext is unavailable, and returns top-5 candidates with scores.
5. `find_knowers` orders by tier (reason-event author → owner/team lead → contributors within 90 days → 1-hop contributors), honours `exclude_ids`, and always excludes the persona who ingested the triggering change.
6. `get_path` uses `shortestPath` with the relationship-type whitelist (no MENTIONS/SOURCED_FROM/AUTHORED hub edges).
7. Feature tests against a seeded test graph: owner lookup returns path + sources; a suspect fact returns its reason event; a superseded fact is excluded unless `include_history`; an unknown id returns the not-found error; a Cypher-injection-shaped name is treated as a literal.

## Wireframe

none

## Requirements implemented

- gather.md: Scope: fixed parameterised Cypher tool catalogue
- gather.md: Business rules #2, #5
- gather.md: Security & reliability #2, #3, #4

## Depends on

- LORE-STORY-001-004

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
