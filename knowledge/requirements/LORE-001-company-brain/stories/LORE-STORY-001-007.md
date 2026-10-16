# LORE-STORY-001-007: Sandbox sessions and personas

## Description

As a visitor, I want my own resettable copy of the company created only when I first change something, so that my actions never affect other visitors and the demo stays up.

## Acceptance criteria

1. `POST /api/session` sets an HttpOnly, Secure, SameSite=Lax `lore_vid` cookie with a 128-bit random hex id when absent; no graph is created on session start or on any GET route.
2. `ensure_writable` (per-vid `asyncio.Lock`) lazily copies `lore_base` to `sbx_<vid>` on first write, registers it in `lore_registry`, and validates the name against `^sbx_[0-9a-f]{32}$`.
3. When `MAX_SANDBOXES` is reached, the least-recently-seen sandbox outside a 60s grace window is evicted, leaving a tombstone; a later write from the evicted visitor returns 409 `sandbox_expired` and leaves no orphan graph.
4. When FalkorDB `used_memory` > 85% of `maxmemory`, a new copy is refused with 503 `demo_at_capacity` and the visitor remains read-only on `lore_base`.
5. The sweeper (startup + every 5 min) deletes sandboxes idle > `SANDBOX_TTL_S`, removes old tombstones, and deletes `sbx_*` graphs with no registry row via `GRAPH.LIST`, taking each vid's lock and re-checking the registry first.
6. Per-sandbox node-count ceiling (3× base + margin) returns 413 for all writes once exceeded.
7. `GET /api/personas` returns `[{id,name,title,team,owns[],good_start,unread}]` for `is_persona` people only; real GitHub people are never personas.
8. `PUT /api/session/persona` validates `^person:[a-z0-9-]{1,64}$` and `is_persona`, and never triggers a copy.
9. `POST /api/session/reset` drops the sandbox and clears any tombstone; the next write recopies.
10. Tests cover: concurrent first writes create exactly one sandbox; write after eviction → 409 with no orphan; capacity guard → 503; reset recopies; GET never creates a graph.

## Wireframe

none

## Requirements implemented

- gather.md: Business rule #10
- gather.md: Security & reliability #7
- gather.md: Key decisions: access via persona switcher, per-visitor sandbox

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
