# Gather: Lore — self-healing company brain

## Feature summary
Lore is a company brain built on FalkorDB for Graph Hacks Track 3 ("Company Brain").
It ingests docs, tickets and chat threads into one graph of people, teams, projects,
systems, documents, tickets, chat threads and decisions, where every fact carries the
source it came from. An agent answers questions over that graph using a fixed catalogue
of parameterised Cypher tools (exposed via MCP), and every answer shows the graph path
it followed plus source links — or says explicitly "not in the graph". What makes it a
better company brain is a **self-healing knowledge loop**: facts are flagged suspect when
their subject changes after they were last confirmed (event-driven, not time-based), the
agent asks the one person most likely to know, writes the reply back as a new fact that
supersedes the old one, and notifies only the people whose work depends on the changed
fact. It is for hackathon judges evaluating the track ("the best agent that understands
how a company works and gets work done") and, as a product, for teams whose internal
knowledge goes stale silently.

## User flows

### Primary flow — Ask
1. Visitor opens the deployed app, picks a persona (a synthetic employee) from a switcher.
   A per-visitor sandbox graph is created as a copy of the base graph.
2. On the Ask screen they type a question ("Who owns payments-api?", "Why did we pick
   VendorY?", "What depends on auth-v1?", "What's happening on Project Atlas?").
3. The agent resolves named entities against the graph, picks one or more catalogue
   tools, runs them, and returns: answer text, the graph path (rendered as a small
   Cytoscape graph), and source links (doc/ticket/chat excerpts with author + date).
4. Each fact used in the answer shows its freshness state: trusted / suspect (with the
   triggering event) / open question (asked X, awaiting reply) / unverified / nobody knows.
5. If an entity cannot be resolved or a traversal is empty, the agent replies
   "Not in the graph" and names the closest knower to ask (if any).

### Self-healing loop
1. **Ingest** — new docs/tickets/chats arrive (seeded timeline replay or Ingest-live box).
   Extraction produces facts and change events, each tagged with subject + aspect.
2. **Staleness detection** — a fact becomes *suspect* when a later event matches its
   subject AND aspect, or when a structural event touches its subject (owner moved team,
   system deprecated, decision superseded). Computed as a graph query.
3. **Ask the knower** — for a suspect fact (or an unanswerable question / low-confidence
   fact), the graph ranks knowers (owner, else nearest recent contributor by path) and the
   agent sends the top person a targeted question in their Inbox: Confirm / Correct / Not me.
4. **Escalation** — "Not me" or no reply passes to the next-ranked knower, max 3. While
   pending, answers using the fact show "open question, asked <person>". If all 3 decline,
   the fact is marked "nobody knows" and stays suspect.
5. **Write back** — Confirm re-stamps the fact's last-confirmed time; Correct creates a new
   fact linked to the responder with a timestamp that SUPERSEDES the old fact.
6. **Notify dependents** — on a correction, traverse dependency edges to find people whose
   work relies on the fact and drop a "this changed, here's the source" notice in their Inbox.

### Alternate flows
- **Ingest live**: visitor pastes a ticket/chat message (as their persona) into the Ingest
  screen; extraction runs; the graph, Freshness board and Inbox update.
- **Freshness board**: side-by-side view of trusted vs suspect facts, each with the reason
  path; demo contrast = old doc about a frozen system stays trusted, last week's doc about
  a churning system is flagged.
- **Persona switch**: switching persona shows that persona's Inbox; answering an inbox item
  as that persona drives the write-back.

## Scope boundaries
### In scope
- One FalkorDB graph model covering people, teams, projects, systems, docs, tickets,
  chat threads, decisions, facts, change events, inbox items, agent memory.
- Ingest pipeline with LLM extraction (cached), evidence span + confidence per fact.
- Synthetic company dataset with timestamped change events and a hand-labelled gold set.
- Real-data slice: issues, PRs and maintainers from the public FalkorDB/FalkorDB repo.
- Fixed parameterised Cypher tool catalogue, exposed via an MCP server, used by the agent.
- Entity resolution before any query.
- Answers with path + sources; explicit "not in the graph" outcome.
- Self-healing loop: staleness detection, ask-the-knower, escalation, write-back, notify.
- Screens: Ask (chat + path graph), Inbox (per persona), Freshness board, Ingest live.
- Persona switcher; per-visitor sandbox graph copied from a base graph, resettable.
- Live public deployment.
- README with extraction precision/recall on the gold set.

### Out of scope
- Real Slack / Jira / Google Drive / Notion connectors.
- Slack delivery of inbox items (in-app inbox only).
- Real authentication / user accounts.
- Multi-company / multi-org support.
- LLM-generated Cypher.
- Rate limiting / spend caps (explicitly declined — see decisions).

## Business rules
1. Every fact stored in the graph must link to at least one source (doc/ticket/chat/inbox reply).
2. Every agent answer must include the path and sources for each fact it uses, or the
   explicit "not in the graph" outcome. The agent never asserts a fact not returned by a tool.
3. A fact is *suspect* only when a later event matches both its subject and aspect, or a
   structural event (owner change, deprecation, decision superseded) touches its subject.
   Unrelated activity on the same subject does not flag the fact.
4. Aspect vocabulary is fixed: ownership, dependency, limits/config, api-contract, status, process.
5. Knower ranking: the author of the triggering change first, then the owner of the subject; else
   nearest recent contributor by graph path. The persona who ingested the triggering change is
   never asked to confirm it — the question goes to the next-ranked knower.
6. Escalation: max 3 knowers per question; "Not me" or no reply (timeout) moves to the next.
   All 3 decline → fact state "nobody knows", remains suspect.
7. Confirm re-stamps last-confirmed; Correct creates a superseding fact attributed to the responder.
8. Notifications go only to people reachable via dependency edges from the changed fact's
   subject — never broadcast.
9. Low-confidence extracted facts start as "unverified" and enter the ask-the-knower queue.
10. Each visitor works in their own sandbox graph; actions never mutate the base graph.

## Security & reliability constraints
1. No real auth; personas are synthetic. No real PII — the real-repo slice uses only
   public GitHub data (usernames, public issue/PR text).
2. Agent can only call catalogue tools with typed arguments; entity names resolved against
   the graph before querying (prevents invented names and Cypher injection).
3. Read paths use FalkorDB read-only queries (`ro_query`); only explicit write tools mutate.
4. All Cypher is parameterised — no string interpolation of user input.
5. Input validation on the Ingest-live box and chat input (length cap, type checks) per
   security rules, even though rate limiting was declined.
6. LLM API key provided via environment variable, never committed; `.env` gitignored.
7. Sandbox graphs per visitor must be cleaned up (TTL) to avoid unbounded growth.
8. Failure mode: if the LLM is unavailable, answers fall back to a sentence built from
   tool rows (path + sources still shown).
9. Staleness is a core promise — a stale fact must never be shown as trusted once a
   matching event has been ingested.

## Technical & integration constraints
1. FalkorDB is mandatory and the primary datastore (hackathon rule); use `select_graph`
   for sandboxes and `ro_query` for reads.
2. Backend: Python + FastAPI, falkordb-py client, official MCP Python SDK.
3. LLM: DeepSeek via its OpenAI-compatible API (OpenAI SDK with custom base URL), used for
   the agent (tool selection + answer phrasing) and for ingest extraction (cached).
4. Frontend: React + Vite SPA, Cytoscape.js for path graphs.
5. Hosting: API container on Fly.io or Railway; FalkorDB Cloud or a FalkorDB container.
6. Real data: GitHub public API (issues, PRs, users) for FalkorDB/FalkorDB, fetched once
   and snapshotted to disk.
7. Deadline: submissions close 11:59 PM IST, Oct 18 2026. Team of 2–4.
8. Submission and main FalkorDB implementation must be new work done during the event.

## Key decisions made
- Product name **Lore**, prefix `LORE`.
- Data: synthetic company + real FalkorDB/FalkorDB repo slice.
- Inbox is in-app only (per persona); no Slack.
- Access: open, persona switcher, per-visitor sandbox graph that resets.
- **No rate limits or spend caps** on the public deploy — user accepted the cost risk.
- Escalation to next-ranked knower, max 3, then "nobody knows".
- LLM: DeepSeek (user has an API key), not Claude/OpenAI.
- Stack: Python FastAPI + React/Vite + Cytoscape; Fly or Railway.
- Staleness relevance filter: subject + aspect match from a fixed vocabulary, plus
  structural events.
- Extraction quality: evidence span + confidence per fact, gold set, precision/recall in README.
- Tool catalogue is fixed, parameterised Cypher via MCP — never LLM-written Cypher.

## Deferred / open questions
- Fly.io vs Railway — decide at deploy story time.
- FalkorDB Cloud vs self-hosted container — decide at deploy story time.
- DeepSeek tool-calling reliability is an unverified external-contract assumption; spike on day 1.
- "No reply" timeout value for escalation (demo needs it short, e.g. seconds/minutes, or a
  manual "simulate timeout" control).

## Wireframe decisions
(2026-10-16)
- Ingesting persona is excluded from knower ranking for the change they ingested
  (`ingest.result-asked`, `inbox.question-why-you`).
