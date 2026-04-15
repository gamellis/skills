# Research: Shape Up Framework for Agent-Driven Spec Generation

## Question

How can we scope features before a spec document so that an autonomous agent loop can generate a requirements document, implementation design, and task list from a feature request? Can Basecamp's Shape Up framework provide the right structure for humans to give agents the information they need to produce specs?

The goal: create a Linear task with structured shaping information → an agent extracts that information → generates a spec triad (requirements.md, design.md, tasks.md) → an agent builds it. We want to validate we're building the right thing *before* generating requirements.

## Thesis

Shape Up's "pitch" format — with its five ingredients (Problem, Appetite, Solution, Rabbit Holes, No-Gos) — can serve as the input contract between a human shaper and an agent-driven spec generation pipeline. The framework's emphasis on being "rough, solved, and bounded" maps directly to what an autonomous agent needs: clear direction without over-constraint, explicit scope limits, and flagged risks.

## Findings

### Shape Up Core Concepts

Source: [basecamp.com/shapeup](https://basecamp.com/shapeup)

Shape Up is a product development methodology built around three phases:

1. **Shaping** — senior people define the work at the right level of abstraction
2. **Betting** — leadership decides what to commit to (the "betting table")
3. **Building** — a team executes within a fixed time cycle

The key innovation is **fixed time, variable scope**: instead of estimating how long something takes, you decide how much time it's *worth* and adjust scope to fit. This flips the traditional relationship between scope and schedule.

### The Five Ingredients of a Pitch

Every shaped pitch contains:

#### 1. Problem
A specific, concrete story showing why the status quo doesn't work. Not "users want X" but "when a user tries to do Y, they can't because Z, which means they have to do W instead." The problem grounds all evaluation of whether proposed solutions actually matter.

#### 2. Appetite
The time budget, stated as a constraint, not an estimate. The appetite becomes a creative constraint that forces simplification — "it takes work and design insight to get to a simple idea that fits in a small time box."

#### 3. Solution
Described at breadboard level using two techniques:

- **Breadboarding**: captures interface topology using Places (screens), Affordances (buttons/fields), and Connection Lines (navigation flow). Words, not pictures.
- **Fat Marker Sketches**: deliberately crude visuals using thick markers to prevent over-specification of layout/design.

The solution must be "solved" at the macro level — all main elements present and connected — but "rough" enough to leave room for implementation decisions.

#### 4. Rabbit Holes
Specific technical risks identified during shaping. These are found through:
- Slow-motion walkthroughs of the use case
- Questioning each component: does this require novel work? unvalidated assumptions? unproven design?
- Probability thinking: well-shaped work has a thin-tailed distribution (predictable timeline); unresolved risks create fat tails

De-risking involves three moves:
- Patch holes with compromises (accept "messy" solutions to eliminate complexity)
- Declare boundaries (mark features as out of bounds)
- Present to technical experts with time-bound questions ("is X possible in 6 weeks?" not "is this possible?")

#### 5. No-Gos
Explicit exclusions that prevent scope creep and clarify boundaries. These are firm — not "nice to haves we'll skip" but "we are deliberately not doing this."

### Three Properties of Shaped Work

| Property | Meaning | Why It Matters |
|---|---|---|
| **Rough** | Deliberately unfinished, signals open space for contributions | Prevents over-specification that constrains implementers |
| **Solved** | Main elements present and connected at macro level | Provides clear direction despite roughness |
| **Bounded** | Explicit scope limits tied to appetite | Tells the team (or agent) where to stop |

### How Shape Up Maps to Agent-Driven Development

| Shape Up Concept | Agent Equivalent | Mapping |
|---|---|---|
| **Problem** | Requirements source | Drives user stories and acceptance criteria in requirements.md |
| **Appetite** | Scope constraint | Bounds the number of tasks in tasks.md; prevents gold-plating |
| **Solution** | Design seed | Provides the architectural skeleton for design.md |
| **Rabbit Holes** | Risk register | Agent avoids spending turns on flagged dead ends |
| **No-Gos** | Scope boundaries | Agent knows where to stop — critical for autonomous loops |
| **Rough** | Implementation freedom | Agent makes tactical decisions within design.md |
| **Solved** | Clear direction | Agent can generate a spec without ambiguity on the approach |
| **Bounded** | Turn/task limits | Prevents runaway spec generation |

The key insight: Shape Up's properties solve specific agent failure modes. "Bounded" prevents gold-plating and spiraling. "Solved" eliminates ambiguity that causes agents to guess wrong. "Rough" gives the agent room to make tactical implementation decisions. "No-Gos" are critical because autonomous agents have no natural sense of "that's enough" — they need explicit stop signals.

### Appetite Calibration

| Appetite | Task Count | Description |
|---|---|---|
| Small | 1–2 tasks | Contained change, single area of code |
| Medium | 3–6 tasks | Multiple files/areas, some coordination |
| Large | 7–14 tasks | Cross-cutting, multiple subsystems |

The agent must fit the generated spec within the appetite constraint. If the solution can't fit, that's a signal the pitch needs more shaping (narrower scope or identified compromises).

### Validation Gate

The pitch IS the validation artifact. Before spec generation, ask:

1. **Is the problem real?** Does the story resonate?
2. **Is the appetite right?** Is this worth N tasks?
3. **Is the solution shaped enough?** Can an agent generate a spec from this?
4. **Are the rabbit holes identified?** Will the agent get stuck?
5. **Are the boundaries clear?** Will the agent know where to stop?

If any answer is "no," the pitch goes back for more shaping — it doesn't enter the spec generation pipeline.

## Open Questions

1. **Who shapes?** In Shape Up, shaping is done by senior people who understand both product and technical landscape. Could an agent assist with shaping itself — e.g., given a raw feature request, help identify rabbit holes by reading the codebase?

2. **Agent-assisted shaping**: Could a lightweight agent read a raw feature idea + codebase and produce a *draft* pitch that a human then refines? This would lower the barrier to shaping while keeping human judgment in the loop.
