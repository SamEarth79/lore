# LORE-STORY-001-008: Ask agent endpoint

## Description

As a visitor, I want to ask a question in plain English and get an answer built only from graph facts, with the path and sources, or an honest "not in the graph".

## Acceptance criteria

1. `POST /api/ask {question 1..500}` returns `{answer, outcome, facts[], path, sources[], suggested_knower?, did_you_mean[], tool_trace[]}`; questions over 500 chars are rejected with 422.
2. The agent has read tools only, runs ≤4 tool rounds at temperature 0, and its system prompt marks tool results as untrusted data.
3. Resolve-first is enforced server-side: tools taking `entity_id` reject ids not returned by `resolve_entity` in this request or in session memory.
4. Citation validation: the answer may cite only fact ids present in this request's evidence set; any foreign id discards the LLM text and uses the fallback template. Path and sources are built from tool rows, never from LLM text.
5. No usable facts → `outcome: not_in_graph` with `suggested_knower` from `find_knowers` on the nearest resolved entity, or `did_you_mean` candidates when nothing resolved.
6. LLM error/timeout or `LLM_DISABLED=1` → deterministic intent router + templates produce `outcome: fallback` with path and sources still populated.
7. Session memory stores turns in the sandbox (pruned to 20) or, without a sandbox, in an in-process LRU capped at 1,000 vids × 5 turns; `/ask` never creates a sandbox.
8. Tests: owner question answered with path+sources; made-up system → not_in_graph; foreign citation → fallback; LLM disabled → fallback; an ingested source containing "ignore instructions" text does not change the agent's tool permissions. LLM-mocked tests are PASS WITH CAVEATS until story 002's live spike passes.

## Wireframe

none

## Requirements implemented

- gather.md: User flows — Primary flow — Ask
- gather.md: Business rule #2
- gather.md: Security & reliability #2, #8

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
