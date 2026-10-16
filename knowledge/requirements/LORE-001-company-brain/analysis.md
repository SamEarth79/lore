# Analysis: Lore — self-healing company brain

## Summary

> Lore is a company brain built on FalkorDB for Graph Hacks Track 3 ("Company Brain").
> It ingests docs, tickets and chat threads into one graph of people, teams, projects,
> systems, documents, tickets, chat threads and decisions, where every fact carries the
> source it came from. An agent answers questions over that graph using a fixed catalogue
> of parameterised Cypher tools (exposed via MCP), and every answer shows the graph path
> it followed plus source links — or says explicitly "not in the graph". What makes it a
> better company brain is a **self-healing knowledge loop**: facts are flagged suspect when
> their subject changes after they were last confirmed (event-driven, not time-based), the
> agent asks the one person most likely to know, writes the reply back as a new fact that
> supersedes the old one, and notifies only the people whose work depends on the changed
> fact.

Six specialist lenses (developer, UI, UX, security, business, outside-the-box) reviewed
`gather.md` independently; this document merges their findings. Converged decisions from
the analyze question round are recorded at the end.

## Relevant existing code

Greenfield. The repo contains only the `.claude/` framework and `knowledge/` (config,
state, this feature's `gather.md`). No `strategy.md`, no prior features, no existing
conventions — every convention is set by this feature: Python 3.12 + FastAPI + Pydantic v2
+ pytest; React + Vite + TypeScript + Vitest; Playwright in `e2e/`.

Confirmed from external docs (developer lens):
- FalkorDB `GRAPH.COPY src dest` / `graph.copy(name)` snapshots nodes, edges, indexes and
  constraints; errors if destination exists (reset = delete then copy).
- `GRAPH.RO_QUERY` rejects writes and accepts a ms timeout; falkordb-py exposes `ro_query`
  and an asyncio client.
- Procedures available: `db.idx.fulltext.createNodeIndex/queryNodes`, `algo.BFS`,
  `algo.SPpaths`, `algo.pageRank`, `algo.WCC`, `algo.betweenness`.
- DeepSeek supports OpenAI-style `tools` at `https://api.deepseek.com`; non-thinking mode
  via `extra_body={"thinking":{"type":"disabled"}}`; current model names per docs are
  `deepseek-flash` / `deepseek-v4-pro` (older `deepseek-chat` no longer listed).

## Constraints and risks

### Build & external dependencies
1. **DeepSeek tool-calling reliability is unverified** (external-contract assumption, per
   `rules/testing.md`). Day-1 spike with ~20 golden questions against the real API.
   Mitigations: deterministic fallback router, server-enforced resolve-first, citation
   validation. Agent tests are `PASS WITH CAVEATS` until the spike runs live.
2. **DeepSeek latency** — multi-tool turns may take 10–30s; fallback covers "down", not
   "slow". Staged progress UI + 30s per-call timeout + ≤4 rounds.
3. **DeepSeek JSON output mode** for extraction not confirmed; parse with Pydantic, retry
   once. Seeded data uses the committed cache.
4. falkordb-py `params=` signature, `CALL db.idx.fulltext.queryNodes` inside `ro_query`,
   and `GRAPH.COPY` on the chosen host are unconfirmed — verified in the scaffold story;
   fallbacks: `CONTAINS` matching, self-hosted container on Railway.
5. FalkorDB variable-length bounds must be literal; queries use `*1..3` filtered by
   `length(p)` and avoid `shortestPath` placement pitfalls.
6. **Dataset authoring is the hidden critical path.** The demo contrast (frozen vs
   churning), knower ranking, escalation and notifications only appear if the synthetic
   company is written backwards from the demo script. Gold labels must be hand-checked.
7. **Scope is large for ~2.5 days** (business + outside-the-box lenses). User chose to keep
   everything, including the real-repo slice in the full loop. Cut order if behind is
   recorded in Open questions.

### Security & abuse (design-security, R1–R23)
8. Sandbox identity must be server-generated (≥128-bit, HttpOnly cookie); graph name never
   a tool/API argument; base graph refuses writes (R1–R2).
9. **Sandbox exhaustion** is the top residual risk with no rate limits. Mitigated by lazy
   copy on first write, 300 LRU cap, 2h TTL, per-sandbox content cap, `noeviction` FalkorDB
   memory policy (R3–R6). A scripted attacker can still churn the LRU and evict live
   sandboxes — accepted residual risk.
10. **LLM spend / key suspension** with no caps: a script can drain the DeepSeek balance or
    get the key suspended during judging. Accepted by user; `LLM_DISABLED` kill switch,
    bounded loop, extraction cache, input caps reduce it. Recommend a low-balance alert in
    the DeepSeek dashboard and key rotation after the event.
11. **Prompt injection** via ingested text and public GitHub issues: agent is read-only;
    all writes deterministic; extraction output schema-validated with exact-substring
    evidence; live free-text facts always start unverified; author is server-set (R8–R11).
12. **Search-syntax injection** in fulltext resolution (RediSearch syntax) — escape input or
    use `CONTAINS` (R12). All Cypher parameterised, bounded, LIMITed, timed out (R13).
13. **XSS** from ingested/GitHub/LLM content — React escaping only, no raw HTML, markdown
    without raw HTML or images, http/https-only links, text-only Cytoscape tooltips, CSP
    (R15–R16). Content never crosses sandboxes; live extraction cache is sandbox-local (R17).
14. **State integrity**: atomic single-query ingest writes, CAS inbox transitions,
    idempotent responses, no correcting a superseded fact (R18–R20).
15. **Third-party data flow**: pasted text goes to DeepSeek; Ingest screen shows a notice;
    logs never contain prompts, full text, or secrets (R21–R23).

**Residual risks accepted by the user (no rate limits/spend caps):** scripted fresh-cookie
writes can churn the LRU and evict every real sandbox, including a judge's mid-demo; each
such ingest costs a DeepSeek call (`LLM_DISABLED` is the only brake); public `/mcp` and
`/api/ask` are unthrottled CPU paths (bounded per query by timeouts and LIMITs).

### Product & business
16. **Owner spam** on busy subjects (judge-flagged): subject+aspect relevance filter, one
    open inquiry per fact, max 3 knowers, notifications ≤2 hops / ≤10 recipients; owner
    load measured in the eval.
17. **"Gets work done" is under-served** if the loop is presented as self-maintenance —
    frame it as routing questions, unblocking the asker and informing affected people.
18. **Real people**: FalkorDB maintainers (possibly judges) must never be personas, inbox
    recipients, or shown as answering — routed to a labelled simulated steward.
19. **Correction authority**: any asked knower can supersede a fact; attributed, not
    reviewed. Worth one honest pitch line.
20. **Compliance**: GitHub snapshot stores usernames, public text excerpts and URLs only (no
    emails); excerpts link back; README notes synthetic vs real metrics and a removal
    contact.

### UX & UI
21. **The loop is invisible without guidance** (highest UX risk): judges arrive cold. A
    guided 6-step tour, suggestion chips, preset ingest events, per-persona unread badges,
    "View as <person>" links and receipts that name who was notified tie the four screens
    together.
22. **Badge consistency is the story**: one server-computed `freshness` field with fixed
    precedence, one `FreshnessBadge` component (icon + label + colour), refetch after every
    mutation via `graphVersion` so no view shows a pre-ingest trusted state.
23. Cytoscape: canvas is not accessible → always pair with a text reason path; call
    `resize()/fit()` when shown; use deterministic `breadthfirst` layout; tokens shared via
    a TS constant.
24. Accessibility: keyboard-operable everything, live regions for answers/receipts, focus
    management on persona switch and inbox actions, reduced motion, 4.5:1 contrast.

### Success metrics (produced by `scripts/eval.py`, reported in README)
- Extraction precision ≥0.8 / recall ≥0.7 on synthetic gold set (~50 facts) + a small
  hand-labelled real sample.
- Staleness precision/recall and false-suspect rate on frozen-system facts (target 0),
  synthetic and real reported separately.
- Groundedness: 100% of ~30 fixed questions carry path + sources; "not in the graph"
  correct on ~10 unanswerable questions.
- Owner load (max/median inbox items per person over a full timeline replay) and
  notification precision.
- Time to heal < 60s live; p95 answer latency; fallback works with `LLM_DISABLED=1`.

## Relation to prior features
None — first feature in this repo.

## Converged decisions (analyze question round)
- Scope: keep everything, including the real-repo slice in the full self-healing loop.
- Sandboxes: lazy copy on first write + 300 LRU cap + 2h TTL + content cap; no per-IP throttle.
- Agent read-only; all writes deterministic; public read-only `/mcp` on `lore_base`.
- Correct = structured pre-filled value + optional note.
- Defaults adopted: knower asked automatically when a fact turns suspect; escalation lazy
  90s timeout + "Simulate no reply"; guided tour in scope; timeline replay as a stepper on
  Ingest; Freshness = Trusted vs Suspect columns with sub-badges + Recently healed strip;
  refetch after mutation (no streaming); real maintainers never contacted.

## Open questions (non-blocking)
- Cut order if behind schedule: (1) gold-set size → 30, (2) real-slice staleness eval
  depth, (3) graph Expand modal / board filters, (4) MCP polish beyond a working mount.
- Fly vs Railway / FalkorDB Cloud vs container — default Railway + container, settled in
  the scaffold story.
- Exact DeepSeek model name the user's key accepts.
- Whether judges evaluate primarily via the live deploy or the video (affects polish split).
- Dark mode — non-goal; light theme only.
