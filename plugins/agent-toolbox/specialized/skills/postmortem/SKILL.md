---
name: postmortem
description: ユーザーからの指摘・修正を受けた後、同じ失敗を繰り返さないために根本原因を分析し、CLAUDE.md・rules・既存skillのうち該当する箇所を特定して修正する。「二度と起きないように」「同じミスをしないように」「rulesを直して」「なぜ気づけなかったか直して」等と言われたときに使う。
---

# Postmortem Skill

Use this skill when a correction exposes a judgment or instruction failure and the goal is to prevent recurrence. Convert the evidence into a durable behavior change in the correct source of truth. A postmortem is not an apology, an incident summary, or a token-, byte-, or line-count optimization exercise.

References supplement this contract with target-specific placement checks and reporting format. They are not a substitute for the generic requirements below, and the existence of a reference does not justify omitting a requirement needed to perform the work.

## When to Activate

- The user points out a mistake and then asks to prevent recurrence, such as "二度と起きないように", "同じミスをしないように", or "rulesを直して"
- The user explicitly invokes `/postmortem` immediately after a correction

Do not activate for a plain bug fix or a one-off technical workaround with no judgment failure. That belongs to the appropriate implementation or learning workflow.

## Required Contract

- The correction, existing instructions, current files, and current diff are evidence. Inspect them before editing; do not infer the source of truth from a familiar path, a file name, or the fact that a rule appears to exist.
- Separate confirmed facts, the expected judgment, the actual judgment or output, the root cause, and the recurrence condition. The root cause must explain why the available control failed or why no control existed, not merely restate the wrong output.
- Classify the cause as information that was available but not used, an existing rule that was not followed, or a genuinely new judgment for which no rule existed. Record the classification and the evidence for it.
- Derive a standing behavior that prevents the recurrence without generalizing a project-specific fact into a universal rule. Keep generic operating contracts in the applicable skill or shared rule and concrete domain, provider, repository, or tool facts in the appropriate reference or project context.
- Search all plausible existing sources before writing. Select the one source of truth that owns the behavior, extend a close existing rule, and do not add duplicates, contradictory wording, or a new skill for a single incident. Memory alone is not a durable governance change.
- Preserve every condition, exception, example, path, and acceptance requirement needed to execute the behavior. Reduce accidental prose and duplication, but never remove necessary elements to meet a character, byte, or line target.
- Complete the analysis, placement decision, and full content of the change before moving to verification. Do not declare the postmortem complete from a partial file, a reference link, or a passing check while required content is still missing.
- Do not create or request a custom validation script to compensate for missing tooling. After the content and placement are complete, use existing project checks when they exist and report checks that were not applicable or could not run.
- Keep the edit within the selected source-of-truth boundary, preserve unrelated user changes and existing comments, and match the target's language, structure, and conventions. Do not use destructive or lossy file operations.

When the selected artifact concerns planner output, preserve the planner-owned plan path, ADR ownership, and user-owned approval state. Never move a plan to a legacy path, change `Approval` from `[ ]` to `[x]`, or claim that a plan was approved. The target-specific checks and exact path rules are in [Target checks](references/target-checks.md).

## Process

1. Collect the correction, the expected behavior, the actual behavior, the affected artifacts, and the current repository or instruction-source state.
2. Reconstruct the root cause and recurrence condition using the required distinctions above. Identify the control that should have caught the failure.
3. Classify the lesson and choose one source of truth. Use the placement decision reference for the target-specific boundary.
4. Define the standing rule or checklist change, including its scope, trigger, owner, exceptions, and how its presence or behavior can be checked. Keep all required elements even when the resulting change is longer than the original.
5. Edit only the selected artifact with a targeted patch. For planner output, follow the target-specific checks in the target reference.
6. Once the content is complete, perform applicable existing checks and record the evidence, unrun checks, and remaining uncertainty in the report format.

## Completion Contract

Do not report completion until the report can identify the root cause, recurrence condition, classification, selected source of truth, reason for placement, concrete change, and verification status. The change must preserve the required elements and target conventions, avoid duplicate or contradictory rules, and leave planner approval and path ownership untouched. A local or static check does not prove an external, generated, CI, or deployed state; report those boundaries separately.

## References

- [Placement](references/placement.md) — classify the lesson and select the owning governance artifact without duplicating it.
- [Target checks](references/target-checks.md) — inspect planner plans and ADRs without changing their ownership or approval state.
- [Report](references/report.md) — record the decision log, concrete diff, evidence, and unresolved items.
