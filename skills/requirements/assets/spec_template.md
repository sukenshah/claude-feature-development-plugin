# Spec: <Feature Name>

- **Author / Requestor:** <name>
- **Date:** <YYYY-MM-DD>
- **Status:** Draft | Reviewed | Approved
- **Size:** Lean | Standard | Comprehensive

## 1. Problem & Motivation

What hurts today. Who feels it. Why now. 2–4 sentences.

## 2. Goals

- Numbered, testable outcomes the feature should achieve.
- G-1: ...
- G-2: ...

## 3. Non-Goals

Explicit list of things this work is *not* doing, to bound scope.

- ...

## 4. Users / Actors

Who triggers this, who consumes it, internal vs external, expected scale of users.

## 5. Code Context

Files / modules read while preparing this spec, with a one-line note on what each does today and how it relates to the change.

- `path/to/file.ts` — current responsibility, relevance to feature.
- `path/to/other.py` — ...

## 6. Functional Requirements

Numbered, testable. A reviewer must be able to say yes/no on each.

- **FR-1:** ...
- **FR-2:** ...

### Happy path

Step-by-step description of the primary flow.

### Edge cases & failure modes

- Edge: ...
- Failure: <trigger> → <expected behavior>

## 7. Non-Functional Requirements

Only include categories that apply at this feature's size. Mark N/A explicitly for categories deliberately skipped.

- **NFR-1 (Performance):** ...
- **NFR-2 (Reliability):** ...
- **NFR-3 (Security):** ...
- **NFR-4 (Observability):** ...
- **NFR-5 (Compatibility / Migration):** ...
- **NFR-6 (Compliance):** ... (or N/A)
- **NFR-7 (Cost):** ... (or N/A)
- **NFR-8 (Accessibility / i18n):** ... (or N/A)
- **NFR-9 (Maintainability):** ...

## 8. Data & API Contracts

Schemas, payloads, endpoints, events. Include shape, validation, and versioning.

## 9. Assumptions

Every assumption underpinning the spec. Each should be confirmed by the requestor.

- A-1: ...

## 10. Risks

Likelihood × impact × mitigation. Be honest when no mitigation exists.

- **R-1:** <risk> — likelihood: low/med/high, impact: low/med/high, mitigation: ...

## 11. Anti-Patterns Considered & Avoided

If during the interview a tempting-but-bad path was identified and rejected, record it so future readers don't re-propose it.

- Tempting: ... — Why avoided: ...

## 12. Alternatives Considered

(Standard+) Other approaches and why they were rejected. Even one rejected option is useful.

- **Alt A:** ... — rejected because ...

## 13. Rollout & Backout

(Standard+) Flag strategy, dark-launch, migration ordering, how to revert if it goes wrong.

## 14. Success Metrics

How we'll know post-launch whether this worked. Quantitative where possible.

## 15. Open Questions

Anything unresolved. Each needs an owner and a needed-by point.

- **Q-1:** ... — owner: <name>, needed by: <phase>

## 16. Out of Scope (re-statement)

Optional restatement for clarity at handoff.
