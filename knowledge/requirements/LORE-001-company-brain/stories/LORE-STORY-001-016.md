# LORE-STORY-001-016: Guided demo tour

## Description

As a judge arriving cold, I want a short guided tour that walks me through the full loop, so that I see the differentiator without instructions.

## Acceptance criteria

1. `app-shell.tour-toggle` "Show me the loop" opens `app-shell.tour` "Show me the loop · step {n} of 6" with `app-shell.tour-steps` listing the six steps verbatim from the spec.
2. `app-shell.tour-next` "Next", `app-shell.tour-prev` "Back" and `app-shell.tour-skip` "Skip tour" move through/dismiss the tour; completed steps show ✓ and "Progress remembered in this browser."
3. Tour progress uses localStorage wrapped in try/catch; with storage unavailable the tour still works for the session.
4. Each step's action is achievable via the real UI (no mocked state): following steps 1–6 on a fresh sandbox reproduces the full loop end to end (E2E).
5. Keyboard-operable with visible focus; respects reduced motion.

## Wireframe

Spec: `../wireframes/wireframe-spec.md`

- Screen: `app-shell` — states: tour active
  - `app-shell.tour`
  - `app-shell.tour-next`
  - `app-shell.tour-prev`
  - `app-shell.tour-skip`
  - `app-shell.tour-steps`
  - `app-shell.tour-toggle`

## Requirements implemented

- gather.md: UX: discoverability (analysis risk #21)
- gather.md: User flows — Self-healing loop

## Depends on

- LORE-STORY-001-012
- LORE-STORY-001-013
- LORE-STORY-001-014
- LORE-STORY-001-015

## Agents likely needed

- [x] frontend
- [ ] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
