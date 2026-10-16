# /design

## Position in the framework

This is the **entry point and orchestrator** for the design phase. It does
not do any work itself — it runs these steps in order, each defined in its
own command file:

1. `commands/gather.md`
2. `commands/analyze.md`
3. `commands/wireframe.md`
4. `commands/draft.md`
5. `commands/review.md`

There is no finalize step. Whatever is on disk in
`knowledge/requirements/<feature-folder>/` when the user stops giving
feedback during step 5 is what `/implement` will later read as final.

## Trigger

`/design` or `/design <feature description>`

## What this command does

### 1. Resolve the product prefix and config

- Check whether `knowledge/config.json` exists at the product repo root.
- If it does not exist: ask the user for the product name, derive a prefix
  (e.g. "LibraryApp" → `LIB`), and create `knowledge/config.json`:
  ```json
  {
    "prefix": "<PREFIX>",
    "nextFeatureNumber": 1,
    "nextStoryNumber": {}
  }
  ```
- If it exists, read it. This is the single source of truth for the prefix
  and counters used by every later step.
- Likewise ensure `knowledge/state.json` exists (see `CLAUDE.md`'s State
  tracking section for the shape); create it with an empty `features`
  object if missing.

### 2. Resolve the feature folder

- If the user did not name the feature, ask for a short feature name now.
- Slugify it (lowercase, hyphenated).
- Take `nextFeatureNumber` from config, format as 3 digits, build the folder
  name: `<PREFIX>-<NNN>-<slug>` (e.g. `LIB-001-user-auth`).
- Create `knowledge/requirements/<feature-folder>/stories/`.
- Increment `nextFeatureNumber` in `knowledge/config.json` and add
  `"<feature-folder>": 1` to `nextStoryNumber`.
- Add the feature to `knowledge/state.json`:
  `features["<feature-folder>"] = { "design": "in_progress", "branch": null, "stories": {}, "pr": { "number": null, "url": null, "state": "none", "reviewCycles": 0 } }`.
- This `<feature-folder>` value is what every subsequent step (and later,
  `/implement`) refers to. Pass it explicitly into each step below.

### 3. Run the steps in order

Run each step as described in its own file, passing along `<feature-folder>`
and the running context:

1. **`gather.md`** — multi-round Q&A conversation (business logic, UX flows,
   security, technical constraints). Produces
   `knowledge/requirements/<feature-folder>/gather.md`. Expect this to take
   10–20 minutes; don't rush it.

2. **`analyze.md`** — reads `gather.md`, scans the codebase, writes
   `knowledge/requirements/<feature-folder>/analysis.md` and
   `knowledge/requirements/<feature-folder>/architecture.md`. Then proposes
   the story breakdown and waits for explicit user confirmation before
   proceeding.

3. **`wireframe.md`** — skipped (in one line) when the feature has no
   frontend surface. Otherwise: generates one low-fidelity HTML wireframe
   per screen under `knowledge/requirements/<feature-folder>/wireframes/`,
   loops with the user until approved, writes `wireframe-spec.md` (the
   enforceable contract, with stable `data-wf` element IDs), then
   **reconciles** `gather.md`, `architecture.md`, and the story list
   against what the iteration changed — including rewriting core
   `architecture.md` sections (re-gated by `design-security`) when the
   wireframe invalidated the approach, not just appending. Re-confirms
   the story list with the user if it changed.

4. **`draft.md`** — receives the reconciled story list from step 3 (or the
   confirmed list from step 2 if step 3 was skipped), writes
   `knowledge/requirements/<feature-folder>/stories/<STORY-CODE>.md` for
   each story with a `## Wireframe` section referencing the spec, then
   creates an Asana Epic task (in the Sprint backlog project) with
   subtasks for each story.

5. **`review.md`** — presents the full design (architecture, wireframes,
   stories) to the user and loops on feedback, patching files in place.

### 4. End state

When `review.md` ends:

- Set the feature's `design` to `"complete"` in `knowledge/state.json`
  (its stories were already registered as `pending` by `draft.md`, and
  `review.md` kept them in sync with any story additions/removals).
- Tell the user where everything landed
  (`knowledge/requirements/<feature-folder>/`) and that they can run
  `/implement-epic <feature-id>` (e.g. `/implement-epic LIB-002`) to
  implement all stories in one go, `/implement <story-code>` for a single
  story, or later just `/resume` to continue where things left off. Do not
  ask them to "confirm" or "finalize" — the design phase is simply over
  when they stop talking.
- If the design was abandoned partway (user bails before `draft.md`
  produced anything), remove the feature's entry from `state.json` and
  delete the feature folder — including any `wireframes/` drafts — rather
  than leaving a permanently `in_progress` ghost. The burned feature
  number is fine, gaps in numbering are expected.
