---
description: Execute an implementation plan by parent-orchestrating parallel code and test subagents per step. Reads plan.md and spec.md, spawns a code agent and a test agent in parallel for each step, handles file overlap via git worktrees, then merges and verifies. Falls back to inline execution if subagents are unavailable.
argument-hint: "[path/to/plan.md] — optional; auto-discovered if omitted"
---

# /execute — Parent-orchestrated implementation

You are the **orchestrator**. You do not write code or tests yourself. Your job: read the plan, dispatch a code subagent and a test subagent in parallel per step, manage isolation, merge their work, and verify against the spec.

The discipline that makes this work:

- **Code agent never touches tests.** Test agent never touches production code. Both read the same spec + plan, work against the documented contracts, and meet in the middle.
- **Parallelism per step** unless step-N depends on a side effect from step-(N-1) (schema migration before code that uses it, etc.). Sequential when the dependency is real, parallel within a step always.
- **Worktrees only when needed.** If the code agent's files and the test agent's files don't overlap, run in shared tree. If they overlap (e.g., Rust `#[cfg(test)] mod tests` blocks in the same file, Python tests next to source), use worktrees and merge.

## Step 1 — Locate plan and spec

Resolve the plan file:

1. If `$ARGUMENTS` is non-empty, treat as plan path. Read it.
2. Otherwise: `ls specs/*/plan.md 2>/dev/null`
   - One match → use it.
   - Multiple → ask via `AskUserQuestion`.
   - Zero → tell user no plan found; suggest `/design` first. Stop.

Read the plan in full. Also read the linked `spec.md` in same directory — the test agent will rely on FRs, NFRs, and edge-case notes from there.

## Step 2 — Pre-flight

1. `git status --porcelain` — working tree must be clean. If dirty, stop and surface to user.
2. Confirm the test command for this repo (read `package.json` / `pyproject.toml` / `Makefile` / README). Ask the user once if unclear; remember for the run.
3. `TodoWrite` one todo per step from plan's "Implementation Steps" section, keeping the same numbering.

## Step 3 — Detect subagent availability

If you don't have the Agent tool available in this session, skip to **Fallback** at the bottom.

If subagents are available, the orchestrated flow below is the default.

## Step 3.5 — Choose execution mode

Ask the user via `AskUserQuestion` how they want code + tests sequenced. One choice for the whole run; remember it.

Options to present:

- **Parallel (default)** — Code and test subagents spawn together per step. Faster wall-clock. Both work against the plan's contracts; orchestrator reconciles drift. Best when plan contracts are precise.
- **Sequential (TDD)** — Test agent goes first per step, commits failing tests. Code agent then reads those tests + plan + spec, implements until green. Slower but tests literally define the contract; less reconciliation needed. Best when plan contracts are sketched rather than detailed, or when the developer wants stricter TDD discipline.
- **Code only** — No test agent. Only spawn code agent. Use when the user explicitly says "no tests" or the work is genuinely test-exempt (config, scripts, docs).

Default the recommendation to **Parallel** if the plan's Contracts section is detailed (signatures, shapes, error envelopes named). Recommend **Sequential** if the plan's Contracts section is sketched or missing — let the tests pin down what the code agent has to deliver.

## Step 4 — Per-step orchestration loop

For each step in order:

### 4a. Read the step

Re-read the plan's step block. Pull out:

- Files to touch (production code paths)
- Detail level (Detailed / Sketched / One-line)
- The "Changes" description and "Done when" criterion
- Any contracts/signatures the plan called out for this step (look at plan section 4)

### 4b. Decide dispatch shape for this step

The user's mode from Step 3.5 sets the default. Per-step exceptions:

**If user chose Parallel:**

- **Parallel pair (default):** step adds or changes behavior. Spawn code + test in parallel.
- **Solo code agent (no test agent):** step is pure refactor with no behavior change, or step is config/wiring with no logic to test. Spawn only the code agent.
- **Solo test agent:** step is adding tests to existing untested behavior. Rare. Spawn only the test agent.
- **Forced sequential within step:** only if the test agent literally cannot write tests without first seeing the new signature (e.g., generated types). Prefer to fix the plan's contracts section so this isn't needed. If unavoidable, run code agent first, then test agent.

**If user chose Sequential (TDD):**

- Spawn test agent first. Wait. Test agent commits failing tests.
- Then spawn code agent with the committed test files referenced in its prompt as the contract.
- For pure refactor / config / wiring steps with no testable behavior, skip the test agent and spawn only the code agent — same as Parallel mode's solo-code case.

**If user chose Code only:**

- Spawn only the code agent for every step. No test agent ever. Skip the reconciliation sub-step (4e) since there's nothing to reconcile.

### 4c. Detect file overlap

Compute the test agent's likely file set from codebase convention:

- Separate test tree (`tests/`, `__tests__/`, `*.test.ts`, `*.spec.ts`, `*_test.go`, `test_*.py`) → no overlap with code paths.
- Tests co-located in source files (Rust `mod tests`, doctests, inline pytest fixtures) → overlap with code paths.
- Mixed → assume overlap for those specific files.

Decision:

- **No overlap** → both subagents work in the shared tree concurrently.
- **Overlap** → use git worktrees. Create one worktree per agent off the current branch; agents work in isolation; orchestrator merges back.

### 4d. Spawn the subagents

**Parallel mode**: send code + test agents in a single message with multiple Agent tool calls so they run concurrently.

**Sequential (TDD) mode**: spawn test agent first, wait for it to commit, then spawn code agent in a second message. The code agent's prompt must reference the committed test files as the contract to satisfy.

**Code only mode**: spawn only the code agent.

Each subagent gets a self-contained prompt — they have no shared conversation context with you or each other.

#### Code agent prompt template (Parallel mode)

```
You are the CODE subagent for step <N> of an implementation plan.

Plan file: <abs path to plan.md>
Spec file: <abs path to spec.md>
Step number in plan: <N>
Working directory: <repo path or worktree path>

Your job:
1. Read plan.md and spec.md in full.
2. Implement step <N> ONLY. Do not modify earlier or later steps.
3. STRICTLY no test files. Do not create, modify, or delete anything under:
   <list of test paths/patterns for this codebase>
   If the codebase co-locates tests in source files, do not modify the test-block
   regions of those files. A parallel test subagent owns all test code.
4. Match existing codebase conventions (naming, error handling, logging).
5. Implement to the contracts in plan.md section 4 EXACTLY — the test subagent
   is writing tests against those contracts in parallel and cannot see your code.
6. Match function/method signatures specified in plan.md. If the plan doesn't
   pin a signature you need, choose the obvious one and report it in your result
   summary so the orchestrator can reconcile.
7. Do not run tests yourself (you can't — they may not exist yet, or the test
   agent hasn't finished). Run only static checks (typecheck, lint, build).
8. Commit your changes when done with subject "step-<N>: <goal>".

Report back:
- Files changed (paths + brief description)
- Any signatures you had to invent (not in the plan)
- Static check results (typecheck / lint / build output)
- Anything you deviated from in the plan, with reasoning
- Open questions for the orchestrator
```

#### Code agent prompt template (Sequential / TDD mode)

```
You are the CODE subagent for step <N> of an implementation plan.

Plan file: <abs path to plan.md>
Spec file: <abs path to spec.md>
Step number in plan: <N>
Working directory: <repo path>

A test subagent has already written failing tests for this step. They are
committed at HEAD. Read them — they define the contract you must satisfy.

Test files for this step:
<list of test file paths the test agent reported>

Your job:
1. Read plan.md and spec.md in full for context.
2. Read every test file listed above. The tests are the precise contract.
3. Implement step <N> production code to make those tests pass. Do not modify
   the tests under any circumstances — if a test seems wrong, surface it in
   your report; do not "fix" it.
4. STRICTLY no test files. Same exclusions as parallel mode.
5. Match existing codebase conventions.
6. Run the tests yourself as you go. Iterate until they pass. Then run the
   full suite to confirm no regressions.
7. Commit when green with subject "step-<N>: <goal>".

Report back:
- Files changed
- Final test run output (must be all green)
- Any tests you believe are wrong (do not edit them; flag them)
- Anything you deviated from in the plan, with reasoning
- Open questions for the orchestrator
```

#### Test agent prompt template

```
You are the TEST subagent for step <N> of an implementation plan.

Plan file: <abs path to plan.md>
Spec file: <abs path to spec.md>
Step number in plan: <N>
Working directory: <repo path or worktree path>

Your job:
1. Read spec.md and plan.md in full.
2. Write tests for the behavior step <N> introduces or changes. ONLY tests.
3. STRICTLY no production code. Do not modify anything outside test paths:
   <list of test paths/patterns for this codebase>
   A parallel code subagent owns all production code.
4. Tests must be written against the spec's FRs and the plan's contracts —
   you cannot see the code subagent's work. Treat plan.md section 4 (Contracts)
   as the API you're testing against.
5. Match the codebase's test style (framework, naming, fixtures, mocking).
6. Cover:
   - Primary flow (happy path) for every FR this step delivers
   - Alternate flows the spec/plan called out
   - Edge cases relevant to this step's domain
   - Per the spec's NFRs where unit-testable (e.g., input validation, error mapping)
7. If a contract in the plan is ambiguous, write the test against the most
   sensible interpretation and report the ambiguity. The orchestrator will
   reconcile if the code subagent interpreted it differently.
8. Run the tests once to confirm they FAIL for the expected reasons (no
   implementation yet). In Parallel mode this confirms test wiring. In
   Sequential / TDD mode this is the red phase that the code subagent
   will turn green next.
9. Commit with subject "step-<N>: tests".

Report back:
- Test files created/modified (absolute paths)
- Coverage summary: which FRs / contracts each test exercises
- Any contract ambiguities you encountered and how you interpreted them
- Failing test output (to confirm tests run and fail for the right reasons)
- Open questions for the orchestrator
```

### 4e. Reconcile (Parallel mode only)

**Sequential mode skips this** — the code agent already ran tests green against the test agent's committed contract, so there's nothing to reconcile. **Code-only mode also skips** — no test agent to drift against.

**Parallel mode**: when both subagents complete:

1. If worktrees were used, merge each worktree back onto the working branch. Resolve conflicts deliberately — never `--ours` / `--theirs` blindly.
2. Read both agents' reports.
3. **Reconcile contract drift.** Compare:
   - Signatures the code agent invented vs signatures the test agent assumed.
   - Contract ambiguities the test agent flagged vs how the code agent implemented them.
4. If they diverge:
   - **Small drift, obvious fix** → patch the test or code yourself (the orchestrator can edit). Update the plan's Contracts section to reflect what landed so the record is true.
   - **Material drift** → don't paper over it. Either re-dispatch the affected subagent with a clarified contract, or stop and surface to the user.

### 4f. Run the tests

**Sequential mode**: the code agent already ran tests green before committing. Just rerun the full suite for regression check.

**Parallel mode**: now run the actual test command for this step's scope (just the new tests if the test runner supports targeting; otherwise the full suite).

**Code only mode**: run whatever verification command applies (typecheck, build, manual smoke per plan's "Done when" criterion).

- All pass → step is done. Mark the todo completed.
- Some fail:
   - Failures clearly the code's responsibility → spawn the code agent again with the failing test output as context.
   - Failures clearly the test's responsibility (over-strict assertion, wrong expectation) → spawn the test agent again with reasoning.
   - Failures from a real mismatch in contract → reconcile per 4e, then re-spawn.
- Don't loop forever. After 2 retries on the same step, stop and surface to user.

### 4g. Checkpoint

Tell the user: step N done, what landed, what's next. One line. Then proceed to step N+1.

## Step 5 — Final verification

After all steps complete:

1. Run the full test suite. Read the output.
2. Re-read the spec's FRs and NFRs. For each, state whether the built thing satisfies it (one line each). Surface anything that doesn't.
3. Update plan's `Status:` header to `Complete`.
4. Report to user: steps done, deviations from plan, unmet spec items, suggested next move (PR, deploy).

## Handling deviations

If during execution either subagent reports the plan is wrong:

- Small (wrong file, missed dep, minor contract tweak) → orchestrator decides, records in plan's "Open Questions / Checkpoints" so the record stays honest.
- Material (design decision doesn't survive contact with code) → stop. Surface to user. Kick back to `/design` rather than improvise.

## Worktree mechanics

When file overlap forces worktrees:

```bash
# Create worktrees off current branch
git worktree add ../.<repo>-code-step-N HEAD
git worktree add ../.<repo>-test-step-N HEAD

# Pass each path as the agent's working directory.

# After both complete, merge back:
git -C ../.<repo>-code-step-N format-patch -1 --stdout | git am
git -C ../.<repo>-test-step-N format-patch -1 --stdout | git am
# Or cherry-pick if cleaner.

# Clean up:
git worktree remove ../.<repo>-code-step-N
git worktree remove ../.<repo>-test-step-N
```

Don't leave worktrees around between steps — create per step, remove after merge.

## Coexistence with superpowers

If `superpowers:executing-plans` / `subagent-driven-development` / `test-driven-development` are available, you can still drive the parent-orchestration above — those skills enforce good per-agent discipline (TDD, verification) which strengthens what each subagent does. Don't replace this orchestration with the superpowers skills; layer them inside the subagent prompts where helpful.

If `superpowers:systematic-debugging` is available and a step fails repeatedly, invoke it before re-spawning subagents.

## Fallback — no subagents available

If you don't have the Agent tool:

1. Per step, do it yourself with the same discipline: write the tests first (against the plan's contracts), commit, then write the code, run tests, fix, commit.
2. Use the plan as source of truth across the run.
3. Same final verification and reporting as Step 5.

## Things to avoid

- **Letting subagents see each other's WIP.** That defeats the contract-driven separation. They communicate via spec + plan only, with the orchestrator as the only point of reconciliation.
- **Skipping the overlap check.** Two agents editing the same file without worktrees produces broken merges.
- **Patching contract drift silently.** When you reconcile in 4e, write what landed back into the plan so the record matches reality.
- **Running tests before merging.** If the agents committed in separate worktrees, run tests only after the merge.
- **Looping on failures forever.** Two retries max per step; then escalate to user.
