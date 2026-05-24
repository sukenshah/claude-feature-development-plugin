# Implementation Plan: <Feature Name>

- **Spec:** [`specs/<slug>/spec.md`](./spec.md)
- **Author:** <name>
- **Date:** <YYYY-MM-DD>
- **Status:** Draft | Approved | In Progress | Complete
- **Spec Size:** Lean | Standard | Comprehensive

## 1. Goal Echo

Restate the goal in one or two sentences, in your own words. Confirms the plan understood the spec.

## 2. Architecture Sketch

What components/modules are involved. New vs modified vs reused. ASCII or short prose is fine — don't draw a UML monolith.

```
[caller] -> [new module] -> [existing service]
                |
                v
            [new table]
```

## 3. Data Model

Entities, fields, indexes, migrations. Source of truth for each piece of data. Mark new vs modified.

## 4. Contracts

Public API surface, internal interfaces, event/message shapes, error envelopes. Versioning if any.

## 5. Seams & Touchpoints

Where new behavior plugs into existing code. Files/modules at the boundary, why this is the right seam, what stays untouched and why.

## 6. Failure & Concurrency Model

Per relevant FR/NFR — timeouts, retries, idempotency, ordering, partial-failure behavior, degraded mode.

## 7. Observability

Logs (structured fields), metrics (names + labels), traces, alerts, dashboards. What oncall needs to debug this at 3am.

## 8. Test Strategy

- **Unit:** what's covered, what's not.
- **Integration:** what's covered, what's not.
- **E2E / Manual:** what's covered, what's not.
- **Gaps:** what the test suite can't catch and how we'll know if it breaks.

## 9. Design Decisions

For each non-obvious call, one line. Format: **Decision** — tenets in tension — why this won.

- **D-1:** <decision> — <tenets in tension, e.g., DRY vs YAGNI> — <reasoning, tied to context from code review>.
- **D-2:** ...

## 10. Alternatives Considered

(Standard+) Approaches rejected and why. Even one is useful.

- **Alt A:** <approach> — rejected because <reason>.

## 11. Implementation Steps

Ordered, mergeable, self-contained. Each is a single PR's worth.

### Step 1 — <goal>

- **Files:** `path/to/file.ts`, `path/to/other.py`
- **Detail level:** Detailed | Sketched | One-line
- **Changes:** <description appropriate to detail level>
- **Tests:** <added or modified>
- **Done when:** <concrete checkable criterion>

### Step 2 — <goal>

...

## 12. Risks (Design-Level)

Risks specific to this design choice (the spec already covered feature-level risks). Mitigations or escalation path.

- **R-1:** ...

## 13. Rollout & Backout

(Standard+) Flag strategy, phased rollout, migration ordering, revert plan.

## 14. Out of Plan

Things the spec asked for that this plan deliberately defers. State why and when they come back.

## 15. Open Questions / Checkpoints

Things deferred to build time on purpose, with a checkpoint condition.

- **Q-1:** <unknown> — resolve at <checkpoint, e.g., "after step 3 we'll know the actual perf shape">.
