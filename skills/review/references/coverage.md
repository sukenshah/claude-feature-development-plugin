# Coverage Checklist — Primary / Alternate / Edge

The spec lists the FRs. The diff implements *something*. This checklist helps you verify the something matches the FRs across the flows that matter, not just the happy path.

For each FR in the spec, work through three levels.

## 1. Primary flow

The intended happy path. Most-common inputs, most-common actors, no errors.

Questions:

- Is there code that implements it? Point to the file:line.
- Is there a test that exercises it end-to-end (or as close as the codebase tests reach)?
- Does the observable behavior actually match the FR's wording? Don't accept "close enough" — read both, compare directly.
- Are the success outputs (return value, side effects, downstream events) the ones the spec called for?

## 2. Alternate flows

Variations the spec calls out explicitly: different actors, different inputs, different entry points, different states.

Questions:

- For each alternate the spec named, is it implemented? Tested?
- Are there alternates the spec implied but didn't call out (e.g., "admin can also do X" implies admin auth check exists, even if the spec only described the user case)?
- If the feature has both a UI and an API entry, both go through the same logic (or there's a deliberate reason they don't)?

## 3. Edge cases

The places real systems break. Walk this list against the diff's domain — most won't apply, but you should reach for each consciously rather than skip silently.

### Input edges

- Empty input (null, "", `[]`, `{}`)
- Single-element input where logic was written for many
- Maximum-size input (longest string, largest payload, deepest nesting)
- Boundary values (0, 1, -1, max int, min int, off-by-one)
- Unicode (emoji, RTL text, combining characters, normalization)
- Whitespace (leading/trailing, only whitespace, tabs vs spaces)
- Case sensitivity
- Encoding (UTF-8 vs UTF-16, base64 padding)
- Locale-specific (decimal separators, date formats, plural rules)
- Timezone (UTC vs local, DST transitions)
- Numeric (floating-point precision, integer overflow, division by zero, NaN/Infinity)
- Malformed input (broken JSON, truncated, wrong types)
- Malicious input (oversized, recursive, injection payloads — see security checklist)

### State edges

- First-time state — user with no data yet, system bootstrap
- Empty result set, single result, large result set
- Pagination boundaries — first page, last page, exact-page-size, page beyond the end
- Stale state — cached value that's been invalidated upstream
- Concurrent state change — two writers modify the same resource

### Concurrency & lifecycle edges

- Two concurrent requests on the same resource
- User navigates away mid-action
- Retry of a request that already partially succeeded
- Idempotency key collision (same key, different request)
- Process restart mid-operation
- Background job interleaving with foreground request

### Failure edges

- Dependency timeout
- Dependency returns malformed data
- Dependency returns a different version's contract
- Dependency rate-limits
- Network partition (request sent, response lost)
- Disk full / quota exceeded
- Permission denied at a layer below where you checked

### Permission & auth edges

- Authenticated but unauthorized
- Authorized for one resource, tries another
- Token just expired
- Token revoked but cached client-side
- Cross-tenant access attempt

### Observability edges

- No log/metric for a path you'd need to debug at 3am
- Log emits PII or secrets
- Metric cardinality explosion (label includes user ID, request ID, etc.)
- Trace doesn't propagate across the new boundary

## How to score coverage in the report

For each FR/NFR in the spec, end at one of:

- **Covered** — implemented and tested across primary + relevant alternates + at least the obvious edges.
- **Partial** — implemented but missing tests, or implemented but skipping an edge the spec named, or implemented but tested only on the happy path for a feature that has obvious failure modes. Say what's missing.
- **Not addressed** — the spec asked for it, the diff doesn't deliver it. Blocker by default.
- **Out-of-scope-per-spec** — the area was touched but the spec explicitly excluded it. Informational.

Edge cases that the spec didn't name but should have:

- If they're real edges that will hit production, flag as Major (or Blocker if dangerous), and note the spec didn't mention them — that's useful feedback even though the spec isn't this skill's output.

## Test-vs-code coverage

A common gap: the code handles a case, but no test exercises it. That's still a coverage gap — untested code drifts. When flagging:

- "Implemented in `file.ts:42`, no test exercises the alternate path where `flag=false`."

Don't insist on tests for trivia, but for any path the spec called out by name, there should be a corresponding test (unit, integration, or e2e — codebase's convention).
