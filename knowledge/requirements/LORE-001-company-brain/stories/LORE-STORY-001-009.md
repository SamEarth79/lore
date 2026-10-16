# LORE-STORY-001-009: Ingest pipeline, staleness and timeline replay

## Description

As a visitor, I want to feed Lore a new ticket, chat or doc (or the next seeded event) and see facts extracted, staleness detected and the right person asked automatically.

## Acceptance criteria

1. `POST /api/ingest {kind: doc|ticket|chat, title? ≤200, text 1..4000}` strips control chars and validates at the boundary; calls `ensure_writable`; per-sandbox cap of 200 ingested items returns 413.
2. Extraction uses the committed global cache (read-only at runtime) for seed/timeline/preset text; live free text is cached only inside the sandbox; cache misses call DeepSeek with the entity roster and aspect enum, text delimited as untrusted.
3. Validation drops facts whose evidence span is not an exact substring, whose subject doesn't resolve, or whose aspect is outside the enum; caps ≤10 facts and ≤5 events per item; facts from live free text always start `unverified`; author is the current persona.
4. Source, facts and change events are written in one Cypher query (atomic); structural events fan out TOUCHES as specified; all timestamps come from the monotonic sandbox clock (each write advances it ≥1s).
5. After writing, newly suspect/unverified facts on touched subjects each get one knower question (no duplicate open inquiry), never addressed to the ingesting persona; real-repo knowers route to the Repo Steward with `suggested_real_knower` set.
6. Response returns `{source_id, facts_created[] (with evidence offsets and confidence), events[], flagged_facts[], inbox_items_created[], graphVersion}`.
7. `GET /api/ingest/presets` returns the 3 presets; `GET /api/timeline` returns `{clock, next_item_preview, remaining}`; `POST /api/timeline/next` ingests the next scripted item.
8. Tests: preset ownership handover flips "Payments team owns payments-api" to suspect and creates a question for Dana Lee; a payments UI ticket (different aspect) does not flag the rate-limit fact; confirm-then-later-timeline-event makes the fact suspect again (clock ordering); extraction failure writes nothing; XSS payload text is stored verbatim and returned as data.

## Wireframe

none

## Requirements implemented

- gather.md: Self-healing loop steps 1–3
- gather.md: Business rules #3, #4, #9
- gather.md: Security & reliability #5, #9

## Depends on

- LORE-STORY-001-002
- LORE-STORY-001-005
- LORE-STORY-001-007

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
