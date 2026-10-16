---
description: End-to-end hackathon idea pipeline — generate 5 out-of-the-box ideas, then have an isolated reviewer judge them against the full hackathon brief and pick the winning material.
argument-hint: [keywords / constraints / hackathon details, or paste the full hackathon brief]
---

Run the full hackathon idea pipeline for this request: $ARGUMENTS

## Step 1 — Gather inputs

You need two things before generating ideas:
- **Keywords/constraints**: theme, required tech, sponsor APIs, team/time limits.
- **Hackathon details**: the full brief — description, rules, and (critically) the judging/winning criteria.

Pull both from `$ARGUMENTS` first. If either is genuinely missing (not just terse), ask the user for it in one short question rather than guessing — the reviewer step is worthless without real judging criteria.

## Step 2 — Generate ideas

Invoke the `hackathon-ideate` skill with the keywords/constraints to produce 5 ideas, each with the pitch, the twist, how it uses constraints, and a claimed X-Factor (0-5).

## Step 3 — Review

Launch the `hackathon-idea-reviewer` subagent (via the Agent tool, `subagent_type: hackathon-idea-reviewer`) with a self-contained prompt containing:
- The full hackathon details/brief and judging criteria (verbatim, don't summarize them away).
- All 5 ideas from Step 2, exactly as generated (pitch, twist, constraint usage, claimed X-Factor).
- Instructions per the agent's own spec: validate each X-Factor, score fit against winning criteria, pick 1-2 winners, propose concrete improvements.

Run this in the foreground (`run_in_background: false`) — the next thing you do is report its result, so there's nothing useful to do while waiting.

## Step 4 — Report

Present to the user:
1. The 5 generated ideas (brief).
2. The reviewer's per-idea verdicts (X-Factor validation + criteria fit).
3. The 1-2 winning picks with the reviewer's sharpening suggestions, clearly called out as the recommended direction.

Keep the final report tight — this is a decision aid, not a spec document.
