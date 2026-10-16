# LORE-STORY-001-015: Ingest screen

## Description

As a visitor, I want to ingest a preset, my own text, or the next timeline event and see exactly what changed, so that I can trigger the self-healing loop myself.

## Acceptance criteria

1. `ingest.heading` "Ingest"; `ingest.presets` lists the three `ingest.preset` buttons with the spec copy; clicking one ingests it.
2. Free text: `ingest.kind` with `ingest.kind-ticket`, `ingest.kind-chat`, `ingest.kind-doc`; `ingest.title` "Title (optional)"; `ingest.text` with placeholder "Paste a ticket or chat message as {persona}…"; `ingest.char-count` "{n}/4000"; `ingest.submit` "Ingest" disabled when empty or over limit; `ingest.llm-notice` with the exact spec copy.
3. `ingest.timeline` shows `ingest.timeline-clock` "Company clock: {date}", `ingest.timeline-remaining` "{n} events left", `ingest.timeline-preview` "Next: …" and `ingest.timeline-next` "Replay next event"; when exhausted, `ingest.timeline-done` "All seeded events replayed." with `ingest.timeline-reset` "Reset sandbox".
4. `ingest.result-empty` shows the placeholder copy before any ingest; `ingest.progress` (aria-live) shows the three staged steps while processing.
5. `ingest.result` shows "Ingested as {persona} · {kind} · company clock now {date}", `ingest.result-text` with evidence highlighted from offsets, `ingest.result-fact` cards with badge and `ingest.result-confidence` "confidence {n}%" and the note "Facts from pasted text start Unverified until someone confirms them.", `ingest.result-event` items, `ingest.result-flagged` "Made suspect: {n} facts" with `ingest.result-open-freshness`, and `ingest.result-asked` with `ingest.result-view-as`.
6. `ingest.result-none` shows the spec copy when nothing is extracted; `ingest.error` "Extraction failed. Your text wasn't added." with `ingest.error-retry`; `ingest.error-llm-down` "Extraction is temporarily unavailable. Presets and the seeded timeline still work."
7. Validation: `ingest.text-error` "Text is limited to 4,000 characters ({n}/4,000)."; `ingest.title-error` "Title is limited to 200 characters."; a 413 shows `ingest.cap-error` "Your sandbox has reached its limit of 200 ingested items. Reset to start over."
8. E2E: preset handover → `ingest.result-flagged` lists the ownership fact; an XSS payload pasted into `ingest.text` renders inert in `ingest.result-text`.

## Wireframe

Spec: `../wireframes/wireframe-spec.md`

- Screen: `ingest` — states: empty form, processing, result, nothing extracted, extraction failed, LLM down, replay finished, validation errors, cap
  - `ingest.cap-error`
  - `ingest.char-count`
  - `ingest.error`
  - `ingest.error-llm-down`
  - `ingest.error-retry`
  - `ingest.heading`
  - `ingest.kind`
  - `ingest.kind-chat`
  - `ingest.kind-doc`
  - `ingest.kind-ticket`
  - `ingest.llm-notice`
  - `ingest.preset`
  - `ingest.presets`
  - `ingest.progress`
  - `ingest.result`
  - `ingest.result-asked`
  - `ingest.result-confidence`
  - `ingest.result-empty`
  - `ingest.result-event`
  - `ingest.result-fact`
  - `ingest.result-flagged`
  - `ingest.result-none`
  - `ingest.result-open-freshness`
  - `ingest.result-text`
  - `ingest.result-view-as`
  - `ingest.submit`
  - `ingest.text`
  - `ingest.text-error`
  - `ingest.timeline`
  - `ingest.timeline-clock`
  - `ingest.timeline-done`
  - `ingest.timeline-next`
  - `ingest.timeline-preview`
  - `ingest.timeline-remaining`
  - `ingest.timeline-reset`
  - `ingest.title`
  - `ingest.title-error`

## Requirements implemented

- gather.md: Alternate flows — Ingest live
- gather.md: Self-healing loop steps 1–3

## Depends on

- LORE-STORY-001-009
- LORE-STORY-001-012

## Agents likely needed

- [x] frontend
- [ ] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
