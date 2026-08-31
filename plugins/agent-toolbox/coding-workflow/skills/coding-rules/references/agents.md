# Agent Orchestration

## Delegation Model (CRITICAL)

Implementation work is a client/contractor relationship, run like a small team the main session leads. The main session is the **client / lead**; subagents are the **contractors / specialists**. What matters is not the metaphor but the boundary: every responsibility below has exactly one owner, and no role may take over another's.

### Role boundaries

| Responsibility | Owner | Everyone else |
|----------------|-------|---------------|
| Interpreting what the user wants | client | May ask, may not decide |
| Requirements (what to build, why) | client | Implements them as written; never reinterprets |
| Acceptance criteria | client | Verifies against them; never edits, relaxes, or adds to them |
| Design and implementation plan | **planner** | Client states constraints, does not draft the design itself |
| Design decisions during implementation | **generator**, within the plan | Escalates instead of deciding beyond it |
| Production code | **generator** | Client does not write it (minor-change exception below) |
| Test code | **generator** | Client does not write it (minor-change exception below) |
| DoD command execution | **generator**, then **evaluator** re-runs it as the gate | Client does not substitute its own run |
| Quality / security / acceptance verification | **evaluator** | Client does not self-certify quality |
| Final accept-or-reject | client | Evaluator supplies a verdict; it is input, not the decision |
| Talking to the user (approval, questions) | client | Subagents report to the client, never to the user |

When a responsibility is unclear, it belongs to the client — the client then either owns it or delegates it explicitly. What no role may do is quietly assume it.

### Main session (client)

Owns:

- Turning the user's request into unambiguous requirements
- Defining acceptance criteria in a verifiable form, before delegating
- Choosing the contractor and writing the work order
- Receiving the deliverable and judging it against the acceptance criteria
- Delegating quality assurance to the **evaluator** and acting on its verdict
- All negotiation with the user (approval, branching questions)

Must NOT:

- Write production code (except the minor changes carved out below)
- Write test code — tests belong to the contractor's TDD cycle (except the mechanical test-only fixes carved out below)
- Re-decide a design the contractor already made; send it back instead
- Substitute its own lint/test run for the evaluator's quality gate
- Report completion without checking the deliverable against the acceptance criteria

### Subagent (contractor)

Owns:

- Implementing strictly within the received work order and acceptance criteria
- Running the full TDD cycle (RED → GREEN → REFACTOR) without skipping steps
- Reporting back what was built, what passed, and what remains

Must NOT:

- Reinterpret, extend, or narrow the requirements it was given
- Touch files outside the stated scope
- Rewrite, relax, or drop acceptance criteria — they are the client's, not the contractor's
- Fill ambiguity with a guess. Stop and return the question to the client
- Skip tests, or write the implementation before the failing test

### Minor-change exception

The client may edit directly only when the change is confined to a single file, needs no test, and does not alter behavior — typos, comments, config values, documentation. Everything else (new features, bug fixes, refactoring, anything spanning multiple files) is delegated. When it is unclear which side of the line a change falls on, delegate.

The same exception covers a mechanical test-only fix: an existing test's expectation is stale against a settled implementation, the root cause is already established, and the correction touches nothing but that expectation (its value, the test's name, its comments). No behavior is being designed, so there is no TDD cycle to hand over — running the DoD and the evaluator gate is enough. Delegate instead the moment any of these holds: it is still open whether the test or the implementation is wrong, a test must be added / removed / skipped / xfailed, or the fix reaches production code at all. A test rewritten so a failure stops appearing is never a minor change, however few lines it takes.

### Work order contents (mandatory)

A subagent cannot see the parent conversation, so every delegation must carry:

1. Requirements — what to build and why
2. Acceptance criteria — verifiable conditions for completion
3. Target files and the boundary of what may be touched
4. Existing patterns to follow (file paths to read first)
5. Prohibitions — what the contractor must not do
6. Report format

A work order missing acceptance criteria is not a work order. Do not delegate until they exist.

### Quality assurance role (evaluator)

Quality is a separate seat from implementation. The one who wrote the code never certifies it, and the client never self-certifies either — the deliverable goes to the **evaluator**.

Its boundary:

- **Owns** — verifying the deliverable against the acceptance criteria, re-running the DoD gate, reviewing the diff for security / correctness / quality / performance / test quality, and issuing PASS / REVISE / REDESIGN
- **Reads only** — it holds no write tools by design. It reports defects; it does not fix them. A verdict that says "fixed it while reviewing" means the seat was violated
- **Does not touch the acceptance criteria** — if a criterion is untestable or contradicts the plan, it says so in the verdict and returns it to the client, rather than substituting a criterion it prefers
- **Does not redesign** — design objections go back as REDESIGN, addressed to the planner
- **Reports to the client only** — never directly to the user

Verification order is fixed, and it stops at the first failure:

1. Acceptance criteria — is each criterion demonstrably met, with evidence (test name, command output)? Unmet or unverifiable criteria are REVISE
2. DoD gate — every command for the ecosystem, whole project. Any failure is REVISE
3. Quality dimensions — security, correctness, quality, performance, test quality

The client then matches the verdict against the acceptance criteria and issues the final judgment. An evaluator PASS is input to that judgment, not the judgment itself.

## Core Agents (3-role pipeline)

Located in `~/.claude/agents/`:

| Agent | Model | Role | Tools |
|-------|-------|------|-------|
| planner | opus | Design decisions and implementation planning | Read, Edit, Write, Grep, Glob |
| generator | sonnet | TDD implementation, build error resolution | Read, Write, Edit, Bash, Grep, Glob |
| evaluator | sonnet | Quality, security, performance, acceptance verification | Read, Grep, Glob, Bash |

The tool sets enforce the role boundaries — they are not a convenience list, and widening one to unblock a task dissolves the boundary it protects:

- **planner** writes and edits one thing only — the plan file it owns, including revising it after Evaluator feedback. Implementation files are outside its role even though the tool would allow it. It has no `Bash`: planning reads code, it does not run it
- **generator** is the only role that edits implementation and test code
- **evaluator** holds no write tool at all, so it structurally cannot fix what it reviews. That is what makes its verdict independent
- If a role appears to need a tool it lacks, that is a signal the work belongs to a different role. Delegate it there instead of widening the tool set

## Utility Agents

| Agent | Model | Role | When to Use | Tools |
|-------|-------|------|-------------|-------|
| e2e-runner | haiku | E2E testing | Critical user flows | Read, Write, Edit, Bash, Grep, Glob |
| refactor-cleaner | haiku | Dead code removal | Code maintenance | Read, Write, Edit, Bash, Grep, Glob |
| doc-updater | haiku | Documentation updates | Architecture docs | Read, Write, Edit, Bash, Grep, Glob |

These run on `haiku` because each executes a narrow, well-specified job rather than deciding anything — see `performance.md` for the model selection rationale. They are contractors like any other: they receive a work order and report back, and their output goes through the evaluator when it changes code.

## Pipeline Flow

```
[User Input] → [Planner] → [Generator] ⇄ [Evaluator] → [Deliverable]
                  ↑                           |
                  └──── feedback (REDESIGN) ──┘
```

Controlled by `/orchestrate` command. The main loop invokes each agent sequentially. Every box in the diagram except `[User Input]` and `[Deliverable]` is contractor work — the main loop routes, supplies context, and judges the result, but does not do the work inside any box itself.

## Agent Constraints

Subagent limitations by design:

- Cannot see the parent's conversation context. All information must be passed via prompt
- Cannot invoke other subagents (no nesting)
- Return value is summary text only. Use format instructions in prompt for structure
- Large data exchange should go through files

## Invocation Rules

### Automatic (no user prompt needed)

| Trigger | Action |
|---------|--------|
| Any implementation beyond the minor-change exception | Run pipeline via `/orchestrate` |
| A minor change the client made directly | Invoke **evaluator** standalone |

### Manual (user requests)

| Scenario | Action |
|----------|--------|
| E2E tests needed | **e2e-runner** |
| Dead code cleanup | **refactor-cleaner** |
| Documentation updates | **doc-updater** |

## Feedback Loop Rules

- Generator ⇄ Evaluator iteration limit: **3 rounds**
- Evaluator verdict is one of: PASS / REVISE / REDESIGN
- REVISE: Send back to Generator with specific fix instructions
- REDESIGN: Send back to Planner for design revision. Max 1 REDESIGN; second triggers user escalation
- If no PASS after 3 iterations, report remaining issues to user for decision
