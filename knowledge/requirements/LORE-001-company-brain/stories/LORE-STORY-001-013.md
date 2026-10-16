# LORE-STORY-001-013: Inbox screen

## Description

As a knower, I want an inbox showing why I was asked and letting me Confirm, Correct or decline, with receipts naming who was notified, so that I can heal stale facts in seconds.

## Acceptance criteria

1. `inbox.heading` reads "{name}'s inbox"; `inbox.tabs` shows `inbox.tab-open` "Open ({n})" and `inbox.tab-done` "Done ({n})".
2. Each `inbox.question` shows "Lore needs your help · {relative time}", `inbox.question-fact` with the freshness badge, aspect and subject, `inbox.question-why-suspect` and `inbox.question-why-you` with the spec copy variants, and `inbox.question-escalation` ("You're knower {k} of 3." or "Asked {prev} first; they said "Not me". You're knower {k} of 3."). Real-repo questions show `inbox.question-real-knower` "Suggested knower: @{handle} (not contacted) — you're answering as the simulated Repo Steward."
3. Actions `inbox.confirm` "Confirm", `inbox.correct` "Correct", `inbox.not-me` "Not me", and (synthetic questions) `inbox.simulate-timeout` "Simulate no reply" call the inbox API.
4. `inbox.correct` expands `inbox.correct-form` with `inbox.correct-value` ("What's true now? ({predicate})", pre-filled with the current value), `inbox.correct-value-count` "{n}/300", `inbox.correct-note` ("Note (optional)"), `inbox.correct-note-count` "{n}/1000", `inbox.correct-submit` "Submit correction" and `inbox.correct-cancel` "Cancel".
5. Receipts: `inbox.receipt-confirmed` "Confirmed. Fact re-stamped as of {date time}."; `inbox.receipt-corrected` "Correction saved. Old fact replaced. Notified {n} people: {names}." with `inbox.receipt-view-graph` "View on graph"; `inbox.receipt-corrected-none` "Correction saved. Old fact replaced. No one depends on this fact."; `inbox.receipt-passed` "Passed to {person}." with `inbox.receipt-view-as`; `inbox.receipt-nobody` "No one else to ask. Marked as "Nobody knows"." Focus moves to the receipt.
6. Each `inbox.notice` shows `inbox.notice-title` "{subject} changed", the struck-through old statement with `Superseded` and the new one with `✓ Trusted`, `inbox.notice-source`, `inbox.notice-why` "You're seeing this because {entity} (which you {own | work on}) depends on {subject}.", `inbox.notice-mark-read` "Mark read" and `inbox.notice-open-ask` "Open in Ask".
7. A 409 shows `inbox.stale-item` "This question was already resolved / passed to {person}." with actions disabled.
8. Empty: `inbox.empty` "Nothing needs you, {first name}. Lore will ask you when a fact you know about goes stale." with `inbox.empty-hint` "People with open items: …" links. Loading: `inbox.loading` (3 skeleton rows). Submit failure: `inbox.error` "Couldn't save your answer." with `inbox.error-retry`; the typed correction is kept.
9. E2E: preset handover → Dana Lee's inbox → Correct → receipt names notified people → a dependent's inbox shows the notice; XSS payload in a correction note renders inert.

## Wireframe

Spec: `../wireframes/wireframe-spec.md`

- Screen: `inbox` — states: populated, correct expanded, receipts, already resolved, empty, loading, submit error
  - `inbox.confirm`
  - `inbox.correct`
  - `inbox.correct-cancel`
  - `inbox.correct-form`
  - `inbox.correct-note`
  - `inbox.correct-note-count`
  - `inbox.correct-submit`
  - `inbox.correct-value`
  - `inbox.correct-value-count`
  - `inbox.empty`
  - `inbox.empty-hint`
  - `inbox.error`
  - `inbox.error-retry`
  - `inbox.heading`
  - `inbox.loading`
  - `inbox.not-me`
  - `inbox.notice`
  - `inbox.notice-mark-read`
  - `inbox.notice-open-ask`
  - `inbox.notice-source`
  - `inbox.notice-title`
  - `inbox.notice-why`
  - `inbox.question`
  - `inbox.question-escalation`
  - `inbox.question-fact`
  - `inbox.question-real-knower`
  - `inbox.question-why-suspect`
  - `inbox.question-why-you`
  - `inbox.receipt-confirmed`
  - `inbox.receipt-corrected`
  - `inbox.receipt-corrected-none`
  - `inbox.receipt-nobody`
  - `inbox.receipt-passed`
  - `inbox.receipt-view-as`
  - `inbox.receipt-view-graph`
  - `inbox.simulate-timeout`
  - `inbox.stale-item`
  - `inbox.tab-done`
  - `inbox.tab-open`
  - `inbox.tabs`

## Requirements implemented

- gather.md: Self-healing loop steps 3–6
- gather.md: Business rules #6, #7, #8

## Depends on

- LORE-STORY-001-010
- LORE-STORY-001-012

## Agents likely needed

- [x] frontend
- [ ] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
