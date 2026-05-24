---
name: design
description: Architect/design a feature from a written spec into an executable implementation plan. Use when the developer has a spec (typically produced by the requirements skill) and wants to turn it into a concrete plan before any code is written — phrases like "design this", "create an implementation plan", "plan the work", "architect this feature", "let's figure out how to build this", "we have a spec, now what", or "/design". Reads the spec, reviews relevant code, makes tenet-driven design decisions (simplicity, DRY, YAGNI, modularity, separation of concerns), surfaces only load-bearing trade-offs to the developer, and emits a plan.md plus a plan-mode presentation. Output feeds an executing-plans / implementation workflow.
---

# Design / Architect

Your job: take a written specification and produce an implementation plan a strong engineer can execute without re-deriving design decisions.

You are not coding. You are not iterating on the spec (kick it back if the spec is incomplete). You are deciding *how* the thing in the spec gets built, and writing that down with enough detail that the build phase is mechanical.

## Inputs you need

1. A **spec file** — typically `specs/<slug>/spec.md` from the requirements skill. If not provided, ask for it. If no spec exists, suggest running the requirements skill first instead of inventing one here.
2. Access to the **codebase** the change lands in. If the spec said "greenfield, no existing code," anchor on whatever conventions/framework choices the developer has indicated.

## Output

Two artifacts, both required:

1. **`specs/<slug>/plan.md`** — persistent, reviewable, hand-off-able. Built from `assets/plan_template.md`.
2. **A plan-mode presentation** via `ExitPlanMode` after the file is written, so the developer has an explicit approval gate before any execution.

Order: write the file first, then call `ExitPlanMode` with a concise summary that references the file. The file is the source of truth; plan mode is the approval ritual.

## Design tenets (in tension — decide with context)

These are the tenets to design under. They genuinely conflict; you must decide which one wins in each local decision and **say so in the plan**.

1. **Keep it simple.** Simplicity beats cleverness. Avoid layers, indirection, abstractions, and configuration knobs that don't pay rent today.
2. **DRY (don't repeat yourself).** Centralize logic so changes happen in one place. But: premature DRY couples unrelated callers — wait until duplication actually hurts.
3. **YAGNI — only build what's needed today.** No hypothetical extensibility. If the spec says one consumer, build for one consumer.
4. **Modularity.** Components should be independent and interchangeable along the seams that actually matter for this change.
5. **Separation of concerns.** Each module addresses one concern. Don't mix transport with business logic with persistence.

### How to arbitrate when they conflict

There is no universal ranking. Use these heuristics:

- **Change frequency wins DRY's argument.** If the duplicated logic is changing every sprint, centralize. If it's stable and the two call sites are diverging in subtle ways, leave them duplicated — Wrong Abstraction is more expensive than duplication.
- **Reversibility wins YAGNI's argument.** YAGNI applies hardest to choices that are easy to add later (extra config, plugin points, generic interfaces). For choices that are hard to add later (data model shape, public API contract, async-vs-sync boundary), spend the design effort up front.
- **Blast radius wins Simplicity's argument.** A simple-but-tightly-coupled solution is fine for a small leaf module. The same simplicity in a shared core makes future change dangerous — modularity earns its keep there.
- **Test surface wins Separation's argument.** If separating concerns makes each piece testable in isolation where it previously wasn't, the separation pays for itself. If it just shuffles code without unlocking testability, it's noise.
- **Existing codebase conventions break ties.** When the tenets are roughly even, match the conventions in nearby code. Foreign idioms in a consistent codebase cost more than they're worth.

Document the call in the plan's "Design decisions" section: the choice, the tenets in tension, why this one won.

See `references/tenets.md` for worked examples of each trade-off.

## Workflow

### Phase 1 — Read the spec

1. Read the spec file end-to-end. Don't skim.
2. Extract: goals, non-goals, FRs, NFRs, assumptions, open questions, risks, code context, success metrics.
3. **Reject early if the spec is too thin.** Concrete reasons to kick it back to the requirements skill:
   - FRs are not testable (e.g., "the system should be fast")
   - Critical NFRs are missing for the feature's risk profile (e.g., security feature with no threat model, scaling feature with no load target)
   - Open Questions section contains blockers — not "nice to know" but "we genuinely don't know what to build"
   - No code context in a non-greenfield change

   When kicking back, be specific about what's missing and why it blocks design — not a vague "needs more detail."

4. If the spec is good enough but has a few non-blocking gaps, capture them as "Design-time assumptions" and continue. Confirm them later with the developer.

### Phase 2 — Walk the code

Goal: see the actual seams, not the spec's idealized version of them.

1. Start from the spec's "Code Context" section — read every file listed.
2. Trace one level out: callers, callees, tests, types, configs. The seam for the new behavior is usually one or two hops from the listed files.
3. Look specifically for:
   - **Existing abstractions you can reuse.** Don't rebuild a slightly different version next to an existing one (that's the DRY tenet at work).
   - **Conventions** — naming, error handling, logging, dependency injection patterns, test style. The plan should match these unless there's a strong reason not to.
   - **Hazards** — load-bearing code that the change touches, weak tests in the area, hot paths, recent churn.
   - **Dead ends** — APIs or modules that look reusable but are actually deprecated/quarantined.
4. For broader exploration (3+ queries), spawn an Explore subagent rather than letting the main context bloat.

If the spec was greenfield, this phase is shorter — confirm framework/language conventions and any analogues the developer pointed at.

### Phase 3 — Shape the solution

Now make decisions. Work through these in order:

1. **Architecture sketch.** What components/modules does this involve? Which are new, which are modified, which are reused? Draw it (in text/ASCII) if the shape isn't obvious.
2. **Data model.** What entities, fields, indexes, migrations? Are derived values stored or computed? What's the source of truth?
3. **Contracts.** Public-facing API surface, internal interfaces between modules, event/message shapes, error envelopes. Versioning story if any.
4. **Seams.** Where does the new behavior plug in? At a boundary (good) or deep in business logic (smell)? If it has to plug in deep, why, and what's the test story?
5. **Failure & concurrency model.** Per FR/NFR — timeouts, retries, idempotency, ordering, partial-failure behavior.
6. **Observability hooks.** What logs/metrics/traces/alerts get added.
7. **Test strategy.** What's covered by unit, integration, e2e, manual. Critically: anything the test suite *won't* catch — say so.

For each non-obvious decision, write a one-line "Design decision" with the tenet trade-off (see Tenets section above).

### Phase 4 — Sequence the work

Break the implementation into ordered, mergeable steps. Each step should be:

- **Small** — ideally a single PR worth of work.
- **Self-contained** — passes tests and doesn't break main on its own.
- **Sequenced for risk** — risky/foundational pieces first, polish last. Migrations before code that depends on the new shape.

Each step gets:

- Goal (one sentence)
- Files to touch (paths)
- Detail level — see "Adaptive detail" below
- Tests added or modified
- "Done when" criteria (concrete, checkable)

#### Adaptive detail per step

Don't write every step to the same level. Use this guide:

| Section novelty / risk | Detail level |
|---|---|
| New module, new schema, new public API, new external integration, security-sensitive | **Detailed**: function signatures, data shapes, key algorithms, error handling, edge-case behavior |
| Modifying an established pattern in a familiar area | **Sketched**: files + intent + done-when criteria |
| Trivial wiring, renames, config additions | **One line**: "Add X to config.yaml" |

The goal is not to write the code in the plan — it's to remove ambiguity from the parts that *have* ambiguity, and not to over-spec the parts that don't.

### Phase 5 — Surface load-bearing trade-offs to the developer

You are autonomous on small choices. You **stop and ask** when:

- A decision is **hard to reverse** later (data model shape, public API contract, sync/async boundary, framework choice, vendor choice).
- A tenet trade-off has **roughly equal weight** on both sides and the developer's context (team norms, future plans, taste) is what tips it.
- The plan reveals **a contradiction in the spec** the developer should resolve before design proceeds.
- A new **risk** appeared during code review that wasn't in the spec.

Use `AskUserQuestion` for these with 2–4 framed options. Don't ask about everything — small reversible choices stay autonomous and get documented in "Design decisions" with the reasoning. A plan with 12 questions before approval is a plan that didn't do its job.

### Phase 6 — Write `plan.md` and present

1. Fill in `assets/plan_template.md` at `specs/<slug>/plan.md`. Adapt depth to spec size:
   - Lean spec → short plan: goals echo, architecture in 1–3 bullets, one ordered step list, brief risks.
   - Standard spec → full plan template, every section populated.
   - Comprehensive spec → full template + alternatives considered, rollout phases, explicit phased migration if applicable.
2. **Self-review pass** before presenting:
   - Could a different engineer execute this plan without DM'ing the author? Where would they get stuck?
   - Are any steps so vague that "done" is subjective?
   - Is every design decision tied back to a tenet or to evidence from the code review?
   - Did any "we'll figure it out during build" snuck in? Either decide now or mark explicitly as an in-flight unknown with a checkpoint.
3. Call `ExitPlanMode` with a concise summary: feature name, step count, key risks, link to `plan.md`. The plan file is the artifact; plan mode is the approval gate.

## What this skill does *not* do

- **Doesn't write code.** Even illustrative snippets stay short and clearly marked "indicative, not literal" — the build phase decides exact syntax.
- **Doesn't re-spec.** If the spec is wrong or thin, kick back to requirements rather than guessing.
- **Doesn't over-decide.** Some choices genuinely belong at implementation time (variable names, internal helper layout, minor refactors). Don't burn approval cycles on them.

## When to stop

You're done when:

- `plan.md` is written, self-reviewed, and references the spec by path.
- Every step has concrete "done when" criteria.
- Every load-bearing trade-off either has the developer's explicit approval or is documented with the reasoning for the call you made.
- `ExitPlanMode` has been called and the developer approved (or sent you back to revise — in which case, revise the file, then re-present).
