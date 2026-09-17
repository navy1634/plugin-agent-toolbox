# Placement Decision

Use this reference after the generic root-cause analysis. It defines the target-specific boundary for turning a lesson into a durable rule. Choose one owning artifact; do not copy the same lesson into several targets merely because more than one target is related.

## Classification

| Lesson | Owning artifact | Use it when |
|---|---|---|
| A judgment process or verification habit that applies across projects | A shared or global rule source | The failure does not depend on one repository's domain or implementation, such as concluding from a surface pattern without tracing the condition's origin |
| Project-specific domain knowledge, design, or operating flow | The project's `CLAUDE.md`, `AGENTS.md`, or equivalent context | The failure came from misunderstanding how a function, setting, workflow, or service behaves in that project |
| A procedure or checklist that must be followed whenever a particular skill is used | The existing skill's `SKILL.md` | The missing step is part of a skill's reusable operation, such as a Terraform or security-review check. Extend the existing skill; do not create a new skill for one incident |
| A reusable technical solution, error fix, or workaround | The applicable learning skill or command | The failure is a new technical fact or solution rather than a judgment or governance failure |

## Placement Checks

1. Search global rules, project context, existing skills, agent instructions, and related references for an existing owner and equivalent wording.
2. Prefer the narrowest artifact that owns the behavior while keeping a genuinely cross-project judgment in a shared rule. Do not put a provider-specific fact in a generic skill merely because the workflow is shared.
3. If an existing artifact contains a close rule, extend or clarify that rule. Do not add a parallel rule with different wording or split one inseparable contract across files for line-count reasons.
4. Keep the generic operating contract in the skill body and place provider, repository, tool, or target-specific mechanics in references or project context. A reference must add concrete facts or procedures; it must not be a one-line indirection that removes the necessary contract from the skill.
5. Do not use a memory entry as the only destination. Memory can support investigation, but a recurrence-prevention rule must land in an artifact loaded by the relevant agent or workflow.

## Decision Record

Record the selected artifact, the nearest alternatives considered, the evidence that established the scope, and why the alternatives were not selected. If no existing artifact owns the lesson, state that a new artifact is necessary and why the existing locations cannot host it. A single incident does not by itself justify a new universal rule or a new skill.
