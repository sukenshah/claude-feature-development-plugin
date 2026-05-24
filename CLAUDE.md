# CLAUDE.md — feature-development plugin

This file is for **Claude when working inside this plugin's own source repo** (editing the skills, command, templates, references). End users of the plugin do not see this file — it's repo-local.

## What this repo is

A Claude Code plugin that produces a feature-development pipeline:

```
/requirements → spec.md → /design → plan.md → /execute → code+tests → review skill → findings
```

Plus a `glossary` skill that maintains shared vocabulary in CLAUDE.md files of consuming repos.

Each pipeline stage reads the **previous stage's persistent artifact**, not the previous stage's conversation. That's the load-bearing design choice — preserve it.

## Layout

```
.claude-plugin/plugin.json       # plugin manifest
commands/execute.md              # slash command (parent-orchestrated implementation)
skills/<name>/SKILL.md           # skill entry point (always loaded when triggered)
skills/<name>/assets/            # templates copied into output (e.g., spec/plan structure)
skills/<name>/references/        # reference docs loaded on-demand by the skill
```

## Conventions when editing skills

- **Frontmatter `description` is the only triggering mechanism.** It must include both *what the skill does* and *when to trigger*. Be specific about user phrases that should match. Skills tend to under-trigger; lean slightly pushy.
- **Keep `SKILL.md` under ~500 lines.** Anything longer goes into `references/<topic>.md` with a clear pointer from `SKILL.md` ("see `references/x.md` for ...").
- **Templates live in `assets/`.** The skill should reference them by path so the model fills them in rather than reinventing structure each run.
- **Imperative voice.** "Read the spec. Extract FRs." not "you should consider reading the spec."
- **Explain the *why*, not just the *what*.** Heavy-handed `MUST` / `ALWAYS` are a yellow flag — reframe with reasoning so the model can judge edge cases.
- **No comments in skill content beyond what a human reader needs.** Skills are read by the model and (often) the developer; redundant narration costs tokens.

## Pipeline contract (do not break)

The four artifacts must stay traceable and inter-readable. Specifically:

- **`spec.md`** uses numbered identifiers: `FR-N`, `NFR-N`, `A-N` (assumption), `R-N` (risk), `Q-N` (open question). Design and review skills reference these by ID.
- **`plan.md`** uses `D-N` for design decisions (with tenets-in-tension cited) and ordered `Step N` entries with concrete "Done when" criteria. `/execute` and `review` both depend on this.
- **`review` skill must remain plan-blind.** Never add a reference from `review/SKILL.md` to `plan.md`. Independence is the entire point of the review stage.
- **`/execute` subagent prompts must keep code and test agents isolated.** Code agent forbidden from test paths; test agent forbidden from production paths. Communication strictly via `spec.md` + `plan.md` + orchestrator reconciliation.

## Adding a new skill to this plugin

1. Create `skills/<new-skill-name>/SKILL.md` with frontmatter (`name`, `description`).
2. If it produces durable artifacts, place templates in `skills/<new-skill-name>/assets/`.
3. If it has reference material the model loads only when needed, place in `skills/<new-skill-name>/references/`.
4. Wire any pipeline relationship explicitly — if the new skill consumes an existing artifact, state which IDs/sections it reads. If it produces an artifact other skills will consume, document the schema.
5. Update [README.md](README.md) — add an entry under "Pipeline at a glance" if relevant, and a short "Usage" subsection.

## Adding a slash command

1. Create `commands/<name>.md` with frontmatter (`description`, optional `argument-hint`).
2. Slash commands are discovered automatically by the plugin loader — no manifest update needed.
3. Slash commands can invoke skills via the `Skill` tool and use any tools available in the session.

## Editing `/execute`'s orchestration

`commands/execute.md` parent-orchestrates subagents. Things to preserve when editing:

- **Mode selection (Parallel / Sequential / Code-only) happens once per run**, not per step. Per-step exceptions are within the chosen mode.
- **Code vs test agent isolation is non-negotiable.** Always include explicit path exclusions in each agent's prompt.
- **Worktree decision based on file-overlap heuristic.** Don't always use worktrees (cost) and don't never use them (correctness). Detection logic must stay in 4c.
- **Reconciliation step (4e) is Parallel-only.** Sequential and Code-only skip it. Don't add it back to those modes — they don't have drift.
- **Retry cap of 2 per step.** Looping forever on a failing step buries the user. Escalate instead.

## Validation

Before releasing changes:

```bash
claude plugin validate
```

Run inside the plugin repo. This catches:

- Missing/malformed frontmatter on skills.
- Bad `plugin.json` shape.
- Component path issues.

## Versioning

Bump `version` in `.claude-plugin/plugin.json` on every release. Tag the repo to match (e.g., `git tag v0.2.0`). Marketplaces pointing at the repo resolve by tag.

## Things to avoid

- **Reading `plan.md` from within `review/SKILL.md`.** Breaks the independence guarantee.
- **Adding "what" comments to skill content.** Skills should read clean; commentary belongs in this CLAUDE.md or the README.
- **Cross-skill imports.** Each skill is self-contained. If two skills need the same reference material, duplicate it — coupling skills via a shared file makes them harder to extract.
- **Long monolithic SKILL.md files.** Use `references/` aggressively. The model only loads referenced files when relevant.

## Glossary

- **Artifact**: a persistent file the pipeline produces (`spec.md`, `plan.md`, glossary entries in `CLAUDE.md`). Carries numbered IDs for cross-stage reference.
- **Code agent / Test agent**: subagents spawned by `/execute`; mutually exclusive scopes (production vs test code).
- **Drift / Reconciliation**: divergence between code and test interpretations of the plan's contracts, reconciled by the `/execute` orchestrator in Parallel mode.
- **Lens (review)**: one of four independent passes — best practices, security, anti-patterns, coverage.
- **Mode (execute)**: Parallel / Sequential / Code-only — chosen once per `/execute` run.
- **Tenet**: one of the five design principles (Simplicity, DRY, YAGNI, Modularity, Separation of Concerns) the `design` skill arbitrates between.
