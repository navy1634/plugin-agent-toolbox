# Target Checks

Read the planner section when the selected artifact is a plan or ADR. These checks describe concrete ownership mechanics; the generic postmortem contract remains in `SKILL.md`.

## Planner Plans and ADRs

Planner-owned output uses:

```text
~/.agents/plan/<repository-slug>/<task-slug>/plan.md
```

Derive `<repository-slug>` from the final directory name of the absolute path returned by `git rev-parse --path-format=absolute --git-common-dir`, with `.git` removed. `<task-slug>` begins with the work-start date in `YYYYMMDD-...` form. Do not use the worktree directory name or another legacy or task-specific path.

The plan starts with an unchecked `Approval` entry. Only the user may explicitly change `[ ]` to `[x]` after reviewing the full plan. Neither postmortem nor any other agent may change the checkbox, infer approval from an implementation request such as "Implement the plan", or claim approval in a report.

Planner owns design decisions and planner-route ADRs. A required ADR belongs in the same task directory and records background, considered options, rejection reasons, decision, rationale, impact, and unresolved items or review conditions, in that order where applicable. A postmortem must return a newly exposed design decision to the existing planner rather than creating or editing the ADR itself unless it is explicitly operating as planner.

## Target Conventions

Match the existing target rather than introducing a new language or layout convention:

- Global rules use an English body and the existing heading and bullet style.
- Project `CLAUDE.md` and `AGENTS.md` use Japanese keigo and natural phrasing; avoid noun-only bullet chains, colon separators outside a decision-log block, and repeated subjectless passive sentences.
- An existing skill keeps Japanese description frontmatter and an English body when that is its established convention.
