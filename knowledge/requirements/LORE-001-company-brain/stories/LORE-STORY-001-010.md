# LORE-STORY-001-010: Self-healing inbox backend

## Description

As a knower, I want to confirm, correct or decline Lore's questions, so that facts are re-stamped or superseded, escalation continues, and only dependents are notified.

## Acceptance criteria

1. `GET /api/inbox` returns `{questions[], notices[]}` for the current persona; without a sandbox it never writes and computes overdue state at read time; with a live sandbox it lazily expires questions whose `asked_at_wall + ESCALATION_TIMEOUT_S` has passed.
2. `POST /api/inbox/{id}/respond {action, value?, note?}` validates `^inbox:[a-z0-9-]{1,64}$`; each action is a single Cypher CAS statement including `NOT f.superseded`; a lost race returns 409 with the current fact.
3. Confirm sets `last_confirmed_at` from the sandbox clock, clears inquiry, adds `CONFIRMED_BY`, stores the reply as a source, and closes other pending items on the fact.
4. Correct (`value` 1..300, `note` ≤1000) creates a trusted superseding fact asserted by the responder and sourced from the reply, marks the old fact superseded, writes a same-aspect change event, and notifies dependents (OWNS/WORKS_ON within 2 DEPENDS_ON hops, excluding the responder, max 10) with a reason path per notice. Real-repo corrections are labelled "simulated correction (sandbox)".
5. Not me / `POST /api/inbox/{id}/simulate-timeout` declines/expires the item and, in the same statement, creates the next-ranked knower's item (max 3 asked, no duplicate pending item); after 3 declines the fact's inquiry becomes `nobody_knows` and stays suspect; a later matching event or Confirm clears it.
6. Steward-mapped ranks are merged into a single ask.
7. Response returns `{fact, superseded_fact?, escalated_to?, notified[], graphVersion}`.
8. Tests: two concurrent Corrects produce exactly one superseding fact; timeout racing a Confirm leaves only the Confirm; 3 declines → nobody_knows; notification recipients match the gold dependents for payments-api; responder never notified.

## Wireframe

none

## Requirements implemented

- gather.md: Self-healing loop steps 3–6
- gather.md: Business rules #5, #6, #7, #8

## Depends on

- LORE-STORY-001-009

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
