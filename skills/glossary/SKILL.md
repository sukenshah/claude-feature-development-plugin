---
name: glossary
description: Extract domain and project-specific terminology from the codebase and maintain a shared glossary section in CLAUDE.md. Use when the developer asks to build, update, refresh, or maintain a glossary; wants a shared vocabulary between code and team; says "what does X mean in this codebase"; mentions onboarding-friendly terminology; or invokes `/glossary`. Reads the code (whole repo or diff), checks the nearest CLAUDE.md for an existing glossary to avoid duplicates, proposes only high-level domain + project-specific technical terms (not generic CS jargon), and confirms with the developer before writing.
---

# Codebase Glossary

Your job: surface the terms developers and the code actually share — the domain entities, project-specific technical names, and internal concepts a new teammate would have to learn — and persist them as a short, high-level glossary in the nearest `CLAUDE.md`.

## What a good glossary entry looks like

- **One sentence per term.** A definition you could put on a hallway whiteboard. No paragraphs.
- **Names what the term means in *this* codebase**, not what it means in textbooks. "Customer" is generic; "Customer (this codebase): a billing-eligible account that owns at least one Workspace" is specific.
- Includes a short usage hint when ambiguous: "Account vs Customer — Account is the auth principal; Customer is the billing entity."
- Avoids implementation detail. Glossary is *what*, not *how*. Don't say "stored in `customers` table with index on `email`" — that's documentation, not glossary.

## What does *not* belong

- Generic CS terms (function, class, hashmap, mutex). The team already knows these.
- Implementation specifics (table schemas, function signatures, file paths). Those are docs, not vocabulary.
- One-off internal helpers (`buildUserList`, `formatDate`). Glossary is shared language, not API reference.
- Vendor product names everyone already recognizes (AWS, Postgres). Include only if used in a non-obvious internal way.
- Terms with only one occurrence in the codebase — wait until a concept has earned recurrence before naming it.

## Inputs

1. **Scope choice** — ask the developer up front via `AskUserQuestion`:
   - **Whole codebase** — comprehensive scan. Default when no glossary exists yet.
   - **Diff only** — branch vs `main`/`master`, or working-tree changes. Default when a glossary already exists (incremental maintenance).
   - Make the default recommendation visible in the question based on whether a glossary section already exists in any `CLAUDE.md`.

2. **Target `CLAUDE.md`** — nearest one to the changed/scanned code.
   - If only one `CLAUDE.md` exists in the repo, use it.
   - If multiple exist (monorepo), pick the one nearest the code being scanned. For whole-codebase mode, this is the root `CLAUDE.md`. For diff mode, find the closest ancestor `CLAUDE.md` to the changed files.
   - If multiple are roughly equidistant, ask the developer which one to update.
   - If no `CLAUDE.md` exists at all, ask the developer where to create it (default: repo root).

## Workflow

### Phase 1 — Discover existing glossary

Read every `CLAUDE.md` you might write to. Look for an existing glossary section. Common headings:

- `## Glossary`
- `## Terminology`
- `## Vocabulary`
- `## Domain Terms`

Capture the existing entries. These are the **dedup baseline** — never re-propose a term that's already defined unless the developer explicitly asks to refresh it.

If no glossary section exists, plan to create one with the heading `## Glossary` (use this exact heading for consistency across the plugin).

### Phase 2 — Resolve scope

Ask the developer using the scope options above. Apply their choice:

- **Whole codebase** — Glob the source tree (excluding `node_modules`, `vendor`, `dist`, `build`, `.git`, lockfiles, and other generated paths). For large repos, prefer spawning an Explore subagent for the discovery sweep so the main context stays focused.
- **Diff only** — `git diff main...HEAD --name-only` (or `master...HEAD`), plus working-tree changes if dirty. Read only files in that set, but also briefly read 1-hop neighbors (callers/callees of changed symbols) since new terms often hide there.

### Phase 3 — Extract candidate terms

Walk the in-scope files and harvest candidates from these signals:

- **Type/class/interface/struct/trait names** that recur across files — strong signal of a shared concept.
- **Module/package/directory names** in the source tree.
- **Recurring identifiers** in function names that aren't generic verbs (e.g., `assignWorkspace`, `revokeEntitlement` → `Workspace`, `Entitlement`).
- **String constants / enums** that represent domain states or kinds (`STATUS_DELINQUENT`, `Tier.PLATINUM`).
- **Acronyms and abbreviations** used as identifiers (`ARR`, `MRR`, `IDP`, `LOA`) — explain what they stand for.
- **Comments and docstrings** that explain "this is the X" or "an X is Y" — the previous author already did some of your work.
- **README / docs in the repo** — often surface intended vocabulary the code drifted from. Worth checking but the code wins ties.
- **Configuration keys / feature flag names** when they encode product-side concepts.

For each candidate, note:

- Where it appears (a couple of representative file:line refs).
- Whether it's a thing (entity), a state, a role, an action, or a system.

### Phase 4 — Filter to glossary-worthy

Apply these gates in order. A term must pass all to make the proposal list.

1. **Recurrence gate** — appears in at least 3 distinct files, or is clearly the name of a core entity / subsystem even if newer.
2. **Specificity gate** — the meaning isn't obvious to a developer who has read mainstream docs for the languages and frameworks in use. If the meaning *is* obvious, drop it.
3. **Project-bound gate** — the term carries a meaning *specific to this codebase or its domain*, not a generic CS or industry concept. Generic CS terms drop here.
4. **Dedup gate** — not already in the existing glossary (case-insensitive, also catch obvious variants like `Account` vs `Accounts`).
5. **High-level gate** — can be defined in one sentence without referencing internal implementation. If you can't, the term is probably too low-level to be glossary material; drop it.

A reasonable end state for a first run on a medium codebase: 10–30 terms. Far more than that signals the gates aren't strict enough. Far fewer signals you under-scanned.

### Phase 5 — Draft definitions

For each surviving term, write:

- The term (capitalized as it appears in the domain).
- One-sentence definition in *this codebase's* meaning.
- (Optional) a "vs <other-term>" disambiguator if the term collides with a near-neighbor.
- (Optional) the expansion if it's an acronym.

Lean on the code, but write for a human. Look at the type definition and at how the type is used. Usage often reveals meaning that the declaration alone doesn't.

When uncertain, flag the term as **(needs developer confirmation)** rather than guessing. Better to ask than to write a wrong definition into a file teammates will read.

### Phase 6 — Confirm with developer

Present the proposed glossary to the developer before writing. Use this layout:

```
Proposed glossary update for <path/to/CLAUDE.md>

Existing terms (kept as-is): <count>
New terms proposed: <count>
Terms needing your confirmation: <count>

New entries:
- **Term1**: <definition>
- **Term2**: <definition>
...

Needs your confirmation:
- **TermX**: <best-guess definition> — <why you're uncertain, e.g. "two competing meanings in the code">

Refresh existing (only if developer explicitly asked):
- ...
```

Ask three concrete things:

1. Definitions to keep / drop / reword.
2. Resolution for the "needs confirmation" terms.
3. Any terms you missed that the developer thinks belong.

Iterate once if the developer pushes back substantially. Don't loop more than that — the goal is a useful first cut, not a perfect one.

### Phase 7 — Write to CLAUDE.md

Update the target `CLAUDE.md`:

- If a `## Glossary` section exists: merge new entries alphabetically with existing ones. Keep existing wording unless the developer asked to refresh it.
- If no glossary section exists: append a new `## Glossary` section at the end of the file (or place it where the developer prefers — ask if the file has strong existing structure).
- Within the glossary, entries are alphabetical, bold-term-then-definition format:

  ```markdown
  ## Glossary

  - **Entitlement**: a grant that allows a Customer to use a specific Product Tier; revocable independently of the underlying subscription.
  - **Workspace**: an isolated tenant container owned by exactly one Customer; the unit of permission boundary in this codebase.
  - ...
  ```

- Keep the section compact. If it grows past ~40 entries, suggest the developer split it (e.g., domain vs technical), but don't split unilaterally.

### Phase 8 — Report

Tell the developer:

- File updated and entry count delta (added / unchanged / refreshed).
- Any "needs confirmation" terms that were deferred (with the developer's resolution noted).
- Suggested next step (e.g., re-run in diff mode after the next feature, or refresh after a domain rename).

## Conduct rules

- **Read the code, don't invent terms.** Every proposed term must be backed by concrete file:line evidence.
- **Don't over-define.** Glossary is a shared vocabulary, not a wiki. One sentence per term; if it needs more, it belongs in docs.
- **Never silently overwrite an existing definition.** If you think an existing entry is wrong, surface it as a "refresh candidate" with reasoning — let the developer decide.
- **Dedup is sacred.** Re-proposing existing terms is the fastest way to make this skill annoying. Trust the existing glossary; only revisit on explicit request.
- **Don't moralize on naming.** If the codebase uses awkward terminology, document the awkward term as-is. Renaming is a separate task.
