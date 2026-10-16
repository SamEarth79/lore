# draft.md

## Position in the framework

Step 4 of 5 in the `/design` workflow, invoked by `commands/design.md`.

- **Previous step**: `commands/wireframe.md`. Its outputs are the
  approved `knowledge/requirements/<feature-folder>/wireframes/*.html`
  files plus `wireframes/wireframe-spec.md`, reconciled
  `analysis.md`/`architecture.md`/`gather.md`, and the **reconciled**,
  user-re-confirmed story list carried in conversation context. If
  `wireframe.md` was skipped (no frontend surface), the inputs are
  `analyze.md`'s outputs and its confirmed story list instead.
- **This step**: write one story `.md` file per confirmed story, then create
  an Asana Epic task with subtasks for each story.
- **Next step**: `commands/review.md`, which presents the design to the user
  and loops on feedback.

## What this command does

### 1. Write story files

For each story in the confirmed list:

1. Read the feature's own number from the feature folder name (the `NNN` in
   `<PREFIX>-NNN-<slug>`) and `nextStoryNumber["<feature-folder>"]` from
   `knowledge/config.json`. Format each as 3 digits. Build the story code:
   `<PREFIX>-STORY-<feature-NNN>-<story-NNN>`.
2. Write `knowledge/requirements/<feature-folder>/stories/<STORY-CODE>.md`
   using `templates/story.md`:
   - Description (what this story delivers)
   - Acceptance criteria (observable, testable). For any story with a
     frontend surface, write UI criteria **against the wireframe spec**:
     name the `data-wf` element and quote the spec's copy/state, e.g.
     "When credentials are rejected, `login.error` is visible with the
     text 'Invalid email or password' and `login.submit` is re-enabled".
     Every state the spec lists for a screen this story owns must be
     covered by at least one criterion — a state with no criterion is a
     state nobody will build or test.
   - `## Wireframe` section: the screen(s) from
     `wireframes/wireframe-spec.md` this story delivers or changes, and
     the list of `data-wf` elements it owns. If two stories touch the
     same screen, partition the elements between them explicitly — no
     element is owned by two stories, and none is orphaned. Write "none"
     for stories with no frontend surface.
   - Which requirement areas from `gather.md` it addresses
   - Which agents it will need (frontend / backend / infrastructure)
   - `Depends on` field: list any story codes in this feature that must be
     committed before this one can start. Leave empty if independent — don't
     invent dependencies.
3. Increment `nextStoryNumber["<feature-folder>"]` in `knowledge/config.json`
   after each story is written.
4. Register each story in `knowledge/state.json`:
   `stories["<STORY-CODE>"] = { "status": "pending", "step": null, "commit": null }`.

### 2. Create Asana Epic and subtasks

After all story files are written:

**2a. Create the Epic task**

Use the `asana_create_task` MCP tool to create a task in the "Sprint backlog"
project (GID: `1216704733478132`, workspace: `slyce.store`,
workspace GID: `1216681525107044`):

- **Name**: `<FEATURE-FOLDER>: <short feature title>` (e.g.
  `LIB-001-user-auth: User Authentication`)
- **Notes**: paste the feature summary from `gather.md` (the one-paragraph
  description), followed by a blank line and the architecture approach from
  `architecture.md`'s opening section (2–3 sentences max)
- **Projects**: `["1216704733478132"]`

**2b. Create subtasks for each story**

For each story, use the `asana_create_subtask` MCP tool with the Epic task's
GID as the parent:

- **Name**: `<STORY-CODE>: <story title>` (e.g.
  `LIB-STORY-001-001: Add login form`)
- **Notes**: the full, verbatim body of the story's `.md` file (everything
  written in step 1 — description, acceptance criteria, requirement-area
  references, agents needed, `Depends on`) — not a summary. The Asana
  subtask must be a faithful mirror of the story file, since
  `/implement-epic` treats Asana as the authoritative story list and the
  subtask body may be someone's first look at the story.

**2c. Record Asana GIDs**

After creating the Epic and all subtasks, record the GIDs in
`knowledge/state.json` under the feature:

```json
"asana": {
  "epicGid": "<epic-task-gid>",
  "stories": {
    "<STORY-CODE>": "<subtask-gid>"
  }
}
```

This allows future commands (e.g. `/implement`) to update the Asana tasks
when stories are committed.

## Output

- `knowledge/requirements/<feature-folder>/stories/<STORY-CODE>.md` (one or
  more)
- `knowledge/config.json` updated with incremented story counters
- `knowledge/state.json` updated with `pending` story entries and Asana GIDs
- Asana Epic task + subtasks created in the Sprint backlog project

These are the required inputs for `review.md` — do not let `review.md` run
without all story files existing and Asana tasks created.

After writing all stories, cross-check the spec's Element index against
the union of every story's `## Wireframe` section: every `data-wf` ID must
be owned by exactly one story. Fix any orphan or double-ownership before
creating Asana tasks.
