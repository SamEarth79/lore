# LORE-STORY-001-018: Production deploy and smoke test

## Description

As a judge, I want a stable public URL that works on first click, so that I can try Lore myself during judging.

## Acceptance criteria

1. API and FalkorDB are deployed on Railway as two services on private networking; FalkorDB has a password, volume, `maxmemory` and `noeviction`, and no public port.
2. `lore_base` is seeded on release (or at startup if missing) from committed data with no LLM calls.
3. Secrets (`DEEPSEEK_API_KEY`, FalkorDB credentials) are platform secrets; `GITHUB_TOKEN` is not set.
4. Single API instance; the machine does not auto-stop (no cold start on a judge's first click).
5. A Playwright smoke suite runs against the production URL: Ask golden path (`ask.answer` visible with `ask.path-text` and `ask.source`), made-up system shows `ask.not-in-graph`, ingest preset → `ingest.result-flagged`, inbox correct → `inbox.receipt-corrected`; CSP header present on `/` and `/api/health`.
6. `/mcp` responds publicly with the read-only tool list.
7. The public URL and MCP URL are recorded in the README.

## Wireframe

none

## Requirements implemented

- gather.md: Technical constraints #5
- gather.md: Key decisions: live public deployment

## Depends on

- LORE-STORY-001-001
- LORE-STORY-001-002
- LORE-STORY-001-003
- LORE-STORY-001-004
- LORE-STORY-001-005
- LORE-STORY-001-006
- LORE-STORY-001-007
- LORE-STORY-001-008
- LORE-STORY-001-009
- LORE-STORY-001-010
- LORE-STORY-001-011
- LORE-STORY-001-012
- LORE-STORY-001-013
- LORE-STORY-001-014
- LORE-STORY-001-015
- LORE-STORY-001-016
- LORE-STORY-001-017

## Agents likely needed

- [ ] frontend
- [ ] backend
- [x] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
