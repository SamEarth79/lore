# Architecture: Lore — self-healing company brain (LORE-001)

> Drafted from `design-developer`'s proposal, revised with the converged answers from
> the analyze round and `design-security` requirements R1–R23. Signed off by
> `design-security` before being written to disk.

## Approach

Lore is one Python FastAPI service backed by one FalkorDB instance, plus a React/Vite SPA
served by FastAPI from the same origin (one container, no CORS, same-origin cookies). All
state lives in FalkorDB. A **base graph** `lore_base` is built deterministically at deploy
time by `scripts/seed_base.py` from committed inputs: the synthetic company, a committed
GitHub snapshot of FalkorDB/FalkorDB, and a committed LLM extraction cache — so seeding
needs no live LLM call. The base graph is never written by request handlers.

Visitors get a server-generated id in an HttpOnly cookie. Reads go to `lore_base` via
`ro_query` until the visitor's **first write** (ingest, inbox action, timeline step), which
copies the base into a sandbox graph `sbx_<id>` (`GRAPH.COPY`). Sandboxes are tracked in a
`lore_registry` graph, expire after 2h idle, and are capped at 300 (LRU eviction). The graph
handle is always resolved from the request context — never from a tool/API argument.

Every graph read goes through one **tool catalogue**: typed, parameterised Cypher functions
in a single Python registry returning a common envelope `{rows, facts, path, sources}`. The
same registry backs (a) the Ask agent (in-process), (b) a public read-only MCP endpoint
`/mcp` (official MCP Python SDK, FastMCP, streamable HTTP) bound to `lore_base`, and (c) REST
handlers. The **Ask agent is read-only**: it runs a bounded DeepSeek tool-calling loop (≤4
rounds, temperature 0, thinking disabled), must cite fact ids returned by tools (validated
server-side), and returns "Not in the graph" when no tool produced a usable fact. If the LLM
fails, times out, or `LLM_DISABLED=1`, a deterministic intent router + templates answer from
tool rows; path and sources still render.

**All writes are deterministic server code**, triggered by UI actions or the ingest
pipeline: creating knower questions, Confirm/Correct/Not me, escalation, supersession,
dependent notifications. The LLM never chooses a recipient, a target fact, or a state
transition.

**Staleness is derived at read time, not stored.** A fact's freshness is computed in a
shared Cypher fragment (`FRESHNESS`) used by every tool: a fact is suspect iff there exists
a ChangeEvent touching its subject, occurring after the fact's `last_confirmed_at`, that is
structural or matches the fact's aspect, and is not from the fact's own source. This makes
gather security constraint 9 hold by construction — there is no flag to drift. After each
ingest, the same predicate scoped to the touched subjects identifies newly suspect facts so
knower questions can be created.

## Components touched

Greenfield — everything is new.

- **Frontend** (`web/`, React + Vite + TypeScript, Cytoscape.js, lucide-react; plain CSS
  with tokens, no component library):
  - `AppShell` (nav: Ask / Inbox (count) / Freshness / Ingest; `PersonaSwitcher` with
    per-persona unread badges; Reset sandbox), `GuidedTour` step strip (6 steps, skippable,
    progress in localStorage wrapped in try/catch).
  - `AskPage`: chat, `SuggestionChips`, `AgentProgress`, `PathGraph` (Cytoscape,
    breadthfirst layout) + text `ReasonPath` alternative, `SourceList`/`SourceCard` with
    evidence highlighting via server offsets, `FreshnessBadge`, `NotInGraphCard`.
  - `InboxPage`: `InboxQuestionItem` (Confirm / Correct structured field + note / Not me /
    Simulate no reply), `InboxNoticeItem` (old→new, dependency reason path, mark read).
  - `FreshnessPage`: Trusted vs Suspect columns (suspect column carries sub-badges for open
    question / unverified / nobody knows), pinned frozen-vs-churning contrast pair,
    "Recently healed" strip, reason breadcrumb per card.
  - `IngestPage`: preset example events, free-text box (as current persona), "Replay next
    timeline event" stepper, extraction result diff (facts + evidence + confidence, events,
    flagged facts, knower asked), third-party-LLM notice.
  - `api.ts` fetch client (credentials included); a global `graphVersion` returned by every
    mutation triggers refetch of visible views.
  - **Rendering contract (R15)**: no `dangerouslySetInnerHTML`/`innerHTML` anywhere. Answer
    text renders as plain text (no markdown; if ever added: raw HTML and images disabled).
    Every `href` is re-checked client-side against `http:`/`https:` in addition to the server
    check. Evidence highlighting slices `excerpt` at server offsets into React text nodes +
    `<mark>` elements — never an HTML string. Cytoscape labels/tooltips/popovers are
    text-only. Content sources treated as untrusted: visitor pastes, GitHub issue bodies,
    LLM output, correction notes.
- **Backend** (`api/lore/`):
  - `main.py` (FastAPI app, routers, SPA static mount, `/mcp` mount, startup: index creation,
    seed-if-missing, orphan sweep, sweeper task), `config.py` (env settings).
  - **Security headers middleware (R16)** on every response:
    `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'
    'unsafe-inline'; img-src 'self' data:; connect-src 'self'; object-src 'none'; base-uri
    'none'; frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`,
    `Referrer-Policy: no-referrer`. No CORS middleware on the app; if browser-based MCP
    inspectors need it, CORS is scoped to `/mcp` only with `allow_credentials=False`.
  - `graph/client.py` (FalkorDB connection; `ro()`/`rw()` helpers that always pass params
    and a query timeout; `rw()` refuses `lore_base`), `graph/schema.py` (indexes),
    `graph/sandbox.py` (resolve, lazy copy, touch, reset, sweep, cap, existence check).
  - `tools/` (registry, read tools, Pydantic arg models, envelope, shared `FRESHNESS` and
    `FACT_PROJECTION` Cypher fragments).
  - `mcp_server.py` (FastMCP from the registry, read tools only, `lore_base` only, mounted
    with `stateless_http=True` so anonymous clients accumulate no server-side session state).
  - `agent/` (`llm.py` OpenAI SDK → DeepSeek, `loop.py`, `prompts.py`, `fallback.py`,
    `citations.py`).
  - `ingest/` (`extract.py` LLM JSON extraction + cache, `validate.py`, `writer.py`).
  - `healing/` (`staleness.py`, `knowers.py`, `inbox.py`, `writeback.py`, `notify.py`).
  - `routes/` (session, ask, inbox, freshness, ingest, timeline, entity, health, personas).
- **Data & scripts**: `data/synthetic/` (entities, sources, timeline), `data/github/
  snapshot.json` (no emails), `data/cache/extract/*.json`, `data/gold/*.json`;
  `scripts/fetch_github.py`, `scripts/seed_base.py`, `scripts/eval.py` (extraction P/R,
  staleness P/R + false-suspect rate, groundedness, owner load, notification precision).
- **Infrastructure**: multi-stage `Dockerfile` (Node build → python-slim, non-root user),
  `docker-compose.yml` (api + `falkordb/falkordb`) for local, `.env.example`, `.gitignore`
  (`.env`), Railway deploy (API service + FalkorDB container on private networking with a
  volume, password set, `maxmemory` + `noeviction`, not publicly exposed). Single API
  instance. FalkorDB Cloud only if `GRAPH.COPY` is confirmed there.

## Data flow

**A. Session (no copy yet)**
1. SPA calls `POST /api/session`. If no `lore_vid` cookie: server sets one with
   `secrets.token_hex(16)` (HttpOnly, Secure, SameSite=Lax). Default persona preselected.
2. `sandbox.resolve(vid)` → if registry has a live `sbx_<vid>` use it, else the read target
   is `lore_base`. Name must match `^sbx_[0-9a-f]{32}$`.

**B. First write → lazy copy** (only POST/PUT routes; **GET routes never call
`ensure_writable`**)
1. `sandbox.ensure_writable(vid)` runs under a per-vid `asyncio.Lock` (single API
   instance), so concurrent first writes and sweeper/eviction cannot interleave with it:
   - If a live registry row exists: verify the graph exists and the per-sandbox node-count
     ceiling (3× base node count + margin) is not exceeded (→ 413 for all writes, including
     inbox responses); touch `last_seen`.
   - If a tombstone row exists (evicted/expired, kept until TTL passes): return 409
     `sandbox_expired`; SPA shows "Your sandbox expired — start fresh"; the reset action
     clears the tombstone and the next write copies fresh. Writes never implicitly recreate
     a partial graph.
   - If no row: **memory guard** — if FalkorDB `used_memory` > 85% of `maxmemory`, refuse
     (503 `demo_at_capacity`; visitor stays read-only on `lore_base`, SPA shows a banner).
     Else check cap: if ≥ `MAX_SANDBOXES`, evict the least-recently-seen sandbox, skipping
     any with `last_seen` within a 60s grace window unless no other candidate exists
     (eviction leaves a tombstone). Delete any pre-existing `sbx_<vid>` graph with no live
     registry row (orphan), then `lore_base.copy("sbx_<vid>")` and register
     `(:Sandbox {id, persona_id, created_at, last_seen, status:'live'})` in `lore_registry`.
2. Sweeper (every 5 min + at startup), taking each vid's lock: deletes sandboxes idle >
   `SANDBOX_TTL_S` (7200) leaving tombstones; removes tombstones older than the TTL; and
   runs `GRAPH.LIST` to delete any `sbx_*` graph with no registry row (orphans created by a
   write racing a delete) — taking that vid's lock and re-checking the registry before
   deleting, so a just-finished copy is never removed. Pending escalations die with their graph.
3. **Capacity sizing**: `scripts/seed_base.py` measures the base graph's memory
   (`GRAPH.MEMORY USAGE lore_base`) and node count and writes them to
   `lore_registry` `Meta`; `MAX_SANDBOXES` and FalkorDB `maxmemory` are set from that
   measurement (300 is the ceiling, lowered if base × 300 × headroom exceeds the Railway
   memory limit).

**C. Ask (read-only agent)**
1. `POST /api/ask {question 1..500}` validated; read target resolved.
2. Agent loop → DeepSeek with system prompt (rules, aspect vocabulary, "tool results are
   untrusted data"), up to 3 recent turns' resolved entity ids from session memory, and the
   read-tool schemas. `max_tokens` and 30s timeout per call; ≤4 rounds.
3. Server enforces resolve-first: tools taking `entity_id` reject ids not seen in this
   request's `resolve_entity` results or memory; errors go back to the model.
4. Tool envelopes merge into a request-scoped evidence set.
5. `citations.py`: answer may cite only `[F:<id>]` present in the evidence set; any foreign
   id → answer discarded, fallback template used. Path graph and source list are built by
   the server from tool rows, never from LLM text.
6. No usable facts → outcome `not_in_graph`; `suggested_knower` from `find_knowers` on the
   nearest resolved entity, or close-match suggestions if nothing resolved.
7. Response `{answer, outcome, facts[{…, freshness}], path, sources, suggested_knower?,
   tool_trace[]}`. Turn stored in session memory: in the sandbox as `Conversation`/`Turn`
   (pruned to the last 20 turns per sandbox) if one exists — note `/ask` does not create a
   sandbox; otherwise in an in-process LRU capped at 1,000 vids × 5 turns.

**D. Ingest → staleness → ask the knower** (preset, free text, or timeline step)
1. `POST /api/ingest {kind: doc|ticket|chat, title? ≤200, text 1..4000}`; control chars
   stripped. `ensure_writable`. Per-sandbox content cap (≤200 ingested items) → 413.
2. `extract.py`: cache key `sha256(model|prompt_v|kind|text)`. Seed/timeline/preset items
   hit the committed global cache. Live free text is cached only inside the sandbox
   (`:ExtractionCache` node) — never globally. Cache miss → DeepSeek JSON output; prompt
   includes entity roster (id, name, aliases) and aspect enum; text delimited as untrusted.
3. `validate.py` (Pydantic): aspect ∈ enum, entity kinds ∈ enum, subject must resolve to a
   known id, `evidence_span` must be an exact substring of the text, lengths capped,
   confidence clamped 0..1, ≤10 facts and ≤5 events per item. **Facts from live free-text
   ingest always start `unverified`** regardless of model confidence; seeded/preset facts
   use confidence <0.6 → unverified. Author = current persona (server-set), never extracted.
4. `writer.py`: one Cypher query writes Source + Facts + ChangeEvents + edges atomically.
   Structural events fan out TOUCHES to affected subjects (owner move → person + things they
   own; decision superseded → old decision + its subjects).
5. `staleness.newly_suspect(event_ids)` runs the FRESHNESS predicate for facts on touched
   subjects; `knowers.rank` + `inbox.ask` create one question per newly suspect/unverified
   fact that has no open inquiry (CAS: `WHERE NOT EXISTS open inquiry`).
6. Response `{source_id, facts_created, events, flagged_facts, inbox_items_created,
   graphVersion}`. `POST /api/timeline/next` does the same with the next scripted item and
   advances the sandbox clock (`Meta {key:'clock'}`).

**Clocks.** Every domain timestamp in a sandbox — `last_confirmed_at`, a correction Fact's
`created_at`, the correction ChangeEvent's `occurred_at`, live-ingest and timeline
`occurred_at` — comes from the sandbox clock `Meta {key:'clock', now}`. The clock never
moves backwards and each write advances it by ≥1s (`now = max(now + 1, scripted_time)`),
so a confirmation can never be dated after events replayed later (preserves gather
constraint 9). Escalation timeouts use a separate wall-clock field `asked_at_wall` compared
against `ESCALATION_TIMEOUT_S`. `lore_base` holds its own frozen clock; seeded questions
in the base (if any) carry `asked_at_wall = null`, and `asked_at_wall` is stamped on copy,
so they never appear expired the moment they are copied.

**E. Inbox → write-back → notify**
1. `GET /api/inbox` never writes when no sandbox exists — overdue state is computed at read
   time from `asked_at_wall`. When a live (non-tombstone) sandbox exists, lazy expiry runs (CAS per item:
   decline-by-timeout and create the escalated item in the same query).
2. `POST /api/inbox/{id}/respond {action, value?, note?}` (`id` pattern-validated). Each
   action is **one Cypher statement** whose MATCH carries the full CAS condition:
   `MATCH (i:InboxItem {id:$id, status:'pending'})-[:FOR]->(:Person {id:$persona}),
   (i)-[:ABOUT_FACT]->(f:Fact) WHERE NOT f.superseded …` — zero rows → 409 "already
   resolved" with the current fact (idempotent; timeout loses cleanly to a reply; two
   concurrent Corrects cannot both supersede, since FalkorDB serialises writes per graph).
   - **Confirm**: `f.last_confirmed_at = clock`, `inquiry = none`, `CONFIRMED_BY` edge,
     reply stored as `InboxReply` source, other pending items on the fact closed.
   - **Correct** (`value` 1..300 structured new value for the fact's predicate, `note`
     ≤1000 optional): new Fact (`trusted`, confidence 1.0, `ASSERTED_BY` responder,
     `SOURCED_FROM` the InboxReply, same subject/aspect/predicate, `created_at =
     last_confirmed_at = clock`) `SUPERSEDES` old; old `superseded=true`; a content
     ChangeEvent (same aspect, `occurred_at = clock`) from the reply so related facts
     re-evaluate. Real-slice supersessions are labelled "simulated correction (sandbox)".
     Then `notify.dependents()`.
   - **Not me / simulate-timeout**: in the same statement, item → declined/expired and, if
     asked < 3 and `knowers.rank(exclude=asked)` (computed just before, passed as params)
     returns a candidate and no other pending item exists for the fact, a new item rank+1
     `ESCALATED_FROM` old; else fact inquiry
     `nobody_knows` (still suspect). A later matching event or Confirm clears it. When
     multiple ranks map to the simulated steward, they are merged into one ask.
   - Question creation after ingest is best-effort: if it fails, freshness is still correct
     (derived) and the fact surfaces on the Freshness board; accepted gap.
3. `notify.dependents()`: people who OWN/WORK_ON the subject or anything that DEPENDS_ON it
   within 2 hops, excluding the responder, max 10 recipients, each notice carrying its
   dependency reason path.

**F. Real-repo slice**
- GitHub issues/PRs become `Ticket {origin:'github'}` sources; maintainers become
  `Person {origin:'github', is_persona:false}`; FalkorDB components become Systems. They run
  through the same extraction and staleness logic.
- Real people are **never personas and never receive inbox items**. When a knower ranking
  for a real-slice fact lands on a github person, the item is routed to the simulated
  persona "Repo Steward (simulated)" and shows "Suggested knower: @handle (not contacted)".
  Real people are never shown as having confirmed/corrected anything.

## Data model changes

FalkorDB property graph; same schema in `lore_base` and every sandbox. Times are integer
epoch seconds. Ids are string slugs (`sys:payments-api`, `person:priya`).

**Entities** (each also labelled `:Entity {id, name, alias_text, kind}`; fulltext index on
`Entity(name, alias_text)`; range index on `Entity.id`):
`Person {handle, origin: synthetic|github, is_persona, title}`, `Team`,
`Project {status: active|paused|done}`, `System {status: active|frozen|deprecated}`,
`Decision {title, status: active|superseded, decided_at}`.

**Sources** (each also `:Source {id, url, author_id, created_at, excerpt, kind}`; range index
on id): `Document {title}`, `Ticket {title, status, origin, number}`,
`ChatThread {channel}`, `InboxReply {inbox_item_id, action}`. `url` validated server-side
to http/https only.

**Knowledge**
- `Fact {id, subject_id, aspect: ownership|dependency|limits_config|api_contract|status|
  process, predicate, object_text, object_id?, statement, confidence, evidence_span,
  evidence_start, evidence_end, extracted_by: llm|seed|human, created_at,
  last_confirmed_at, base_state: trusted|unverified, superseded: bool,
  inquiry: none|open|nobody_knows}` — range indexes on `id`, `subject_id`.
- **Derived `freshness`** (computed in `FACT_PROJECTION`, never stored), precedence:
  `superseded` > `nobody_knows` > `open_question` > `suspect` > `unverified` > `trusted`.
  Answers never use superseded facts; superseded appears only in history, notices and
  "Recently healed".
- `ChangeEvent {id, kind: content|structural, structural_type: owner_change|deprecation|
  decision_superseded|null, aspect?, subject_id, occurred_at, summary}`.
- `InboxItem {id, kind: question|notice, status: pending|confirmed|corrected|declined|
  expired|unread|read, rank 1..3, question_text, asked_at, asked_at_wall, answered_at?,
  suggested_real_knower?}`.
- `Conversation {id, persona_id, created_at}`, `Turn {id, question, answer, outcome, at}`.
- `ExtractionCache {key, payload_json}` (sandbox-only, for live free text).
- `Meta {key:'clock', now}`.
- `Sandbox {id, persona_id, created_at, last_seen, status: live|tombstone}` (in `lore_registry` only).

**Relationships**
- Org: `(Person)-[:MEMBER_OF]->(Team)`, `(Person|Team)-[:OWNS {since}]->(System|Project)`,
  `(Person)-[:WORKS_ON]->(Project)`, `(Person)-[:LEADS]->(Team)`.
- Dependencies: `(Project|System)-[:DEPENDS_ON]->(System)`.
- Decisions: `(Decision)-[:ABOUT]->(Entity)`, `(Decision)-[:SUPERSEDES]->(Decision)`,
  `(Person)-[:DECIDED]->(Decision)`.
- Provenance: `(Person)-[:AUTHORED]->(Source)`, `(Source)-[:MENTIONS]->(Entity)`,
  `(Fact)-[:SOURCED_FROM]->(Source)` (≥1 per fact, asserted by seed check and writer).
- Facts: `(Fact)-[:ABOUT]->(Entity)`, `(Fact)-[:SUPERSEDES]->(Fact)`,
  `(Fact)-[:CONFIRMED_BY {at}]->(Person)`, `(Fact)-[:ASSERTED_BY]->(Person)`.
- Events: `(ChangeEvent)-[:TOUCHES]->(Entity)`, `(ChangeEvent)-[:FROM]->(Source)`.
- Inbox: `(InboxItem)-[:FOR]->(Person)`, `(InboxItem)-[:ABOUT_FACT]->(Fact)`,
  `(InboxItem)-[:ESCALATED_FROM]->(InboxItem)`, `(InboxItem)-[:TRIGGERED_BY]->(ChangeEvent)`.
- Memory: `(Conversation)-[:HAS_TURN]->(Turn)`, `(Turn)-[:RESOLVED]->(Entity)`,
  `(Turn)-[:USED_FACT]->(Fact)`.

**FRESHNESS predicate** (shared fragment; also gives the reason event):
```cypher
OPTIONAL MATCH (e:ChangeEvent)-[:TOUCHES]->(:Entity {id: f.subject_id})
WHERE e.occurred_at > f.last_confirmed_at
  AND (e.kind = 'structural' OR e.aspect = f.aspect)
  AND NOT (e)-[:FROM]->(:Source)<-[:SOURCED_FROM]-(f)
WITH f, e ORDER BY e.occurred_at DESC
WITH f, head(collect(e)) AS reason_event
```

## API surface

### REST (`/api`; read target / sandbox from the `lore_vid` cookie; Pydantic validation)

| Method | Path | Body / query | Returns |
|---|---|---|---|
| POST | `/session` | `{persona_id?}` | `{persona, has_sandbox}` |
| POST | `/session/reset` | – | drops sandbox (next write recopies) |
| PUT | `/session/persona` | `{persona_id}` (`is_persona` only; never triggers a copy — stored in the in-process LRU until a sandbox exists) | persona |
| GET | `/personas` | – | `[{id,name,title,team,owns[],good_start,unread}]` |
| POST | `/ask` | `{question 1..500}` | `{answer, outcome: answered\|not_in_graph\|fallback, facts[], path, sources[], suggested_knower?, did_you_mean[], tool_trace[]}` |
| GET | `/inbox` | – (lazy expiry first) | `{questions[], notices[]}` for current persona |
| POST | `/inbox/{id}/respond` | `{action: confirm\|correct\|not_me, value? 1..300, note? ≤1000}` | `{fact, superseded_fact?, escalated_to?, notified[], graphVersion}` |
| POST | `/inbox/{id}/simulate-timeout` | – | as not_me |
| GET | `/freshness` | `?verify=0\|1&origin=synthetic\|github` | `{trusted[], suspect[{fact, reason_event, reason_path}], recently_healed[], contrast_pair}` |
| POST | `/ingest` | `{kind, title? ≤200, text 1..4000}` | `{source_id, facts_created[], events[], flagged_facts[], inbox_items_created[], graphVersion}` |
| GET | `/ingest/presets` | – | preset example events |
| GET | `/timeline` | – | `{clock, next_item_preview, remaining}` |
| POST | `/timeline/next` | – | as `/ingest` |
| GET | `/entity/{id}` | id `^[a-z]+:[a-z0-9-]{1,64}$` | 1-hop neighbourhood |
| GET | `/health` | – | `{falkordb, llm: ok\|degraded\|disabled}` |

All path ids are pattern-validated (`persona_id` `^person:[a-z0-9-]{1,64}$`, inbox ids
`^inbox:[a-z0-9-]{1,64}$`, entity ids as above). The committed global extraction cache is
read-only at runtime; a cache miss never writes to it. `GITHUB_TOKEN` is used only by
`scripts/fetch_github.py` locally and is never set in the deployed environment. FalkorDB
`TIMEOUT_DEFAULT`/`TIMEOUT_MAX` are configured so timeouts apply to writes as well (verified
against the deployed version in the scaffold story). Where the container supports ACLs, the
app user runs with `-@dangerous -flushall -flushdb -config -keys`.
Generic exception handler: no stack traces or Cypher in responses; debug off. Logs record
ids, lengths, tool names and timings only — never prompts, full text, or secrets.

### MCP tool catalogue

All read tools: `graph.ro_query(cypher, params, timeout=2000)`, bounded variable-length
hops, explicit `LIMIT`, `entity_id` validated against `^[a-z]+:[a-z0-9-]{1,64}$` and must
exist. Labels/relationship types come from fixed enums mapped to hard-coded strings — never
interpolated. Public `/mcp` exposes these tools against `lore_base` only; no graph/sandbox
argument exists on any tool.

| # | Tool | Params | Cypher sketch |
|---|---|---|---|
| 1 | `resolve_entity` | `name ≤100`, `kinds?: enum[]` | Escape to word chars + spaces, keeping `-` only between word chars (no leading-`-` negation); empty after escaping → skip to fallback. `CALL db.idx.fulltext.queryNodes('Entity', $q)` → top 5 by score filtered by kinds; fallback `toLower(n.name) CONTAINS toLower($name)` |
| 2 | `get_entity` | `entity_id` | node + ≤25 1-hop neighbours |
| 3 | `get_owner` | `entity_id` | `OWNS` owner (team → lead), ownership facts w/ FACT_PROJECTION |
| 4 | `get_facts` | `entity_id`, `aspect?`, `include_history=false` | facts ABOUT entity w/ FACT_PROJECTION, sources, reason event, open inquiry person |
| 5 | `get_dependents` | `entity_id`, `depth 1..3=2` | `(d)-[:DEPENDS_ON*1..3]->(s)` filtered by `length(p) <= $depth`, people who own/work on d |
| 6 | `get_dependencies` | `entity_id`, `depth 1..3` | reverse of 5 |
| 7 | `get_decisions` | `entity_id` | decisions ABOUT entity, `SUPERSEDES*0..3` chain, deciders, facts |
| 8 | `get_project_activity` | `project_id`, `since_days 1..90=30` | members, recent sources (≤10), recent events (≤10) |
| 9 | `get_person` | `person_id` | teams, owns, works on |
| 10 | `find_knowers` | `entity_id`, `exclude_ids ≤10`, `limit 1..3=3` | the persona who ingested the triggering change is always in `exclude_ids`; tiers: author of the latest reason event → owner (team → lead) → contributors who authored sources mentioning the entity within 90 days → contributors via 1 hop DEPENDS_ON/OWNS; ordered (tier, hops, -recency) |
| 11 | `get_path` | `from_id`, `to_id` | `shortestPath((a)-[:OWNS\|MEMBER_OF\|WORKS_ON\|DEPENDS_ON\|LEADS\|DECIDED\|ABOUT*1..4]-(b))` — relationship-type whitelist excludes hub edges (MENTIONS, SOURCED_FROM, AUTHORED); falls back to `algo.SPpaths` if placement rules require |
| 12 | `get_freshness` | `aspect?`, `state?` | non-superseded facts with derived freshness and reason event |

Internal write functions (not LLM tools, not on MCP; all `rw()` parameterised, refuse
`lore_base`): `ingest_source`, `ask_knower`, `respond_inbox`, `expire_overdue`,
`notify_dependents`, `sandbox_copy/reset/sweep`.

## Wireframe reconciliation

- 2026-10-16 (additive): `find_knowers` always excludes the persona who ingested the
  triggering change (`ingest.result-asked`); `GET /personas` adds `owns[]` and `good_start`
  (`app-shell.persona-option`); `GET /freshness` adds an `origin` filter
  (`freshness.filter-real`); `POST /ask` returns `did_you_mean[]` (`ask.did-you-mean`).
  No core sections changed; security gate not re-run.

## Key decisions

- **Decision**: One tool registry used in-process by the agent, by REST, and mounted
  read-only at public `/mcp` on `lore_base`.
  **Rationale**: Byte-identical Cypher across surfaces; no internal network hop; judges can
  connect Claude Desktop/Inspector to `/mcp`. Agent-over-MCP transport rejected as cost
  with no visible benefit.
- **Decision**: Agent is read-only; all writes are deterministic server code.
  **Rationale**: Contains prompt injection from ingested text and public GitHub content
  (security AB-5); the loop's routing is graph logic, not LLM judgement.
- **Decision**: Freshness derived at read time via one shared Cypher fragment.
  **Rationale**: Gather constraint 9 holds by construction (no stored flag to drift);
  ingest atomicity (single-query write) means events and facts land together.
- **Decision**: Lazy sandbox copy on first write, 300-sandbox LRU cap, 2h idle TTL,
  per-sandbox content cap, registry graph, expired-sandbox 409 instead of implicit recreate.
  **Rationale**: Bounds memory without rate limits (user declined rate limits/spend caps);
  crawlers and read-only visitors never create graphs.
- **Decision**: Lazy escalation timeout (90s) + "Simulate no reply"; no scheduler.
  **Rationale**: Demoable in seconds, no worker process, CAS makes races safe.
- **Decision**: Correct takes a structured value (pre-filled) + optional note.
  **Rationale**: Deterministic write-back for the key demo moment; no extraction misread.
- **Decision**: Committed extraction cache; base graph built offline and deterministically.
  **Rationale**: Deploy independent of LLM, reproducible eval, zero spend on seeded data.
- **Decision**: Real-repo maintainers are never personas or inbox recipients; questions
  route to a labelled simulated steward.
  **Rationale**: Avoids impersonating real people (possibly judges) while letting the real
  slice run through the full loop.
- **Decision**: Knower order = reason-event author → owner → recent contributors (90d) →
  1-hop contributors; max 3 asked; notifications ≤2 hops, ≤10 recipients.
  **Rationale**: Event author usually knows the new truth; bounds prevent owner spam and
  broadcast-by-dependency.
- **Decision**: DeepSeek via OpenAI SDK, model from env (`DEEPSEEK_MODEL`), thinking off;
  `LLM_DISABLED` kill switch forces fallback.
  **Rationale**: User's provider; kill switch stops cost without a spend cap.
- **Decision**: Railway with a FalkorDB container on private networking (`noeviction`,
  password, volume), single API instance.
  **Rationale**: `GRAPH.COPY` availability on FalkorDB Cloud unconfirmed; private network
  satisfies security rule 8.
