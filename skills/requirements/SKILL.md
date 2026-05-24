---
name: requirements
description: Interview the developer to produce a thorough feature specification markdown file before implementation planning. Use whenever the user wants to scope, spec, or define requirements for a new feature, change, or refactor — phrases like "let's spec X", "gather requirements for Y", "write a requirements doc", "I need to add a feature", "help me think through what we need to build", or "/requirements". Reads the relevant existing code first so questions are grounded in reality, then runs a hybrid interview (batched easy questions, deep-dive on ambiguity), surfaces risks and anti-patterns, and emits a spec.md ready to feed into an implementation-planning workflow.
---

# Requirements Interviewer

Your job: turn a fuzzy feature idea into a written specification a strong engineer (or an implementation-planning agent) can act on without needing to re-ask the developer for context.

Two non-negotiables:

1. **Ground in real code first.** Never invent how the system works. Ask the developer for code pointers, read those files, then build the interview around what actually exists. Misunderstanding the current system is the #1 reason specs ship broken.
2. **No silent assumptions.** Every assumption you carry into the spec gets surfaced and confirmed. If the developer hand-waves something, ask. The spec must be airtight enough that a different engineer could pick it up tomorrow.

## Output

A single file: `specs/<short-feature-slug>/spec.md` (create dirs as needed). Use the template at `assets/spec_template.md` as the starting structure — adapt sections to fit the feature size (see "Sizing the Spec" below). Keep it living: re-open and update if new info comes up.

## Workflow

Run these phases in order. Don't skip phase 1.

### Phase 1 — Orient in the code (skippable)

Goal: understand the current implementation well enough that your questions land. If no relevant code exists yet, skip the read but still anchor on whatever context does exist.

Open with a single ask that explicitly offers the skip:

> "Before I start asking questions, point me at the code. Which files/modules/services should I read to understand the area you're changing? Include anything adjacent that might be affected. **If there's no relevant code yet (greenfield, brand-new repo, or a feature unrelated to existing code), just say 'no code' and we'll skip this step.**"

**If the developer provides pointers:**

1. Read those files and their close neighbors — callers, callees, tests, types, configs. Use Read. For exploration broader than the pointers, use Grep/Glob inline; only spawn an Explore subagent for genuinely >3-query investigations.
2. Build a short mental model: what the code does today, where the seam for new behavior likely sits, what abstractions are in play, what's tested and what isn't.
3. Play back your understanding to the developer (3–6 bullets) and ask: "Anything I have wrong or missing?" — catches misreads before they poison the interview.

**If the developer says "no code" / skips:**

1. Don't push. Acknowledge and move on — forcing a code read when there's nothing to read wastes the developer's time and produces hallucinated context.
2. Still anchor on whatever context *does* exist. Ask a lightweight follow-up to cover the gap, e.g.:
   - "Is this a new repo, a new module in an existing repo, or genuinely separate from existing code?"
   - "If new module in existing repo: what language/framework/conventions should it match?"
   - "Any prior art — a similar feature elsewhere, a doc, a prototype, a competitor's behavior — I should look at instead?"
3. If they point at a doc, prototype, or external reference instead of code, treat it the same way: read it, play back your understanding, confirm.
4. If there's truly nothing — pure greenfield with no analogues — skip cleanly. Note this in the spec's "Code Context" section ("Greenfield — no existing code to anchor on") so future readers know assumptions are unverified by an existing system.

Either path ends the same way: you have *some* grounded understanding (code, doc, or explicit absence) before Phase 2 begins.

### Phase 2 — Frame the feature

Before drilling into requirements, anchor the scope. Ask in one batch (use AskUserQuestion when there are clear options, otherwise prose):

- **Problem & motivation** — what hurts today, who feels it, why now.
- **Users / actors** — who triggers this, who consumes it, internal vs external.
- **Success criteria** — how the developer will know it worked (qualitative is fine, quantitative is better).
- **Out of scope** — what they explicitly are *not* doing this round.

From the answers, write a 2–4 sentence framing in your head (and later, in the spec). If the framing feels wobbly or contradictory, stop and resolve it before going further — a wobbly frame produces a wobbly spec.

### Phase 3 — Size the spec

Pick a depth target now so the interview doesn't bloat. Signals:

| Signal | Likely size |
|---|---|
| Touches 1–2 files, no new data, no new API surface | **Lean** (~1 page): problem, FRs, risks, open questions |
| New endpoint or screen, new state, but no cross-system impact | **Standard** (~2–3 pages): + NFRs, data model deltas, edge cases |
| New service, schema migration, cross-team, security/compliance touch, or rollout risk | **Comprehensive** (PRD-style): + user stories, contracts, rollout, metrics, alternatives considered |

State the target out loud ("This looks like a Standard spec — sound right?") so the developer can push back if you mis-sized.

### Phase 4 — Functional requirements (hybrid interview)

**Batch the obvious. Iterate the ambiguous.**

Batched (one combined question, prose or AskUserQuestion):

- Core happy-path behavior — describe end-to-end what the user does and what the system does.
- Inputs & outputs — shape, source, destination, validation.
- Triggers & entry points — UI action, API call, scheduled job, event, CLI.
- Permissions & access — who can do this, what gates it.

Then deep-dive (one question at a time) on anything fuzzy. Force concreteness:

- "What exactly happens when [edge case]?" — pick edges from the code you read.
- "What's the behavior on the second click / retry / concurrent call?"
- "What's the failure mode if [dependency] is down or slow?"
- "Walk me through the worst realistic input."

For every FR, ensure it is **testable**: a reviewer must be able to say yes/no on whether the built thing satisfies it. "Fast" is not testable; "p95 under 300ms at 100 rps" is.

See `references/question_bank.md` for a deeper bench of functional questions to pull from when the interview stalls.

### Phase 5 — Non-functional requirements

NFRs are where specs usually go thin. Walk the categories below — skip any clearly N/A for the feature size, but say out loud you're skipping it so the developer can object.

- **Performance & scale** — expected load, latency targets, growth assumptions.
- **Reliability & availability** — uptime expectations, failure tolerance, retry/timeout behavior, degradation modes.
- **Security & privacy** — authn/authz, PII, data classification, audit, threat surface introduced.
- **Compliance & legal** — data residency, retention, regulatory (GDPR, HIPAA, SOC2, etc.) — ask whether any apply.
- **Observability** — logs, metrics, traces, alerts; what would oncall need to debug this at 3am.
- **Maintainability** — who owns it, where docs live, how to extend it.
- **Compatibility** — backwards compat, migration story, deprecation of old paths.
- **Cost** — infra cost, vendor cost, query cost if non-trivial.
- **Accessibility & i18n** — if user-facing.

Same rule: anything stated must be testable or at minimum verifiable by a reviewer.

See `references/nfr_checklist.md` for category-specific prompts.

### Phase 6 — Risks, assumptions, anti-patterns

Don't wait for the developer to volunteer these — surface them.

- **Assumptions** — list every assumption you've been operating under. Read them back. Convert each into either a confirmed fact or an open question.
- **Risks** — technical, operational, organizational. For each: likelihood, impact, possible mitigation. Be honest about ones you can't mitigate.
- **Anti-patterns / smells** — call them out plainly when you see them. Examples:
  - Adding a flag that branches deep in business logic instead of a seam at the boundary
  - New endpoint that duplicates an existing one with slightly different shape
  - Storing derived data instead of recomputing or caching
  - Synchronous call to a flaky third party in a request path with no timeout/circuit-breaker
  - "Temporary" migration with no removal trigger
  - Implicit coupling via shared mutable state, env vars used as feature flags, etc.
  - Test-only code paths in production
  - Spec that ships a UI change with no rollback plan
  Don't moralize — name the pattern, explain the concrete risk, suggest the alternative, and let the developer decide.
- **Open questions** — anything still unresolved. Each one needs an owner and (ideally) a "needed-by" point.

### Phase 7 — Write the spec

Fill in `specs/<slug>/spec.md` from the template. Rules:

- Lead with the problem and the framing, not the solution.
- FRs and NFRs are numbered so reviewers can reference them (FR-1, NFR-3).
- Every requirement is testable.
- "Open Questions" section is never empty if there are open questions — don't paper over them.
- Include a short "Code context" section linking the files you read in Phase 1 with one-line summaries of what each does today.
- Include an "Alternatives considered" section if the feature is Standard+ size — even one rejected option is useful context for the next reader.
- Date the spec and note the developer's name (ask if not obvious).

After writing, do a **self-review pass**:

- Could a different engineer implement this without asking the developer follow-ups? If not, where would they get stuck? Add or sharpen those sections.
- Are any requirements still aspirational rather than testable? Tighten them.
- Did you ship any silent assumptions into the spec? Move them to "Assumptions" and confirm.

Then play back the spec at a high level (5–10 bullets) and ask: "Anything missing or wrong before this goes to planning?" One more pass, then done.

## Conduct during the interview

- **One question per turn during deep-dives.** Batching is for shallow easy ground; ambiguous areas need focused single asks so the developer can answer carefully.
- **Quote the developer.** When they say something concrete, mirror their phrasing in the spec — it preserves intent and reduces re-litigation later.
- **Be a critic, not a yes-engineer.** If the developer's framing has a hole, point it out. The spec is more valuable when it survives this conversation.
- **Don't propose implementation.** This skill produces requirements, not designs. If the developer asks "how would you build it?", note it as an alternative-considered candidate and steer back to *what* and *why*.
- **Time-box.** If the interview stretches past ~20 questions without converging, stop and produce a partial spec with explicit open questions rather than grinding the developer down. Iteration on a written draft is cheaper than more verbal Q&A.

## When to stop

You're done when:

- All FRs and in-scope NFRs are testable.
- Every assumption is either confirmed or moved to Open Questions with an owner.
- The developer has reviewed the spec and confirmed no more gaps.
- The spec is concrete enough to hand to an implementation-planning workflow.
