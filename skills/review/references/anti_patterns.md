# Anti-Pattern Catalog for Code Review

Common smells worth flagging. For each: name the pattern, explain the *concrete* risk (not just "it's bad"), suggest a direction. Don't moralize.

## Control-flow & branching

- **Boolean parameter explosion** — `doThing(true, false, true, true)`. Call site is unreadable; adding a fifth case breaks all callers. Direction: options object, or split into named functions.
- **Deep nesting / arrow code** — 4+ levels of nested conditionals. Hides edge cases and makes diff review harder. Direction: early return, extract guard clauses.
- **Catch-and-log-and-continue** — exception caught, logged, then code proceeds as if it succeeded. Hides real failures, produces silent data corruption. Direction: either handle (with explicit recovery) or rethrow.
- **Catch-and-swallow** — `catch (e) {}`. Same as above, worse — not even logged.
- **Return-null-on-error** — function returns `null`/`undefined` to signal failure with no way to distinguish from "legitimately empty." Direction: result type, optional, or throw.

## Abstraction & duplication

- **Premature abstraction** — base class, generic interface, or plugin point with one implementation and a `// TODO: support more later`. Direction: collapse until a second case actually appears.
- **Wrong abstraction** — shared helper called by two callers that have to pass weird flags to make it work for both. Costs more than the duplication it avoided. Direction: inline back and re-extract when the right shape emerges.
- **Copy-paste with subtle drift** — near-identical code in two places with one detail different. Will diverge further. Direction: extract if the change driver is the same; document if it's not.
- **God object / module** — one file or class accumulating unrelated responsibilities. Direction: split by concern.

## State & data

- **Storing derived data without invalidation** — denormalized field cached in a row with no clear "when to recompute" rule. Will go stale. Direction: compute on read, or define invalidation triggers.
- **Implicit coupling via shared mutable state** — module-level singleton mutated by multiple callers, env vars used to communicate between modules at runtime. Direction: explicit parameters, dependency injection.
- **Env vars as feature flags** — `if (process.env.NEW_FEATURE) { ... }`. Untestable, undocumented, accidental rollouts. Direction: real feature flag system, or just remove the branch.
- **Magic numbers / strings** — `if (status === 3)`. Direction: named constant, enum.
- **Stringly-typed APIs** — passing data as untyped strings (`"red,large,sale"`) when a structure exists. Direction: types.

## Time & lifecycle

- **"Temporary" code with no removal trigger** — `// TODO: remove after migration`. The migration finishes, the code stays. Direction: removal criterion in the comment, ticket linked, owner assigned.
- **Time-of-check-time-of-use** — read state, decide, then act on stale state. Direction: atomic operation, or re-check inside the critical section.
- **Synchronous blocking call in async/request path** — file IO, slow lookup, third-party call without timeout. Blocks the event loop or the request thread. Direction: async, timeout, circuit-breaker.
- **No timeout / no retry / wrong retry** — outbound call with no timeout means a stuck dep stalls your whole system. Retry without idempotency means double-effects. Direction: bounded timeout + idempotent retry where appropriate.

## Test smells

- **Test-only branches in production code** — `if (NODE_ENV === 'test') { mockBehavior }`. Tests no longer test the real thing. Direction: dependency injection at the boundary.
- **Tests that assert implementation details** — checking that internal helper was called rather than observable behavior. Brittle, refactor-hostile. Direction: assert on outputs/side effects, not call shapes.
- **Tests with no assertions** — runs the code, asserts nothing, passes regardless. Direction: assert specific outputs.
- **Snapshot tests for everything** — snapshot is updated without anyone reading the diff. Direction: assertions on what actually matters; snapshots only for stable artifacts.
- **Single huge test** — one test exercises 12 things, fails opaquely. Direction: split.

## API & change-management

- **Breaking change without versioning** — public API contract changed in-place. Direction: version the endpoint, support both during deprecation window.
- **New endpoint duplicating an existing one** — slightly different shape, slightly different behavior, both maintained forever. Direction: consolidate or explicitly separate use cases with names.
- **Mass rename mixed with logic change** — diff is mostly noise; the real change is buried. Direction: separate PR for rename, then logic.
- **Refactor + feature in same PR** — neither reviewable. Direction: split.

## Defensive / over-cautious

- **Defensive programming for impossibilities** — validating outputs of trusted internal code, null-checking values that the type system guarantees non-null. Adds noise without value. Direction: trust internal boundaries; validate only at trust boundaries.
- **Try/catch around everything** — every function wrapped. Errors get re-wrapped and re-thrown with less context. Direction: catch where you can act on the error.
- **Logging at every line** — production logs are unreadable; cost adds up. Direction: log at boundaries and meaningful state changes.

## Performance

- **N+1 queries** — loop that does a DB call per iteration. Direction: batch / join.
- **In-memory work that should be in DB** — pulling 100k rows to filter in code. Direction: push the filter to the query.
- **Hot-path allocation** — building large objects in tight loops. Direction: reuse, pool, or restructure.
- **Premature optimization** — caching layer or micro-opt without measurement. Direction: measure first.

## Configuration & environment

- **Config that should be code, code that should be config** — knobs in config files nobody changes, or values hardcoded that genuinely differ per env. Direction: invert.
- **Different code paths per env** — `if (env === 'prod') { ... } else { ... }`. The non-prod path is the one you debug with, but prod is the one users hit. Direction: same path, different config.

## Naming

- **Misleading names** — `getUser` that also writes, `validate` that mutates, booleans named with negatives (`isNotReady`). Reading the call site lies to you. Direction: name the action.
- **Overloaded names** — same word means different things in different layers (`User` is the auth user in one module, the billing user in another). Direction: namespace or disambiguate.

## Documentation & comments

- **Outdated comments** — comment describes the old behavior; reader trusts the comment, misses the bug. Direction: delete or rewrite.
- **What-comments** — `// increment i by 1`. Direction: delete.
- **TODO without context** — `// TODO: fix this`. No owner, no condition, no link. Direction: ticket or delete.
