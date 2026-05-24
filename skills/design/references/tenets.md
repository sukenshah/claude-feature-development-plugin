# Design Tenets — Worked Trade-offs

The five tenets — **Simplicity, DRY, YAGNI, Modularity, Separation of Concerns** — pull in different directions. Below are worked examples of common conflicts and how to arbitrate. Use the reasoning, not the verdict — every situation is local.

## Tenet vs Tenet

### DRY vs YAGNI

**Symptom:** Two call sites have nearly-identical 15-line blocks. Tempted to extract `sharedHelper()`.

- **Extract** when: the logic changes together every time; both call sites are in the same domain; you've seen the duplication change in lockstep across at least 2 PRs.
- **Leave duplicated** when: the call sites are in different domains (e.g., billing and notifications) and the similarity is coincidental; one block is likely to evolve away from the other; you only have 2 instances and no signal they'll change together.
- **Rule of thumb:** *Wrong abstraction is more expensive than duplication.* Wait for the third occurrence before extracting. The Rule of Three exists for a reason.

### YAGNI vs Modularity

**Symptom:** Spec calls for one consumer of a new service. Tempted to add a plugin interface "in case we need more later."

- **Build single-consumer** when: adding a second consumer later is straightforward (it's a function/class extraction, not a rewrite); the consumer interface isn't crossing a hard boundary (network, process, repo).
- **Build with a seam** when: the boundary is genuinely a public contract (external API, cross-service event, plugin SDK); changing it later involves coordinating with other teams or external users; the cost of adding the seam now is hours, the cost of adding it later is weeks.
- **Rule of thumb:** YAGNI applies hardest to *reversible* choices. Irreversible choices (data shape, public API, sync/async boundary) earn extra design time even when you only need one consumer today.

### Simplicity vs Separation of Concerns

**Symptom:** A 60-line function does parsing, validation, business logic, and persistence. Tempted to split into four files.

- **Split** when: each piece becomes individually testable in a way it wasn't before; the function is in a hot/critical path that multiple people will edit; pieces have genuinely different change drivers.
- **Leave as one function** when: nothing else will call the inner pieces; splitting just adds files without unlocking testability; the function is short-lived (scaffolding, migration script).
- **Rule of thumb:** Separation pays for itself when it unlocks isolation (testing, reuse, independent change). Without an unlock, it's just file-shuffling that increases cognitive overhead.

### DRY vs Separation of Concerns

**Symptom:** Same validation rule exists in API layer and in DB layer. DRY says centralize. Separation says each layer owns its own validation.

- **Centralize** when: the rule represents a single domain invariant (e.g., "email must be unique") that shouldn't drift between layers.
- **Duplicate intentionally** when: the layers have different responsibilities for the rule (API rejects fast with a user-friendly message; DB enforces as a last-line guarantee with a constraint); they happen to look similar but are doing different jobs.
- **Rule of thumb:** Ask whether the duplication is *the same logic* or *similar-looking logic*. Same logic → DRY. Similar-looking logic → leave it, document the intent.

### Modularity vs Simplicity

**Symptom:** A small change touches three layers (controller, service, repo). Tempted to bypass the service layer "just this once."

- **Stay modular** when: the layers exist for genuine reasons (testability, auth boundary, transaction scope); skipping one creates a precedent others will follow.
- **Simplify** when: a layer is purely ceremonial (a service that just delegates to the repo with no business logic); the layer is dead weight, not a real seam.
- **Rule of thumb:** If a layer has never caught a bug, prevented a misuse, or enabled a test that wouldn't otherwise be possible, it's not earning its keep. Modularity must pay rent.

## Codebase-Convention Tie-Breakers

When tenets are roughly balanced, follow the existing code. A foreign idiom in a consistent codebase costs more than its theoretical purity is worth. Examples:

- If every existing controller validates inline, your new controller validates inline too — even if a validation middleware would be "cleaner."
- If every existing repository returns `Optional<T>`, you don't introduce nullable returns just because they're newer.
- If the codebase uses one ORM consistently, you don't introduce a second one for a single new table.

The exception: when you have *strong* evidence the existing convention is causing the pain the new feature is trying to solve. Then call it out explicitly in the plan's "Design decisions" — don't sneak the change in.

## Red flags in your own thinking

Watch for these patterns — they usually mean you're over-engineering:

- "We might need to swap implementations later." → No, you won't. Build for today.
- "This should be configurable." → Only if the spec says so. Otherwise it's a knob nobody will turn.
- "Let me just add a base class for these two." → Two is not enough. Wait for three.
- "I'll add a feature flag for safety." → Only if rollback is otherwise hard. Flags are debt.
- "This abstracts away the database." → If the abstraction is leaky (and they usually are), you've added complexity without buying portability.
- "Future-proof for scale." → Build for current scale × 2. Past that, you're guessing.

And the inverse — under-engineering signals:

- "It's just a script, we don't need tests." → Scripts that run in production deserve tests.
- "I'll inline this query, it's only used once." → It's used once *today*. If the query is non-trivial, name it.
- "We can refactor later." → Later rarely arrives. If the refactor is small, do it now.
