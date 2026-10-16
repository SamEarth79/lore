# LORE-STORY-001-017: Evaluation harness and README

## Description

As a judge, I want published metrics on extraction, staleness, groundedness and notification quality, so that the claims are evidence-backed rather than demo-only.

## Acceptance criteria

1. `scripts/eval.py` reports extraction precision/recall against `data/gold/` (match on subject_id + aspect + normalised object), for synthetic and real samples separately.
2. It reports staleness precision/recall and the false-suspect rate on frozen-system facts (target 0) after a full timeline replay.
3. It reports groundedness on ~30 fixed questions (share with path + sources) and not-in-graph correctness on ~10 unanswerable questions.
4. It reports owner load (max/median inbox items per person) and notification precision against gold dependents.
5. README contains: what Lore is, architecture diagram/summary, the MCP tool catalogue and how to connect to `/mcp`, the metrics table (synthetic vs real labelled), the demo script, local setup instructions, the third-party LLM notice and a GitHub-data removal contact.
6. Eval runs with `LLM_DISABLED=1` for staleness/notification metrics (deterministic) and documents which metrics need the live LLM.

## Wireframe

none

## Requirements implemented

- gather.md: Scope: README with extraction precision/recall
- gather.md: Analysis: success metrics

## Depends on

- LORE-STORY-001-009
- LORE-STORY-001-010
- LORE-STORY-001-011

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
