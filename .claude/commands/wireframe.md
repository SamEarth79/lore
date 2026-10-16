# wireframe.md

## Position in the framework

Step 3 of 5 in the `/design` workflow, invoked by `commands/design.md`.

- **Previous step**: `commands/analyze.md`. Its outputs are
  `knowledge/requirements/<feature-folder>/analysis.md`,
  `knowledge/requirements/<feature-folder>/architecture.md`, and a
  user-confirmed story list carried in conversation context.
- **This step**: produce an HTML wireframe for every screen the feature
  introduces or changes, iterate on it with the user until approved, then
  **reconcile** — fold whatever the iteration changed back into
  `gather.md`, `architecture.md`, and the story list, so `draft.md` writes
  stories against the approved wireframe rather than the pre-wireframe
  design.
- **Next step**: `commands/draft.md`, which receives the (possibly
  revised) story list and the approved wireframe spec.

## When to skip

If `analysis.md` / `architecture.md` list **no** frontend surface
(`Components touched → Frontend: none`), say so in one line, write nothing,
and proceed straight to `draft.md`. Fix batches (`/fix implement`) never
run this step — they bypass `/design` entirely.

## What this command does

### 1. Enumerate screens and states

From `architecture.md`'s "Components touched → Frontend", `gather.md`'s
user flows, and the visual-states list the `design-ui-designer` agent
returned during `analyze.md`, build the list of screens to wireframe. For
each screen, the states it must show (empty / loading / populated / error /
content extremes, whichever apply).

Present the list to the user in one message and get a yes before drawing
anything — wireframing a screen the user considers out of scope is wasted
iteration.

### 2. Generate the wireframes

Write one self-contained HTML file per screen to
`knowledge/requirements/<feature-folder>/wireframes/<screen-slug>.html`:

- **Low fidelity on purpose.** Greyscale, system font, boxes and labels.
  No brand colors, no real imagery, no CSS framework. The wireframe
  settles *what is on the screen and where*, not the visual polish — the
  frontend agent inherits polish from the product's existing design
  system.
- **Every state on one page.** Render each state of the screen as its own
  labelled section (or a simple tab/toggle) so the user reviews them
  together.
- **Real copy, real data shapes.** Use the actual labels, button text,
  validation messages, and empty-state wording the feature will ship
  with — these travel into stories as acceptance criteria. Placeholder
  data should have the shape the API returns (per `architecture.md`'s API
  surface), not lorem ipsum.
- **Stable element IDs.** Every interactive or state-bearing element gets
  a `data-wf` attribute: `<screen-slug>.<element>` (e.g.
  `data-wf="login.submit"`, `data-wf="ticket-list.empty"`). These IDs are
  the contract that `/implement` is held to — see "How the wireframe is
  enforced" below. Choose them once and keep them stable across
  iterations; rename only if the element's meaning changes.
- **Reuse markers.** Where an element maps to an existing component the
  `design-ui-designer` identified, annotate it (`data-reuse="<Component>"`
  or a visible `[reuse: DataTable]` label) so the frontend agent doesn't
  build a duplicate.

Publish each file as an Artifact (or open it in the in-app browser) so the
user reviews a rendered page, not markup.

### 3. Iterate until approved

Loop:

1. Present the rendered wireframe(s) and a one-paragraph summary of what
   each screen does.
2. Take feedback. Patch the HTML in place — don't regenerate from scratch,
   or `data-wf` IDs drift.
3. **Track what the feedback implies beyond the wireframe.** Keep a
   running, in-conversation change ledger with one entry per accepted
   change, tagged by blast radius:
   - `wireframe-only` — layout/copy/ordering; no doc impact.
   - `scope` — adds/removes/alters a user-facing behaviour → `gather.md`
     and probably a story.
   - `architecture` — changes what data a screen needs, adds/removes an
     API call, changes components touched, changes data flow →
     `architecture.md` (possibly its core, not just an appendix).
   - `story` — changes the confirmed story breakdown (new story, merged
     stories, changed dependency).
   Tell the user at the time of each `architecture`- or `scope`-tagged
   change that it will be reconciled after approval — don't let them
   discover it at step 5.
4. Repeat until the user says the wireframe is approved. Approval is an
   explicit statement, not silence — ask once if unclear.

### 4. Write the wireframe spec

Once approved, write
`knowledge/requirements/<feature-folder>/wireframes/wireframe-spec.md`
from `templates/wireframe-spec.md`. It is a textual extraction of the HTML,
regenerated (not hand-edited) from the approved files:

- Per screen: purpose, layout regions, every `data-wf` element with its
  type, label/copy, which state(s) it appears in, the data it binds to,
  and any `data-reuse` mapping.
- Per screen: the full state list with the copy shown in each state.
- A global "Element index": every `data-wf` ID in the feature, one per
  line — this is what `execute.md`, `qa`, and `reviewer` check against.

The HTML is the review surface; the spec is the enforceable contract.
Agents diff text well and screenshots poorly, so nothing downstream should
have to interpret the HTML directly.

### 5. Reconcile design docs against the approved wireframe

Walk the change ledger from step 3. This is where the pre-wireframe design
catches up with what was actually approved. **Show the user the proposed
reconciliation as a single diff-style summary before writing anything**,
then apply on confirmation.

- **`gather.md`**: for every `scope` entry, patch the affected section in
  place (user flows, business rules, UX constraints), then append a
  dated `## Wireframe decisions` section listing each decision in one
  line with the `data-wf` IDs it concerns. Patch first, append second —
  the appendix is a changelog, not a substitute for correct sections.
- **`architecture.md`**: for every `architecture` entry, decide the depth
  of change honestly:
  - If it's additive (one more field on an existing response, one more
    component), patch the relevant section and note it under a dated
    `## Wireframe reconciliation` section.
  - If it changes the approach, the data flow, the data model, or the
    API surface in a way that invalidates what's written — **rewrite the
    affected core sections**. Do not bolt a contradicting appendix onto a
    document whose "Approach" now describes something else; `/implement`
    reads the whole file and cannot tell which paragraph wins.
  - Any rewrite of core sections goes back through the same gates
    `analyze.md` used: `design-developer` revises the draft and
    `design-security` re-signs it off (see `agents/design-security.md`
    "Sign-off"). Tell the user this is happening. The wireframe doesn't
    exempt the architecture from the security gate.
- **Story list**: for every `story` entry, update the in-conversation
  story list carried from `analyze.md` — add, remove, split, or re-wire
  dependencies — and re-present it to the user for explicit
  re-confirmation exactly as `analyze.md` step 6 does. `draft.md` must
  receive the *reconciled* list, never the pre-wireframe one.
- If the ledger is empty (pure `wireframe-only` changes or approved as
  drawn), say so in one line and skip the docs edits — but still write
  the spec.

State the outcome plainly: which files were patched, which sections were
rewritten, whether the security gate re-ran, and whether the story list
changed.

## How the wireframe is enforced downstream

This step produces the contract; these places hold implementation to it:

- `draft.md` — each story gets a `## Wireframe` section naming its
  screens and the `data-wf` elements it delivers; acceptance criteria
  reference those IDs and the spec's copy.
- `agents/frontend.md` — takes the HTML + spec as required inputs,
  reproduces layout hierarchy, states, and copy, and **ships the `data-wf`
  attributes in the real DOM**.
- `commands/execute.md` / `agents/orchestrator.md` — after the frontend
  agent reports, a conformance check verifies every `data-wf` ID the story
  owns exists in the rendered output, in each state the spec lists.
- `agents/qa.md` / `rules/testing.md` — Playwright selects by `data-wf`,
  so renaming an element breaks the tests, not just the intent.
- `commands/pr-review.md` / `agents/reviewer.md` — the reviewer gets the
  spec (never the chat) and returns an explicit wireframe-conformance
  verdict alongside the code review.

## Output

- `knowledge/requirements/<feature-folder>/wireframes/<screen-slug>.html`
  (one or more, approved)
- `knowledge/requirements/<feature-folder>/wireframes/wireframe-spec.md`
- In-place patches (and, where warranted, core rewrites) to `gather.md`
  and `architecture.md`, each with a dated reconciliation section
- A reconciled, re-confirmed story list in conversation context, passed
  to `draft.md`

`draft.md` must not run until the spec exists on disk and the story list
has been re-confirmed after reconciliation (or the step was legitimately
skipped for having no frontend surface).
