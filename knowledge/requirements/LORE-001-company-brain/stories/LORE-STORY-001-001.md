# LORE-STORY-001-001: Scaffold, local infra and skeleton deploy

## Description

As the team, I want a running FastAPI + React/Vite skeleton backed by FalkorDB locally and on Railway, so that every later story builds on a verified, deployable base.

## Acceptance criteria

1. `docker compose up` starts the API and a `falkordb/falkordb` container; `GET /api/health` returns `{falkordb: "ok", llm: ...}`.
2. FastAPI serves the built Vite SPA from the same origin; no CORS middleware is installed on the app.
3. Every response (SPA and `/api`) carries the CSP, `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer` headers exactly as specified in architecture.md.
4. `graph/client.py` exposes `ro()` and `rw()` helpers that always pass params and a query timeout; `rw()` raises when targeting `lore_base`. A test proves falkordb-py's params argument works (parameterised query returns the bound value).
5. A test proves `GRAPH.COPY` works on the local FalkorDB and on the deployed Railway FalkorDB (copy a tiny graph, read it, delete it); result recorded in implementation notes.
6. FalkorDB is configured with a password, `maxmemory` and `noeviction`, and is reachable only over the private network on Railway; `TIMEOUT_DEFAULT`/`TIMEOUT_MAX` configured and verified to apply to writes on the deployed version.
7. Dockerfile is multi-stage (Node build → python-slim) and runs as a non-root user.
8. `.env.example` lists `DEEPSEEK_API_KEY`, `DEEPSEEK_MODEL`, `FALKORDB_URL`, `LLM_DISABLED`, `SANDBOX_TTL_S`, `MAX_SANDBOXES`, `ESCALATION_TIMEOUT_S` with placeholders; `.env` is gitignored.
9. A hello-world build is deployed to a public Railway URL and `/api/health` responds there.
10. A generic exception handler returns no stack traces or Cypher; debug is off.

## Wireframe

none

## Requirements implemented

- gather.md: Technical & integration constraints #1, #2, #4, #5
- gather.md: Security & reliability #6
- gather.md: Deferred: Fly vs Railway, FalkorDB Cloud vs container (settled here: Railway + container)

## Depends on

-

## Agents likely needed

- [x] frontend
- [x] backend
- [x] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
