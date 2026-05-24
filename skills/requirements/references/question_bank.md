# Functional Requirements Question Bank

Pull from this list when the interview stalls or when you want to stress-test a feature. Don't ask all of them — pick the ones the current feature actually needs. Convert vague answers into concrete, testable requirements.

## Behavior & flows

- Walk me through the primary user flow end to end, step by step.
- What's the trigger? UI action, API call, scheduled job, event, webhook, CLI?
- What state changes as a result of this action? (DB rows, cache, external service, file)
- What does the user see immediately after the action? What do they see later?
- Is this action idempotent? What happens on retry?
- Is order of operations significant? What guarantees ordering?
- Are there secondary side effects (emails, notifications, downstream events)?

## Inputs & validation

- What are the inputs (fields, files, payload shapes)?
- Where do inputs come from (user, upstream service, config)?
- What validation rules apply? Sync vs async validation?
- What's the behavior on invalid input — block, sanitize, partial-accept?
- What's the largest realistic input? The malicious input?
- What's the default when an optional input is missing?

## Outputs

- What does the response look like (shape, fields, error envelope)?
- What's the success representation? Failure representation?
- Are there partial-success states?
- Who consumes this output downstream and what do they expect?

## State & data

- What new entities does this introduce? What existing entities does it modify?
- What's the source of truth for each piece of data touched?
- Are there derived/computed values? How fresh do they need to be?
- Any migrations needed? Backfill needed?
- Retention / deletion policy for new data?

## Permissions & access

- Who is allowed to perform this action? How is that enforced?
- Are there per-resource permissions or org-level only?
- Audit trail: who did what, when, captured where?
- Service-to-service auth required?

## Concurrency & lifecycle

- What happens with two concurrent requests on the same resource?
- What happens if the user navigates away mid-action?
- What's the behavior on a partial failure midway through?
- Is there a "draft" or "in-progress" state? How does it expire?

## Edges & failure

- What happens when the dependency is down? Slow? Returning malformed data?
- What's the timeout? What's the retry policy? Where do retries happen?
- What's the user-visible behavior on each class of failure?
- What's the worst case — data loss, double-billing, leaked info?

## Integration boundaries

- Which other systems/services does this talk to?
- Sync or async? Pub/sub, request/response, polling?
- Are contracts versioned? What's the breaking-change story?
- Are there flaky upstreams that need circuit-breakers or fallbacks?

## UX (if user-facing)

- What states does the UI need: loading, empty, partial, error, success?
- Keyboard / accessibility expectations?
- Mobile parity? Offline behavior?
- Localized strings? Right-to-left?

## Testing

- How will this be tested — unit, integration, e2e?
- Are there fixtures or seed data needed?
- Are there flows that *can't* be tested automatically? How will we verify them?
