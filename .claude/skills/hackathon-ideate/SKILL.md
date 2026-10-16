---
name: hackathon-ideate
description: Generate five bold, out-of-the-box hackathon project ideas from a set of keywords and constraints (tech stack, sponsor tracks, rules). Use when the user wants hackathon ideas, asks to brainstorm a hackathon project, or gives keywords/constraints and asks "what should I build".
---

# Hackathon Ideate

Generate five hackathon ideas from user-supplied keywords and constraints.

## Inputs

Ask the user only for what's missing, then proceed:
- **Keywords**: theme, domain, sponsor tech, buzzwords to build around.
- **Constraints**: required tech/APIs to use, hackathon rules, team size, time limit, judging criteria — anything that bounds the idea space.

If the user gives partial input, don't stall on clarifying questions unless the request is genuinely too vague to start — reasonable assumptions are fine and can be stated briefly.

## What counts as a good idea here

The bar is **originality and "did not see that coming"**, not raw complexity and not safety. Explicitly reject:
- The obvious first-thought application of the keyword (e.g. keyword "healthcare" → generic symptom checker chatbot).
- Ideas that are complex/ambitious purely for the sake of looking impressive — complexity is not the goal, unexpectedness is.
- Ideas that are just an existing well-known product with the keyword swapped in.

Aim for ideas that combine the keywords/constraints in a way a judge hasn't seen five times that day — an unusual mechanism, an unexpected data source, a surprising angle on the constraint, or a twist on what the tech is "supposed" to be used for.

## Process

1. Brainstorm broadly and silently discard anything that's the first, most obvious idea for the keyword — that's the baseline to beat, not an output.
2. Push each surviving idea to be sharper or weirder: what's the one twist that makes a judge lean forward?
3. Select five ideas that are diverse from each other (different mechanisms/angles, not five variations on one concept).
4. For each, assign an **X-Factor rating (0-5)** — a measure of surprise/originality/"wow", not effort or feasibility. 5 = genuinely hasn't been done, judges will remember it. 0-2 = don't include; if an idea scores that low, replace it.

## Output format

For each of the 5 ideas:

**N. [Idea name]** — X-Factor: X/5
- **The pitch**: 1-2 sentences, what it does and why it's unexpected.
- **The twist**: the specific mechanism/angle that makes this not-obvious.
- **How it uses the constraints**: which required tech/rules it satisfies and how.

Keep each idea tight — no elaborate architecture docs or build plans unless asked. This is ideation, not a spec.
