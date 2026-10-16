# LORE-STORY-001-002: DeepSeek client and tool-calling spike

## Description

As the team, I want a DeepSeek client and a measured tool-calling spike, so that we know how much the Ask agent can rely on the LLM versus the fallback router.

## Acceptance criteria

1. `agent/llm.py` wraps the OpenAI SDK pointed at `https://api.deepseek.com`, model from `DEEPSEEK_MODEL`, thinking disabled, `max_tokens` set, 30s timeout.
2. When `LLM_DISABLED=1`, every call raises a typed `LLMUnavailable` without network access; `/api/health` reports `llm: "disabled"`.
3. A spike script runs ~20 golden questions against stub tools mirroring the catalogue schemas and records: right-tool rate, malformed-argument rate, median and p95 latency.
4. Spike results are recorded in `knowledge/implementations/LORE-001-company-brain/test-results.md` with a go/no-go note on fallback-router reliance; the exact model name the user's key accepts is recorded.
5. Unit tests with mocked responses cover: valid tool call, malformed JSON arguments, unknown tool name, timeout → `LLMUnavailable`. These are marked PASS WITH CAVEATS until the live spike has run.
6. The API key is never logged; logs record only model, latency and token counts.

## Wireframe

none

## Requirements implemented

- gather.md: Technical & integration constraints #3
- gather.md: Deferred: DeepSeek tool-calling reliability

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
