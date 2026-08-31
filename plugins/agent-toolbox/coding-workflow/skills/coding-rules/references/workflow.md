# Work Process & Verification Flow

## Overview

Every task follows three phases: **Plan → Implement → Verify**.

These phases are split across the client/contractor boundary defined in `agents.md`:

| Phase | Owner | What that side does |
|-------|-------|---------------------|
| Plan | Main session (client) | Fixes requirements and acceptance criteria, gets user approval, writes the work order. Design detail may be delegated to **planner**, but the requirements and the acceptance criteria stay with the client |
| Implement | Subagent (contractor) | Implements the work order with TDD. The client does not write code or tests here |
| Verify | **evaluator**, then the client | The evaluator runs the DoD gate and reports a verdict; the client judges the deliverable against the acceptance criteria |

## Phase 1: Plan

- Present the work approach as a proposal, not a fait accompli
- Acceptance criteria are the client's deliverable for this phase. Write them before delegating — a work order without them is incomplete (see `agents.md`)
- A request for an opinion ("which design is better?", "〇〇だとどれ？") is not approval. Answer with a recommendation, then ask for the decision — never treat your own reasoning as the approval that unlocks implementation
- Define clear, testable completion criteria
- Obtain user approval before starting implementation
- Surface dependencies, risks, and reversibility concerns

## Phase 2: Implement

This phase belongs to the contractor. The client hands over the work order and stops touching production and test code, apart from the minor-change exception in `agents.md`. Everything below binds whoever holds the work order.

### Follow existing patterns

- Read surrounding code before writing new code
- Match the project's conventions (naming, structure, error handling, module layout)
- Do not introduce custom patterns to work around standard project tooling
- Verify folder structure before creating new files; add new files only when existing structure genuinely cannot host the change

### Task runner is the source of truth (CRITICAL)

Every project defines a task runner (taskipy, npm/pnpm scripts, mise, make, etc.). Lint / format / test / type-check MUST be executed through it. Never invoke the underlying tool directly — options, target paths, and tool versions live in the task runner definition, and direct execution silently uses a different scope than CI.

| Ecosystem | Task runner call | Direct tool call (prohibited) |
|-----------|------------------|-------------------------------|
| Python (uv + taskipy) | `uv run task lint` / `test` / `format` / `type-check` | `uv run ruff check .`, `uv run mypy .` |
| Node.js (pnpm) | `pnpm lint` / `pnpm test` | `pnpm exec eslint .`, `npx tsc` |
| Node.js (npm) | `npm run lint` / `npm test` | `npx eslint .` |
| mise | `mise run lint` / `mise run test` | direct binary invocation |
| Makefile | `make lint` / `make test` | direct binary invocation |
| Terraform | (no task runner) — see `terraform` skill DoD | — |

If no task runner exists in the project, the first task is to define one; not to work around its absence with ad-hoc commands.

### Ad-hoc single-target execution

Running a single test file, targeting a single lint rule, or scoping a type-check to one module during iteration is allowed. This applies only while debugging — DoD verification always uses the task runner's full-project command.

Example: `uv run pytest tests/test_user.py::test_case` while debugging is fine; DoD still requires `uv run task test` against the whole project.

## Phase 3: Verify (Completion Gate)

### Who runs what

The contractor runs the DoD commands to finish its own work order. That run is not the gate. The gate is the **evaluator**, invoked by the client after delivery — it re-runs the DoD, reviews the diff, and returns PASS / REVISE / REDESIGN. The client does not replace that step with its own lint/test run, and does not accept a deliverable the evaluator has not passed. Once the verdict is PASS, the client matches the deliverable against the acceptance criteria written in Phase 1 and makes the final call.

### Prerequisites

- Implementation is complete — no half-done branches, no TODO placeholders
- Tests have been added for the change. Writing tests is not optional. Reporting "done" without tests is prohibited.

### DoD execution order (never skip or reorder)

1. Implementation is done
2. Tests are written for the change
3. Run every DoD command for the ecosystem
4. On any failure, fix the root cause and restart from step 3 (no resuming mid-way)
5. Report completion — no hedging, no extra questions

### DoD commands per ecosystem

The concrete command list lives in the ecosystem's skill. Do not duplicate it here — refer out:

| Ecosystem | Authoritative DoD source |
|-----------|--------------------------|
| Python | `coding-standards` skill |
| Terraform | `terraform` skill |
| GitHub Actions workflow | `gh-actions` skill |
| Other | Project's task runner definitions (lint / format / test / type-check targets) |

When no skill covers the ecosystem, the DoD is: **all lint / format / test / type-check commands defined by the project's task runner, executed against the entire project.**

### Scope

Whole project, not just changed files. Per-file or per-directory verification (e.g. `<runner> lint src/changed_file`) is not acceptable for DoD, because CI runs against the entire project and drift in unchanged files still breaks the merge.

### Definition of Done

- [ ] Tests added for the change
- [ ] Every DoD command passed with zero errors
- [ ] Verification covered the entire project (not scoped to changed files)
- [ ] No error remains — a single failure means the task is not done
- [ ] Evaluator returned PASS (for delegated work)
- [ ] The deliverable was matched against the Phase 1 acceptance criteria

## Prohibited Actions

- Writing production or test code in the main session for work that should be delegated
- Delegating a work order that carries no acceptance criteria
- Accepting a deliverable without an evaluator verdict
- Reading or editing files with `sed`, `cat`, `awk` — use the Read / Edit tools instead
- Bypassing the task runner by invoking linters, formatters, or type-checkers directly
- File-scoped or directory-scoped DoD verification (must be whole-project)
- Custom scripts written to simplify or work around standard project tooling
- Implementation patterns that deviate from the existing codebase without explicit justification

## Implementation Patterns

### Following conventions

- Read existing code before writing new code
- Refactor similar functionality consistently across the codebase
- Document custom patterns only when they are genuinely unavoidable

### Token efficiency

- Batch independent operations into parallel tool calls
- Compress context strategically between phases
- Avoid repetitive or redundant work
