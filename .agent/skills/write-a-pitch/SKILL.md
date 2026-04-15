---
name: write-a-pitch
description: "Write a Shape Up pitch for a feature or project. Use this skill when the user wants to create a pitch, shape work, write up a feature proposal, or draft a Shape Up pitch document. Also trigger when the user says things like 'write a pitch', 'shape this up', 'pitch this feature', 'create a pitch for', or wants to produce a problem/appetite/solution/rabbit-holes/no-gos document."
user_invocable: true
argument-hint: "[feature or problem description to pitch]"
---

# Write a Shape Up Pitch

Collaborate with the user to produce a well-shaped pitch — one that is **rough, solved, and bounded**. The pitch should contain enough structured information that an autonomous spec generation agent could produce a requirements/design/tasks triad from it without asking clarifying questions.

Shape Up pitches have five ingredients: **Problem, Appetite, Solution, Rabbit Holes, and No-Gos.** Your job is to help the user think through each one, grounding everything in the actual codebase.

## References

Before writing, read `references/example-pitches.md` in this skill's directory — it contains two real pitches at different scales that demonstrate the right tone and level of detail.

For deeper Shape Up theory (especially how pitches map to agent-driven spec generation), consult `references/shape-up-research.md`.

## How to collaborate

The user might arrive with anything from a vague idea ("we need better logging") to a half-written pitch to a Slack thread they want shaped. Meet them where they are:

- **Vague idea** — Start by asking questions to surface the problem. Who has this problem? What do they do today? What's unacceptable about it?
- **Clear problem, no solution** — Explore the codebase to understand the landscape, then propose a breadboard-level solution for the user to react to.
- **Solution already in mind** — Work backwards: validate the problem is real, check that the solution fits the appetite, identify rabbit holes from the code.
- **Existing draft** — Review it against the five ingredients. Strengthen what's weak, flag what's missing.

Throughout, **explore the codebase aggressively**. A pitch that isn't grounded in the actual code will produce a spec that misses existing patterns, duplicates work, or walks into traps the codebase already solved. Reference specific file paths — this helps downstream agents navigate the code.

Don't write the pitch all at once. Work through the ingredients with the user, getting their input and confirmation as you go. The final assembled pitch should feel like something the user co-authored, not something that was handed to them.

## The five ingredients

### Problem

A specific story showing why the status quo doesn't work. Name who is affected, what they currently do, and why that's unacceptable. The problem provides a baseline — every solution decision gets tested against "does this actually improve the situation?"

A good problem statement makes the reader feel the pain. A bad one just names a feature wish.

**Good:** "After running `hew init`, the user has no way to verify everything is wired up correctly. Tokens could be expired, the relay could be misconfigured, the webhook could be pointing at the wrong URL. Without a diagnostic tool, the user won't know until the foreman fails at runtime."

**Bad:** "We need a health check command."

### Appetite

A time budget stated as a design constraint, not an estimate. Appetite forces simplification — it transforms "What's the best solution?" into "What solves this within our constraint?"

| Tier | Task Count | Description |
|------|-----------|-------------|
| Small | 1-2 tasks | Contained change, single area of code |
| Medium | 3-6 tasks | Multiple files/areas, some coordination |
| Large | 7-14 tasks | Cross-cutting, multiple subsystems |

The appetite isn't just a label. Follow it with *why* this scope is right: "Small. A single command that runs a checklist of validation steps. No interactivity, no fixing — just clear pass/fail with actionable suggestions."

If the solution can't fit the appetite, that's a signal to narrow scope or find compromises — not to increase the budget.

### Solution

Describe the solution at **breadboard level**: places (screens/states), affordances (actions available), and connection lines (flow between them). Use words, not pictures. Show what the user can do without specifying what it looks like.

The solution should also include a "what it doesn't do" paragraph that reinforces boundaries from the inside. This is different from No-Gos (which are external exclusions) — it's the solution itself declaring its limits.

Ground the solution in the codebase. Name the models, services, and patterns that already exist and that this work will extend or interact with.

**Good:** "User clicks 'Run Analysis' on the taxonomy page → kicks off a background job → results show as a list of findings with severity and recommendations. The existing `StructuralAnalysisJob` gets extended rather than creating a new job."

**Too abstract:** "There should be some way to run analysis."

**Too concrete:** "A blue button labeled 'Analyze' at (200, 400) opens a modal with a progress bar and a cancel button..."

### Rabbit Holes

Technical risks *within* the project that could derail the build. For each one, state the risk and the compromise that defuses it.

Find these by exploring the codebase and walking through the solution slowly:
- Does anything require novel work or unvalidated assumptions?
- Are there external API dependencies with rate limits or error-handling concerns?
- Data migrations, performance at scale, concurrency risks?
- Hidden coupling between features?

Each rabbit hole should be *resolved* in the pitch, not just flagged. "We'll store files in S3, not build a custom blob store" is a resolved rabbit hole. "File storage might be complex" is not.

### No-Gos

Scope *outside* the project — things deliberately excluded to stay within appetite. These are firm, not aspirational. They tell the builder (or agent) exactly where to stop.

Good no-gos are things that a reasonable person might assume are in scope but aren't: "Don't fix problems automatically — report and suggest." "No Linear API calls from the relay."

## Output format

Assemble the final pitch in this structure:

```markdown
# {Pitch Title}

## Problem

[Specific story...]

## Appetite

[Tier + justification]

## Solution

[Breadboard-level description with codebase references...]

## Rabbit Holes

[Risks + resolutions...]

## No-Gos

[Explicit exclusions...]
```

Append a codebase context section:

```markdown
<details>
<summary>Codebase Context</summary>

- [file paths explored, grouped by directory]
- [existing patterns and models the solution builds on]

</details>
```

## Self-check before presenting

Before presenting the final pitch, verify it against the three properties of shaped work:

- **Rough** — Is there room for the builder to make decisions? If it reads like a spec, it's too concrete.
- **Solved** — Can you walk through the use case end-to-end? If you can't, it's not solved yet.
- **Bounded** — Are there explicit no-gos? Is the appetite stated? If not, scope will creep.

Every section should contain real, specific content. Don't produce a pitch with placeholders — if a section is weak, work with the user to strengthen it.
