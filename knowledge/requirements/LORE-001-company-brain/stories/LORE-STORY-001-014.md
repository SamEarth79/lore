# LORE-STORY-001-014: Freshness board

## Description

As a judge, I want to see trusted vs suspect facts side by side with the reason for each, so that event-driven staleness is visible at a glance.

## Acceptance criteria

1. `freshness.heading` "Freshness" and `freshness.explainer` "Facts go suspect when what they describe changes after they were last confirmed — not when they get old."
2. `freshness.contrast` "Same age, different fate" pins `freshness.contrast-frozen` (old ledger-batch doc, ✓ Trusted, "Last change to {subject}: {age} (frozen system)") beside `freshness.contrast-churning` (recent payments-api chat, ⚠ Suspect, "{subject} changed {age}: "{event}""), from `contrast_pair`.
3. `freshness.filters` offers `freshness.filter-aspect` (All aspects + the six aspects), `freshness.filter-state` (All states · Suspect · Asking… · Unverified · Nobody knows) and `freshness.filter-real` "Real repo only" (backend: `origin` filter on `GET /freshness`).
4. `freshness.col-trusted` "Trusted ({n})" and `freshness.col-suspect` "Suspect ({n})" list `freshness.fact-card`s with badge, subject/aspect chips, date and a "real repo" chip when applicable; suspect cards show `freshness.reason` "Why" with `freshness.reason-path` in the spec format, plus `freshness.open-source`, `freshness.open-ask` and (asking) `freshness.open-inbox` "View {person}'s inbox".
5. Sub-badge copy: Nobody knows "{p1}, {p2} and {p3} all said "Not me". Still suspect."; Unverified "Low-confidence extraction ({n}%). Queued for confirmation."
6. `freshness.healed` "Recently healed ({n})" lists `freshness.healed-item` "~~{old}~~ → {new} · Corrected by {person} · {date}".
7. After a mutation that flags facts, `freshness.toast-flagged` "{n} facts just became suspect." appears and changed cards are briefly highlighted (respecting reduced motion).
8. Empty/filtered/loading/error: `freshness.empty-suspect` "Nothing is stale. Every fact matches the latest events."; `freshness.empty-trusted` "No trusted facts yet."; `freshness.empty-filtered` "No facts match these filters." with `freshness.clear-filters`; `freshness.loading` skeletons; `freshness.error` "Couldn't load facts." with `freshness.error-retry`.
9. Extremes: statements clamp to 3 lines; columns paginate at 50.

## Wireframe

Spec: `../wireframes/wireframe-spec.md`

- Screen: `freshness` — states: populated, just flagged, empty, filtered empty, loading, error, extremes
  - `freshness.clear-filters`
  - `freshness.col-suspect`
  - `freshness.col-trusted`
  - `freshness.contrast`
  - `freshness.contrast-churning`
  - `freshness.contrast-frozen`
  - `freshness.empty-filtered`
  - `freshness.empty-suspect`
  - `freshness.empty-trusted`
  - `freshness.error`
  - `freshness.error-retry`
  - `freshness.explainer`
  - `freshness.fact-card`
  - `freshness.filter-aspect`
  - `freshness.filter-real`
  - `freshness.filter-state`
  - `freshness.filters`
  - `freshness.heading`
  - `freshness.healed`
  - `freshness.healed-item`
  - `freshness.loading`
  - `freshness.open-ask`
  - `freshness.open-inbox`
  - `freshness.open-source`
  - `freshness.reason`
  - `freshness.reason-path`
  - `freshness.toast-flagged`

## Requirements implemented

- gather.md: Alternate flows — Freshness board
- gather.md: Business rule #3

## Depends on

- LORE-STORY-001-005
- LORE-STORY-001-012

## Agents likely needed

- [x] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
