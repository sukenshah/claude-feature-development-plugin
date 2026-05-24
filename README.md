# feature-development

A Claude Code plugin that turns a fuzzy feature idea into shipped code through a four-stage pipeline of durable, numbered artifacts.

```
/requirements  →  spec.md
       ↓
/design        →  plan.md
       ↓
/execute       →  code + tests (parent-orchestrated subagents)
       ↓
review skill   →  independent findings against the spec
```

Plus a `glossary` skill that keeps `CLAUDE.md` populated with the shared vocabulary between team and codebase.

## Why this exists

Most planning workflows produce ephemeral context that lives in a single conversation and disappears. This plugin produces **persistent, numbered artifacts** (`FR-1`, `NFR-3`, `D-2`) you can hand off, review against, and trace back to during incidents. Every stage reads the previous stage's artifact, not the previous stage's conversation.

## Pipeline at a glance

| Stage | Skill / Command | Reads | Writes |
|---|---|---|---|
| Define what to build | `requirements` skill | Code (for context) | `specs/<slug>/spec.md` |
| Decide how to build it | `design` skill | `spec.md` + code | `specs/<slug>/plan.md` |
| Build it | `/execute` command | `plan.md` + `spec.md` | code & tests |
| Verify against intent | `review` skill | `spec.md` + diff (NOT plan.md) | inline findings |
| Maintain vocabulary | `glossary` skill | Code + existing `CLAUDE.md` | `CLAUDE.md` glossary section |

## Install

### Option 1 — Own marketplace (recommended for teams)

Push this repo to GitHub, then set up a separate marketplace repo (template included if you want to bootstrap one). Engineers install once:

```
/plugin marketplace add <your-org>/<marketplace-repo>
/plugin install feature-development
```

### Option 2 — Local install (for testing or solo use)

```
/plugin install /path/to/feature-development
```

Or add to `~/.claude/settings.json`:

```json
{
  "plugins": {
    "feature-development": {
      "path": "/absolute/path/to/feature-development"
    }
  }
}
```

### Option 3 — Per-project install

Add to the target repo's `.claude/settings.json` and commit:

```json
{
  "plugins": {
    "feature-development": {
      "path": "../path/to/feature-development"
    }
  }
}
```

The whole team picks it up on clone.

## Usage

### 1. `requirements` skill — gather requirements

Triggers when the developer wants to scope or spec a feature: "let's spec X", "gather requirements for Y", "I need to add a feature."

- Asks for code pointers first (or accepts "no code" for greenfield).
- Runs a hybrid interview: batched easy questions, deep-dive on ambiguity.
- Sizes the spec adaptively (Lean / Standard / Comprehensive).
- Surfaces risks, assumptions, anti-patterns; tests that every FR is concrete.
- Writes `specs/<feature-slug>/spec.md`.

### 2. `design` skill — produce an implementation plan

Triggers on "design this", "create an implementation plan", "architect this feature."

- Reads `spec.md` end-to-end. Kicks back if the spec is too thin to design from.
- Walks the relevant code, then makes architecture / data / contracts / seams decisions.
- Arbitrates the design tenets — **Simplicity, DRY, YAGNI, Modularity, Separation of Concerns** — and documents which tenet won each conflict.
- Surfaces only load-bearing trade-offs to the developer; decides small reversible choices autonomously.
- Writes `specs/<slug>/plan.md` and presents via `ExitPlanMode` as an approval gate.

### 3. `/execute` command — implement the plan

Run with `/execute` (auto-discovers `specs/*/plan.md`) or `/execute path/to/plan.md`.

Asks one question up front: how to sequence code + tests for the whole run.

- **Parallel** (default): code subagent and test subagent spawn together per step. Faster. Both work against the plan's documented contracts; orchestrator reconciles drift. Best when contracts in the plan are detailed.
- **Sequential (TDD)**: test agent writes failing tests first, then code agent reads those tests and implements until green. Best when contracts are sketched.
- **Code only**: no test agent. For test-exempt work.

Detects file overlap automatically — uses git worktrees when needed (e.g. Rust `#[cfg(test)]` blocks in source files), shared tree otherwise. Falls back to inline execution when subagents aren't available. Composes with the `superpowers` plugin's execution skills when present.

### 4. `review` skill — independent code review

Triggers on "review this", "code review", "review the diff."

- Reads `spec.md` to understand intent. **Deliberately ignores `plan.md`** — independence is the whole point.
- Detects diff scope adaptively (branch vs main, or working-tree).
- Applies four lenses: missing best practices, security, anti-patterns, coverage (primary / alternate / edge per FR).
- Reports inline with `Blocker / Major / Minor / Nit` severity, every finding tied to `file:line` with reasoning and a suggested direction.

### 5. `glossary` skill — maintain shared vocabulary

Triggers on "build a glossary", "what does X mean in this codebase", or `/glossary`.

- Scans whole codebase or recent diff (developer chooses).
- Finds the nearest `CLAUDE.md` to update.
- Filters candidates through five gates (recurrence, specificity, project-bound, dedup, high-level) so only useful terms survive.
- Confirms proposed entries with the developer before writing.

## Repo layout

```
feature-development/
├── .claude-plugin/plugin.json
├── commands/
│   └── execute.md                  # /execute slash command
├── skills/
│   ├── requirements/
│   │   ├── SKILL.md
│   │   ├── assets/spec_template.md
│   │   └── references/{question_bank,nfr_checklist}.md
│   ├── design/
│   │   ├── SKILL.md
│   │   ├── assets/plan_template.md
│   │   └── references/tenets.md
│   ├── review/
│   │   ├── SKILL.md
│   │   └── references/{security,anti_patterns,coverage}.md
│   └── glossary/
│       └── SKILL.md
└── README.md
```

## Customizing

- **Template tweaks** — edit `skills/<skill>/assets/*.md` to change spec / plan section structure. The skills reference these templates by path.
- **Question banks / checklists** — edit `skills/<skill>/references/*.md` to adapt to your team's specific concerns (e.g., add a `compliance.md` reference if you're in a regulated industry).
- **Tenets** — `skills/design/references/tenets.md` encodes the five design tenets and arbitration heuristics. Fork this file for team-specific principles.
- **Severity calibration** — `skills/review/SKILL.md` defines what each severity means. Tighten or loosen to match your team's review culture.

## Relationship to other plugins

- **`superpowers`** — broad process discipline across many situations (TDD, debugging, verification, etc.). This plugin layers artifact-driven feature flow on top. `/execute` delegates to `superpowers:executing-plans` and related skills when present.
- **`claude-md-management`** — improves and audits `CLAUDE.md` content generally. Pairs naturally with `glossary` (which produces glossary entries) — `claude-md-improver` can then audit the result.

## License

MIT — see [LICENSE](LICENSE).
