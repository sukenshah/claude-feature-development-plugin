# Security Review Checklist

Apply categories that match the diff. Skip ones that obviously don't apply, but don't skip silently in your head — confirm none of these surfaces were touched.

Findings here default to higher severity than other lenses — downside is asymmetric.

## Authentication & Authorization

- Every new endpoint or sensitive action has an explicit auth check. Don't trust "the router enforces it" without verifying.
- Authorization checked at the right layer — route-level alone is not enough for per-resource permissions; need data-layer or service-layer checks too.
- No "soft" checks via comments or naming conventions — must be code-enforced.
- Privilege escalation paths — can a user pass an ID for a resource they don't own and get it back?
- Service-to-service auth — internal endpoints exposed to anonymous callers via misconfig.
- Token/session handling — expiry, refresh, revocation.

## Input Handling

- **SQL injection** — parameterized queries everywhere; no string concatenation into queries; ORMs used correctly (raw escape hatches scrutinized).
- **Command injection** — exec/shell calls with any user input; prefer no-shell variants; argument arrays not strings.
- **NoSQL/LDAP/XPath injection** — same idea; check the driver's escape mechanism.
- **Log injection** — user input written to logs without sanitization can poison log search and break alerting.
- **Path traversal** — file paths from user input must be canonicalized and confined to allowed roots.
- **Unsafe deserialization** — never deserialize untrusted data with formats that allow code execution (Python's binary serializer, Java native serialization, YAML's full loader). Prefer JSON or other data-only formats.
- **SSRF** — outbound URLs from user input must validate against allowlist; block private IP ranges, link-local, metadata endpoints.
- **XXE** — XML parsers configured to disable external entities.
- **Prototype pollution** — JS object merges with user-controlled keys.
- **ReDoS** — user-supplied regex or text matched against complex regex.
- **Mass assignment / over-posting** — request bodies bound directly to models without allow-listing fields.

## Output Handling

- **XSS** — output encoded for the context (HTML, attribute, JS, URL, CSS). Templating engine's auto-escape on. Raw-HTML injection sinks flagged (e.g., framework escape hatches that bypass auto-encoding, direct DOM HTML writes).
- **Open redirect** — redirect target validated against allowlist.
- **Response splitting / header injection** — newlines stripped from values placed in headers.
- **Error message leakage** — stack traces, SQL errors, internal paths not surfaced to end users in prod.

## Crypto

- Standard library / vetted library used; no hand-rolled crypto.
- Algorithm choice — no MD5/SHA-1 for security purposes; no DES/3DES; no ECB mode.
- Key handling — no hardcoded keys; keys come from vault/env; key rotation possible.
- Nonces/IVs — generated, not reused; correct length.
- Randomness — CSPRNG modules for security, not non-cryptographic PRNGs.
- Password storage — bcrypt/argon2/scrypt with sane params; never plain SHA, never reversible.
- Constant-time comparison for secrets.

## Secrets & Credentials

- Nothing that looks like a token, key, password, or credential in source.
- `.env` / config files with secrets are gitignored; example files present.
- Secrets not logged. Common slip: logging the full request body when it contains an Authorization header.
- Secrets not returned in error messages or API responses.
- Secrets not exposed via debug endpoints or in client-side bundles.

## PII & Privacy

- Data minimization — does the new code collect/log more PII than it needs?
- Redaction in logs — emails, names, IDs masked where appropriate.
- Retention — new data has a deletion path that matches policy.
- Right-to-export, right-to-delete — does this new data participate?
- Cross-region data flow — if data residency applies.

## Sessions & Cookies

- Cookies set with `HttpOnly`, `Secure`, `SameSite` appropriate for context.
- Session fixation prevented — session ID rotates on login.
- CSRF protection on state-changing endpoints with browser/cookie auth.

## CORS

- No wildcard `Access-Control-Allow-Origin` combined with credentials.
- Allowed origins explicit; not reflected from `Origin` header without validation.

## Rate Limiting & Abuse

- Public endpoints have rate limits.
- Login / token-issuance endpoints have stricter limits (account enumeration, credential stuffing).
- Resource-expensive endpoints (exports, heavy queries) gated.

## Supply Chain

- New dependencies — quick check: maintained, reasonable size, known publisher, license compatible.
- Lockfile updated and committed.
- Pinned versions where the codebase otherwise pins.
- Post-install scripts from new deps scrutinized.

## File Uploads

- Content-type and extension validated against allowlist (not blocklist).
- Magic-byte / actual-content check, not just trust the extension.
- Size limits enforced.
- Stored outside webroot or with no-execute.
- Filename sanitized.

## Infrastructure-Adjacent

- IAM changes — over-broad permissions, wildcards in resource ARNs.
- New cloud resources public by default — buckets, queues, databases.
- Network egress — outbound calls to new hosts.
- Logging destinations — sensitive data flowing to a less-secure log sink.

## Quick Wins to Always Check

- Any dynamic-code-construction primitive (eval, Function constructor, etc.)
- Any string built into a shell command or query
- Any HTTP call to a host derived from user input
- Any redirect target derived from user input
- Any new dependency
- Any new public endpoint
- Any new env var or config value
- Any code branch gated on `NODE_ENV` / `DEBUG` / similar that might behave differently in prod
