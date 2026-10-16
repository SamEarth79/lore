# {{STORY_CODE}}: {{STORY_TITLE}}

> Filled in by `draft.md` during `/design`. One file per story, inside the
> feature's `stories/` directory.

## Description

As a {{user role}}, I want to {{action}}, so that {{benefit}}.

## Acceptance criteria

{{Numbered, testable criteria. Each should map to at least one feature-test
assertion during `/implement`'s `test.md` step.}}

1.
2.

## Wireframe

{{Screens from `../wireframes/wireframe-spec.md` this story delivers or
changes, and the `data-wf` elements it owns. Each element belongs to
exactly one story in the feature. The `frontend` agent reproduces these
screens and ships the `data-wf` attributes; `execute.md`'s conformance
check, `qa`'s E2E selectors, and `reviewer`'s verdict all verify against
this list. Write "none" if the story has no frontend surface.}}

- Screen: `{{screen-slug}}` — states: {{populated, empty, …}}
  - `{{screen}}.{{element}}`
  - `{{screen}}.{{element}}`

## Requirements implemented

{{Reference the section(s) of gather.md this story satisfies, e.g.
"Business rules #2", "User flows — error state".}}

-

## Depends on

{{Story codes in this feature that must be committed before this story can
start (e.g. the backend story whose API this story consumes). Leave the
list empty if independent. `/implement`, `/implement-epic`, and `/resume`
enforce this ordering.}}

-

## Agents likely needed

{{Initial guess at which specialist agents this story will need — confirmed
or adjusted by orchestrator's plan.md during /implement.}}

- [ ] frontend
- [ ] backend
- [ ] infrastructure

## Status

{{Ticked by `commit.md` when the story's commit lands — never by hand.
`knowledge/state.json` holds the authoritative machine-readable status;
this checklist mirrors it for human readers.}}

- [ ] Implemented
- [ ] Tested
- [ ] Committed
