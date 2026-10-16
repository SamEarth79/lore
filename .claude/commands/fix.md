# /fix

## Position in the framework

A standalone, lightweight entry point for small fixes and bug fixes —
parallel to `/design` but far cheaper: no `gather.md` Q&A, no six-agent
`analyze.md` review. It collects a running list of fixes across multiple
conversation turns, then on `/fix implement` converts that list into a
**new Epic with one story per fix** and hands off into the **existing**
`/implement-epic` → PR pipeline unmodified, so state tracking, Asana, and
PR machinery stay uniform with the rest of the framework.

- **Previous step**: none — starts from a clean default branch.
- **This step**: `/fix` opens a collection session (branch + log file);
  ongoing conversation turns append to the log; `/fix implement` promotes
  the log into an Epic + one story per fix, creates the Asana Epic and
  subtasks, and runs the standard `/implement-epic` pipeline (which itself
  ends in the standard PR phase).
- **Next step**: once `/fix implement` hands off, everything downstream
  (per-story `plan.md` → `execute.md` → `test.md` → `commit.md`, then the
  PR phase) is identical to a normal `/implement-epic` run — see
  `commands/implement-epic.md`.

## Trigger

- `/fix` — start a new fix session (or resume the current one, if a
  `collecting` session already exists).
- `/fix implement` — finalize the current session's log into an Epic (one
  story per fix) and implement it.

## Part A — starting a session (`/fix`)

1. Read `knowledge/state.json`. If a `fixes` entry already has
   `status: "collecting"`, resume that session instead of starting a new
   one: tell the user which fix batch is already open, remind them of the
   fixes logged so far, and stop here (don't re-branch or re-pull).
2. Check `git status`. If the working tree is dirty, warn the user and ask
   them to commit or stash before proceeding — do not discard anything.
3. Checkout the repo's default branch and `git pull`.
4. Ensure `knowledge/config.json` exists (bootstrap exactly as
   `commands/design.md` step 1 does — ask for the product name and derive
   a prefix if this is the first command ever run in this repo). Ensure it
   has a `nextFixNumber` counter (default `1` if missing).
5. Ask the user for a short name for this fix batch if they haven't
   implied one already (e.g. from their first message), and slugify it.
   Build the fix ID: `<PREFIX>-FIX-<NNN>-<slug>` (3-digit `NNN` from
   `nextFixNumber`), e.g. `LFC-FIX-003-search-crash`. Increment
   `nextFixNumber` in `knowledge/config.json`.
6. Create branch `feature/<FIX-ID>` from the just-pulled default branch —
   same branch-naming convention as regular features (see `CLAUDE.md`'s
   Git conventions), since this fix batch becomes a regular `features`
   entry in `state.json` the moment it's implemented, and reusing the
   convention means `implement.md`, `commit.md`, and `pr-create.md` need
   no special-casing.
7. Create `knowledge/fixes/<FIX-ID>/fixes.md` from `templates/fix-log.md`.
8. Register the session in `knowledge/state.json`:
   ```json
   "fixes": {
     "<FIX-ID>": {
       "status": "collecting",
       "branch": "feature/<FIX-ID>",
       "startedAt": "<ISO timestamp>"
     }
   }
   ```
9. Tell the user the branch and log file are ready, and that they can just
   keep describing fixes in following messages — no need to re-invoke
   `/fix` per item. Running `/fix implement` when they're done finalizes
   the batch.

## Part B — collecting fixes (ongoing conversation)

While `knowledge/state.json` has a `fixes` entry with `status: "collecting"`
and the user's message is not itself a slash command: treat the message as
describing one fix. Immediately:

1. Append a new numbered entry to the `## Fixes` section of
   `knowledge/fixes/<FIX-ID>/fixes.md`, phrased as a short, concrete,
   observable problem statement (paraphrase for clarity, but don't lose
   specifics like repro steps, file/component names, or expected vs. actual
   behavior if the user gave them).
2. Briefly confirm what was logged (one line) and keep going — don't ask
   for confirmation on every single entry, don't implement anything yet.

If the user's message is ambiguous (unclear whether it's a new fix, a
correction to one already logged, or just chat), ask before writing.

## Part C — finalizing (`/fix implement`)

1. Require an active `collecting` session in `knowledge/state.json`. If
   none exists, stop and tell the user to run `/fix` first.
2. Read `knowledge/fixes/<FIX-ID>/fixes.md` in full. If the `## Fixes`
   section is empty, stop and ask what to fix — don't manufacture stories
   with no content.
3. **Write one story per fix.** Each numbered entry in `fixes.md` becomes
   its own story (this is what lets `/implement-epic` parallelize/order
   them, and lets one fix fail or get skipped without blocking the others
   — unlike a single bundled story):
   - Create `knowledge/requirements/<FIX-ID>/stories/`.
   - Write minimal `knowledge/requirements/<FIX-ID>/gather.md`,
     `analysis.md`, and `architecture.md` stubs (a sentence or two each,
     noting this is a fix batch that skipped the full `/design` Q&A and
     six-agent `analyze.md` review — `fixes.md` is the scope).
     `implement.md`'s context-loading step (used per-story by
     `implement-epic.md`) reads `analysis.md` and `architecture.md` in
     full, so both must exist even though neither needs `analyze.md`'s
     full structure here.
   - Initialize `nextStoryNumber["<FIX-ID>"] = 1` in `knowledge/config.json`
     if not already present — same mechanism `draft.md` uses for regular
     features.
   - For each fix, in the order logged: story code
     `<PREFIX>-STORY-FIX<NNN>-<story-NNN>` (the `NNN` from the fix ID; the
     `FIX` marker keeps this numbering space distinct from regular feature
     story codes, since fix IDs and feature IDs are separate counters and
     could otherwise collide). Increment `nextStoryNumber["<FIX-ID>"]`
     after each.
   - Write `knowledge/requirements/<FIX-ID>/stories/<STORY-CODE>.md` using
     `templates/story.md`: Description = that one fix, phrased as a user
     story; Acceptance criteria = the fix resolved and verifiably no
     longer reproducing; Requirements implemented → reference `fixes.md`'s
     entry number directly (there is no `gather.md` business-rule
     breakdown to cite); `Depends on` empty unless two fixes are clearly
     related (e.g. same file, one blocks testing the other — don't invent
     dependencies that aren't obvious); `## Wireframe` = "none" (fix
     batches bypass `/design`, so there is no wireframe contract —
     `execute.md`'s conformance check and `reviewer`'s conformance block
     are simply not triggered); Agents likely needed left for `plan.md`
     to decide.
   - Register each story in `knowledge/state.json` as `pending`, exactly
     as `draft.md` step 1.4 does.
4. **Create the Asana Epic and subtasks.** Same mechanism as `draft.md`
   step 2, in the Sprint backlog project (GID `1216704733478132`,
   workspace `1216681525107044`):
   - **Epic** (`asana_create_task`): name `<FIX-ID>: <short batch
     summary>`; notes = the full, verbatim body of `fixes.md`; projects
     `["1216704733478132"]`.
   - **Subtasks** (`asana_create_subtask`, one per story, parented to the
     Epic): name `<STORY-CODE>: <fix summary>`; notes = the full body of
     that story's `.md` file.
5. **Promote into `state.json`'s `features` map** (this is what makes
   `/status`, `/resume`, `/implement-epic`, and `/pr-create` treat this
   batch exactly like a normal feature from here on — no special-casing
   needed downstream):
   ```json
   "features": {
     "<FIX-ID>": {
       "design": "complete",
       "isFix": true,
       "branch": "feature/<FIX-ID>",
       "stories": {
         "<STORY-CODE-1>": { "status": "pending", "step": null, "commit": null },
         "<STORY-CODE-2>": { "status": "pending", "step": null, "commit": null }
       },
       "pr": { "number": null, "url": null, "state": "none", "reviewCycles": 0 },
       "asana": {
         "epicGid": "<epic-task-gid>",
         "stories": {
           "<STORY-CODE-1>": "<subtask-gid>",
           "<STORY-CODE-2>": "<subtask-gid>"
         }
       }
     }
   }
   ```
   Then delete the `fixes["<FIX-ID>"]` entry — `features` is now the
   single source of truth for this batch, and leaving both would let them
   drift out of sync.
6. Show the user the generated stories and the Asana Epic link, then
   immediately run `/implement-epic <FIX-ID>` (call
   `commands/implement-epic.md` exactly as it's defined — it resolves the
   folder from `state.json`, fetches story order from the Asana subtasks
   just created, and runs every story in sequence). No extra confirmation
   gate is needed here beyond what `implement-epic.md` already asks at
   each step (plan presentation per story, commit confirmation per story,
   PR body confirmation) — the user already opted in by typing
   `/fix implement`.
7. `implement-epic.md` step 5 runs the PR phase once, after the last fix's
   story commits: `pr-create.md` → `pr-review.md` → `fix-review.md` loop →
   merge confirmation. `pr-create.md` links the Asana Epic in the PR body
   automatically (it prefers `epicGid` when present — see
   `CLAUDE.md`/`pr-create.md`'s Asana-link step) — no extra step needed
   here to satisfy "link the Asana Epic in the PR."

## Output

- `knowledge/fixes/<FIX-ID>/fixes.md` — the running log (Part A/B).
- `knowledge/requirements/<FIX-ID>/{gather.md,analysis.md,architecture.md,stories/*.md}`
  (one story file per fix), an Asana Epic + one subtask per fix, and a
  `features["<FIX-ID>"]` entry in `state.json` (Part C).
- From there on, identical output to a normal `/implement-epic` run:
  one commit per fix on `feature/<FIX-ID>`, a PR with an automatic review
  linking the Asana Epic, and `state.json` kept current at every
  transition per `CLAUDE.md`'s State tracking section.
