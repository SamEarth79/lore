# LORE-STORY-001-012: App shell, persona switcher and Ask screen

## Description

As a visitor, I want a clear app frame showing who I'm acting as, and an Ask screen that shows answers with freshness badges, the path graph and sources, so that I can see how Lore reasons.

## Acceptance criteria

1. App shell: `app-shell.header` shows `app-shell.wordmark` "Lore", `app-shell.nav` with `app-shell.nav-ask` "Ask", `app-shell.nav-inbox` "Inbox" (with `app-shell.nav-inbox-count` hidden at 0), `app-shell.nav-freshness` "Freshness", `app-shell.nav-ingest` "Ingest", `app-shell.persona-switcher` showing "{initials} · {name} · {team} ▾" with `app-shell.persona-unread`, `app-shell.reset` "Reset sandbox", and `app-shell.footer` with the spec's footer copy.
2. Opening the switcher shows `app-shell.persona-list` with one `app-shell.persona-option` per persona (team, role/owns, unread count, "(you)", "good starting point" hint, and "Repo Steward (simulated) · stands in for real FalkorDB maintainers"); selecting one shows "⟳ Switching to {name}…", calls `PUT /api/session/persona`, refetches the view and moves focus to the page heading.
3. `app-shell.reset` opens `app-shell.reset-dialog` with "Reset sandbox?" / "All your answers, ingests and corrections will be discarded."; `app-shell.reset-confirm` resets and shows `app-shell.toast` "Sandbox reset to the original company graph."; `app-shell.reset-cancel` closes the dialog and returns focus.
4. A 409 `sandbox_expired` shows `app-shell.banner-expired` "Your sandbox expired after inactivity." with `app-shell.banner-expired-action` "Start fresh"; a 503 `demo_at_capacity` shows `app-shell.banner-capacity` with the spec copy; a first write shows `app-shell.banner-preparing` "⟳ Preparing your copy of the company…"; a failed health check shows `app-shell.error-unreachable` "Lore's backend is unreachable right now." with `app-shell.error-unreachable-retry`; cold load shows `app-shell.loading` "Loading Lore…".
5. Ask empty: `ask.tagline`, `ask.empty` "Ask Lore anything about how Northwind Labs works.", `ask.suggestions` with the six `ask.suggestion` chips from the spec, `ask.input` (label "Ask a question"), `ask.char-count` "0/500", `ask.submit` "Ask" disabled, and `ask.graph-empty` "The path Lore follows through the graph will appear here."
6. Ask loading: `ask.progress` (aria-live) shows the staged steps and `ask.graph-loading` renders.
7. Ask answered: `ask.answer` with `ask.answer-text` containing `ask.fact-marker`s; `ask.facts` lists each `ask.fact` with `ask.fact-badge` using the global freshness copy from the API `freshness` field (Trusted / Suspect / Asking {person} with `ask.view-as` "View as {person}"); `ask.tools-used` "Tools used ({n})"; `ask.ask-again`; `ask.graph` (Cytoscape, aria-hidden) with `ask.graph-fit`, `ask.graph-reset`, `ask.graph-expand`; `ask.path-text` lists each edge as "{A} —REL→ {B}"; `ask.sources` shows `ask.source` cards with evidence highlighted from server offsets, first 5 then `ask.sources-more` "+{n} more".
8. Not in graph: `ask.not-in-graph` "Not in the graph" with `ask.suggested-knower` "Closest person who may know: {person} ({reason}).", or `ask.not-in-graph-none` when no knower exists. Partial: `ask.did-you-mean` "Couldn't match '{term}'. Did you mean:" with clickable `ask.did-you-mean-option` chips.
9. Fallback: `ask.fallback-notice` "ℹ Answer generated from graph results directly (language model unavailable)." with path and sources rendered. Error: `ask.error` "Something went wrong reaching the graph." with `ask.error-retry`; the question stays in `ask.input`. Over 500 chars: `ask.input-error` "Questions are limited to 500 characters ({n}/500)." and `ask.submit` disabled.
10. Extremes: answer text max ~70ch; path >25 nodes auto-fits; long labels truncated with full name on hover.
11. Rendering contract: no `dangerouslySetInnerHTML`/`innerHTML`; hrefs re-checked for http/https; an XSS payload in a source excerpt renders as inert text (E2E).
12. Accessibility: all controls keyboard-operable with visible focus; badges use icon + text; reduced motion respected.

## Wireframe

Spec: `../wireframes/wireframe-spec.md`

- Screen: `app-shell` — states: populated, switcher open, switching, reset confirm, after reset, sandbox expired, capacity, preparing, backend unreachable, cold load
  - `app-shell.banner-capacity`
  - `app-shell.banner-expired`
  - `app-shell.banner-expired-action`
  - `app-shell.banner-preparing`
  - `app-shell.error-unreachable`
  - `app-shell.error-unreachable-retry`
  - `app-shell.footer`
  - `app-shell.header`
  - `app-shell.loading`
  - `app-shell.nav`
  - `app-shell.nav-ask`
  - `app-shell.nav-freshness`
  - `app-shell.nav-inbox`
  - `app-shell.nav-inbox-count`
  - `app-shell.nav-ingest`
  - `app-shell.persona-list`
  - `app-shell.persona-option`
  - `app-shell.persona-switcher`
  - `app-shell.persona-unread`
  - `app-shell.reset`
  - `app-shell.reset-cancel`
  - `app-shell.reset-confirm`
  - `app-shell.reset-dialog`
  - `app-shell.toast`
  - `app-shell.wordmark`
- Screen: `ask` — states: empty, loading, answered, not in graph, partial, fallback, error, input too long, extremes
  - `ask.answer`
  - `ask.answer-text`
  - `ask.ask-again`
  - `ask.char-count`
  - `ask.did-you-mean`
  - `ask.did-you-mean-option`
  - `ask.empty`
  - `ask.error`
  - `ask.error-retry`
  - `ask.fact`
  - `ask.fact-badge`
  - `ask.fact-marker`
  - `ask.facts`
  - `ask.fallback-notice`
  - `ask.graph`
  - `ask.graph-empty`
  - `ask.graph-expand`
  - `ask.graph-fit`
  - `ask.graph-loading`
  - `ask.graph-reset`
  - `ask.input`
  - `ask.input-error`
  - `ask.not-in-graph`
  - `ask.not-in-graph-none`
  - `ask.path-text`
  - `ask.progress`
  - `ask.source`
  - `ask.sources`
  - `ask.sources-more`
  - `ask.submit`
  - `ask.suggested-knower`
  - `ask.suggestion`
  - `ask.suggestions`
  - `ask.tagline`
  - `ask.tools-used`
  - `ask.view-as`

## Requirements implemented

- gather.md: User flows — Primary flow — Ask
- gather.md: User flows — Persona switch
- gather.md: Business rule #2

## Depends on

- LORE-STORY-001-008

## Agents likely needed

- [x] frontend
- [ ] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
