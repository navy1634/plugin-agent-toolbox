# Postmortem Report

Use this format for the decision log and completion report. The report must describe evidence and scope, not merely say that a rule was added.

## Decision Log

Record these fields before or alongside the edit:

```markdown
## Postmortem Decision

- Trigger: [correction or recurrence concern]
- Confirmed facts: [what the files, instructions, commands, or user said]
- Expected judgment: [what should have happened]
- Actual judgment or output: [what happened]
- Root cause: [why the control failed or why no control existed]
- Recurrence condition: [the future situation that could repeat it]
- Cause classification: [available information unused / existing rule not followed / no existing rule]
- Selected source of truth: [exact artifact path and section]
- Alternatives rejected: [nearby artifacts and why they do not own this behavior]
- Decision: [standing rule or checklist change]
- Scope and owner: [when it applies and who follows it]
- Unresolved items or review condition: [remaining uncertainty or when to revisit]
```

State the options and rejection reasons before presenting the selected decision when the placement or prevention mechanism is non-trivial. Do not turn an incident narrative into a universal rule without evidence that its scope is cross-project.

## Completion Report

Report the following after the content and placement are complete:

```markdown
## Postmortem Result

- Changed files: [exact paths]
- Change summary: [what standing behavior or checklist was added or clarified]
- Preserved elements: [important existing conditions, examples, paths, and exceptions retained]
- Verification: [command or inspection, target, environment, and result]
- Not run: [check and concrete reason, or `none`]
- External or generated state not proven: [CI, deployed, network, or rendering boundary]
- Remaining work: [user action, unresolved issue, or `none`]
```

For an artifact with a separate source and rendered target, report both paths and the target-level diff status. For planner output, report the plan path and that `Approval` and ADR ownership were left unchanged. Do not report a local static check as proof of an external or generated state.
