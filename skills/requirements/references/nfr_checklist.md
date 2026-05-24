# Non-Functional Requirements Checklist

For each category, ask the prompts that fit the feature. Skip categories that obviously don't apply, but say so out loud — silent skips hide gaps.

## Performance & scale

- Expected traffic / request rate at launch, in 6 months, in 2 years?
- Latency target — p50, p95, p99? Measured where (client, edge, server)?
- Payload size limits?
- Any expensive operations (joins, aggregations, fanout) in the request path?
- Caching strategy — what's cacheable, TTL, invalidation trigger?
- Cold-start sensitivity?

## Reliability & availability

- Availability target (e.g., 99.9%) — does it match dependencies?
- What's the blast radius if this breaks? Just this feature, or broader?
- Degraded mode — can the feature serve a reduced experience when a dep is down?
- Idempotency keys for retries on writes?
- Backpressure / queue overflow behavior?

## Security & privacy

- Does this handle PII, PHI, payment data, secrets, or auth tokens?
- Authn model — session, OAuth, service token, signed URL?
- Authz checks — at the route, at the data layer, both?
- New threat surface — public endpoint, file upload, SSRF risk, deserialization?
- Rate limiting / abuse prevention?
- Input sanitization — XSS, SQLi, path traversal, log injection?
- Secrets handling — vaulted, rotated, never logged?
- Data classification — does this change what classification level a system holds?

## Compliance & legal

- Any regulated data (GDPR, CCPA, HIPAA, PCI, SOC2)?
- Data residency constraints (must stay in region X)?
- Retention requirements — minimum or maximum?
- Right-to-deletion / right-to-export?
- Consent capture needed?
- Audit log retention requirements?

## Observability

- What metrics will tell us this is healthy? Unhealthy?
- What logs are essential for debugging at 3am? Structured?
- Are traces propagated through this code path?
- New alerts to wire up? Who gets paged?
- Dashboards updated?
- SLOs / error budget impact?

## Maintainability

- Who owns this code after merge?
- Where do docs live?
- How do you run this locally? Are there missing dev affordances?
- Is the design extensible in the directions we expect to extend it next?
- Are there knobs that should be config rather than hardcoded?

## Compatibility & migration

- Backwards-compatible with existing clients / API consumers?
- Schema migration plan — online, with backfill, with rollback?
- Feature flag — flag name, default, removal trigger?
- Deprecation plan for any old path being replaced?
- Mobile / external client release coupling?

## Cost

- Infra cost delta — new instances, storage, egress?
- Per-request cost if calling a paid API?
- Query cost — new heavy queries on a shared DB?
- Storage growth rate?

## Accessibility & i18n (user-facing only)

- WCAG level target?
- Keyboard navigation, screen reader labels?
- Color contrast?
- Localizable strings, plural rules, date/number formats?
- RTL layout support?

## Operational

- Runbook entries needed?
- Oncall handoff — does this change paging routes?
- Backup / restore implications?
- Disaster recovery — does this need to be in the DR plan?
