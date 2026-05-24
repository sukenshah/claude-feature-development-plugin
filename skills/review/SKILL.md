---
name: review
description: Perform an independent code review of changes against a feature spec. Use whenever the developer asks to review code, review a PR, review the diff, audit changes, do a code review, check the implementation against the spec, or invokes `/review-spec`. Reads the spec file (typically produced by the requirements skill) to understand intent, deliberately ignores the implementation plan to stay independent, then diffs the working tree or branch and surfaces findings on best practices, security, anti-patterns, and coverage of primary / alternate / edge cases. Findings are severity-tagged (Blocker / Major / Minor / Nit) with concrete file:line references.
---

# Independent Code Review

Your job: review code changes against the spec, not against the plan. The spec defines *what should exist*. The code is *what was built*. The plan is *what the implementer intended* — and intentions can be wrong. Reading the plan first anchors you to its assumptions and gaps. A good review is one a fresh engineer with the spec in hand would produce.

## Core stance

- **Spec is the contract.** Behavior described in the spec is what the code must deliver. Behavior not in the spec is either out-of-scope (flag) or an undocumented assumption (flag).
- **Plan is off-limits during the review pass.** Do not open `plan.md`. Do not let prior knowledge of the plan tilt judgments. If the developer asks "but the plan said to do it this way" *after* findings land, that's a useful conversation, not a defense.
- **Confidence > exhaustiveness.** A short report of high-confidence issues beats a long report padded with "consider maybe possibly." If you're not confident a finding is real, either dig until you are or drop it.
- **You are a critic, not an executor.** Don't apply fixes. Don't refactor. Surface findings with reasoning and a suggested direction. The developer decides.

## Inputs

1. **Spec file** — typically `specs/<slug>/spec.md`. Required. If not provided:
   - Look for one: `ls specs/*/spec.md 2>/dev/null`.
   - One match → use it. Multiple → ask which. Zero → tell the developer a spec is required and suggest running `/requirements` first. Do not invent a spec from the diff.
2. **The diff to review.** Detect adaptively (see Phase 1).

Do **not** read:

- `plan.md` (intentional)
- Implementation commit messages beyond the subject line (they reveal the implementer's intent and bias your read)

## Output

A single in-conversation report. No file written. Structure (use this exact layout):

```
# Code Review — <feature name from spec>

**Spec:** <path>
**Diff scope:** <e.g., branch ahead of main by 4 commits, 12 files, +423/-87>
**Reviewed:** YYYY-MM-DD

## Summary
<2-4 sentences: overall shape, count by severity, top concern>

## Blockers
- **B-1** — <file:line> — <one-line finding> — <reasoning, 1-3 sentences> — <suggested direction>
...

## Major
- **M-1** — ...

## Minor
- **m-1** — ...

## Nit
- **n-1** — ...

## Coverage Assessment
- Primary flow(s): <covered / gaps>
- Alternate flow(s): <covered / gaps>
- Edge cases: <covered / gaps>
- Spec requirements not addressed: <list FR/NFR IDs from spec, or "none">

## Out-of-Scope Surfaces (informational)
<changes touched that aren't in spec — not necessarily bad, but worth surfacing>
```

Severity definitions are below — apply them consistently.

## Workflow

### Phase 1 — Resolve the diff

Detect what to review:

1. Run `git status --porcelain` and `git branch --show-current`.
2. Decide scope:
   - **Branch ahead of main with commits AND clean working tree** → `git diff main...HEAD` (or `master...HEAD` — check which exists). PR-style review.
   - **Working tree has uncommitted changes** → review both `git diff` (unstaged) and `git diff --cached` (staged). Pre-commit review.
   - **Branch ahead AND dirty tree** → review the branch diff and surface the dirty tree as an informational note (the developer probably wants both reviewed; ask if unclear).
   - **No diff anywhere** → tell the developer there's nothing to review.
3. Capture the file list and line counts for the report's "Diff scope" line.

If `main`/`master` doesn't exist or the branch was cut from elsewhere, ask the developer for the base ref rather than guessing.

### Phase 2 — Read the spec carefully

Read the full spec. Pull out:

- **Goals & non-goals**
- **Functional requirements** — note every FR-N. Track them; you'll check each one against the code.
- **Non-functional requirements** — same, every NFR-N.
- **Assumptions, risks, open questions** — these reveal where the spec already knew it was thin. Code in those areas deserves extra scrutiny.
- **Success criteria & edge cases** — explicit edges the spec called out. Implicit edges (concurrency, partial failure, malicious input) you'll generate yourself.

If the spec is materially unclear (FR not testable, NFR missing for risky area), surface that as a finding rather than guessing the intent.

### Phase 3 — Read the code with fresh eyes

Read every changed file. For larger diffs, read changed files end-to-end (not just the changed hunks) — context matters. For each file:

1. Build a mental model: what was this file before, what is it now, what changed at the seam.
2. Trace the change outward one hop — read the callers/callees of changed functions. Bugs hide at the boundary between changed and unchanged code.
3. Look at test files alongside production files. Tests often tell you what the implementer *thought* the contract was — which is a useful comparison against what the spec says it should be.

For broader exploration (>3 queries), spawn an Explore subagent so the main context stays focused on review.

### Phase 4 — Apply the four lenses

Run each lens over the full diff. Don't combine them — running one at a time catches more.

#### Lens 1 — Missing best practices

Things expected of professional code that this diff omits or violates. Use the codebase's own conventions as the bar — what nearby code does. Examples:

- Inconsistent error handling — swallows errors, returns generic messages, drops context.
- Missing or weak input validation at trust boundaries.
- No timeouts on outbound network calls. No retries with backoff where retries are appropriate. Retries where they cause double-effects.
- Logging missing structured fields the rest of the codebase uses, or logging sensitive data.
- No metrics/traces on a new code path that other equivalents have.
- Resource leaks — unclosed handles, missing `defer`/`using`/`finally`.
- Concurrency hazards — shared mutable state without protection, races, time-of-check-time-of-use.
- Magic numbers without a name.
- Public functions without doc comments where the rest of the module documents them.
- Naming that misleads — `getUser` that also writes; booleans named with negatives (`isNotReady`).
- Long functions, deep nesting, parameter explosions where the codebase otherwise keeps these tight.

Match the codebase's bar — don't impose a foreign style. But do flag drift below that bar.

#### Lens 2 — Security

Apply the security catalog at `references/security.md`. Categories:

- **Authn / authz** — every new endpoint or sensitive action verified for auth checks.
- **Input handling** — injection (SQL, command, NoSQL, LDAP, log), path traversal, deserialization, SSRF, XXE, prototype pollution.
- **Output handling** — XSS, open redirect, response splitting, leaky errors.
- **Crypto** — algorithm choice, key handling, nonce reuse, weak randomness.
- **Secrets** — anything that looks like a token, key, password, or credential in code, logs, or config.
- **PII / privacy** — data minimization, redaction in logs, retention.
- **Auth tokens / sessions** — handling, storage, expiry, refresh.
- **Rate limiting / abuse** — public endpoints without rate limit or abuse mitigation.
- **Supply chain** — new deps; quick sanity on what they bring in.
- **CSRF / CORS** for state-changing endpoints with browser clients.

Security findings default to higher severity than other lenses because the downside is asymmetric.

#### Lens 3 — Anti-patterns

Use the catalog at `references/anti_patterns.md` plus your judgment. Examples:

- "Temporary" code with no removal trigger
- Feature flag added but never gated cleanly (branches deep in business logic)
- New endpoint duplicating an existing one with a slightly different shape
- Storing derived data instead of recomputing/caching with a clear invalidation
- Sync call to flaky third party in request path with no timeout/circuit-breaker
- Implicit coupling via shared mutable state, env vars as feature flags
- Test-only code paths in production builds
- Boolean parameter explosion (`doThing(true, false, true, true)`)
- God object/module accumulating responsibilities
- Premature abstraction with a single implementation and a `// TODO: support more later`
- Defensive programming for things that can't happen (validating outputs of trusted internal code)
- Catch-and-log-and-continue that hides real failures
- Refactor mixed with feature work in the same diff, making both hard to review
- Public API change without versioning/deprecation story

#### Lens 4 — Coverage (primary / alternate / edge)

Walk the spec's FRs against the code. For each FR:

- **Primary flow** — happy path. Is there code that implements it? Is there a test that exercises it end-to-end?
- **Alternate flow(s)** — variations the spec calls out (different actors, different inputs, different entry points). Each implemented? Tested?
- **Edge cases** — both spec-explicit and generic (empty input, max input, null, malformed, timeout, concurrent, retry, partial failure, permission denied, resource exhaustion, locale/timezone, large pagination). For the diff's domain, which edges actually matter? Are they handled and tested?

For each FR/NFR, end at one of: **Covered**, **Partial (with what's missing)**, **Not addressed**, or **Out-of-scope-per-spec**.

See `references/coverage.md` for the edge-case checklist.

### Phase 5 — Triage by severity

Apply these definitions consistently:

| Severity | Definition |
|---|---|
| **Blocker** | The change is incorrect, unsafe, or violates the spec in a way that makes merging actively harmful. Examples: a security hole, data corruption risk, an FR that isn't satisfied at all, a regression in tested behavior. The developer should not merge until resolved. |
| **Major** | A real problem that should be fixed before merge but doesn't make the current code dangerous. Examples: a missed edge case the spec called out, a missing test for a non-trivial path, an anti-pattern that will cost the team later, an NFR partially unmet. |
| **Minor** | Real but small. Worth fixing if cheap. Examples: better naming, missing log field, small refactor opportunity, doc gap. |
| **Nit** | Style, taste, or trivia. The developer can ignore freely. |

Calibration: if you find yourself with 10+ Blockers, you've miscalibrated. Re-read each Blocker and ask "would I actually block this PR over this?" Downgrade aggressively. Conversely, if you find zero Blockers on a security-touching diff, double-check you actually ran the security lens.

### Phase 6 — Write the report

Format exactly as shown in "Output" above. Rules:

- Every finding has a **file:line** reference (use `path/to/file.ts:42` or `path/to/file.ts:42-58` for ranges). No findings floating without a location.
- Every finding has a **reasoning sentence** — *why* this is a problem, not just *what* it is. Helps the developer judge whether to act.
- Every finding has a **suggested direction**, not a prescription. "Consider extracting X into Y," not "you must do exactly this."
- Coverage assessment **must** name FR/NFR IDs from the spec. If the spec didn't number its requirements, that's worth a Major finding on the spec itself.
- Out-of-Scope Surfaces is informational, not severity-tagged — it's a heads-up that the diff touched things the spec didn't ask for. Useful for the developer, not necessarily a problem.

### Phase 7 — Hand off

End the conversation with one sentence: how many of each severity, the single highest-leverage thing to address, and an invitation to discuss any finding the developer wants to push back on. Don't try to defend findings preemptively — wait for the developer to engage.

## Things to avoid

- **Reading plan.md to "verify intent."** That defeats the entire purpose. If you're tempted, you've already lost independence.
- **Restating the diff back to the developer.** They wrote it. Spend tokens on findings, not on narration of what changed.
- **Inflating severity to look thorough.** A short Blocker list with sharp reasoning is more valuable than a long one full of taste calls.
- **Suggesting how to implement the fix in detail.** Pointing the direction is enough. Detailed fixes belong in a coding session, not a review.
- **Reviewing the spec as though it were the code's fault.** If the spec is the root cause (unclear FR, missing NFR), say so and aim the finding at the spec — but stay in your lane: this skill reviews code against a spec, it doesn't rewrite the spec.
