# Decisions

One file per decision, named `NNN-short-title.md` (for example `001-default-output-mode.md`). Decisions come from the learning modules in `docs/learning/README.md`, or from any later point where the plan needs the owner's call. A decision that changes the plan is applied to the OpenSpec change before any code that depends on it.

## Template

```markdown
# NNN. <Decision title>

- **Date:** YYYY-MM-DD
- **Module:** <learning module number and name, if any>
- **Status:** accepted | superseded by NNN

## Question
<The choice, in plain language.>

## Options
- **<Option A>:** <what it means and its trade-off>
- **<Option B>:** <what it means and its trade-off>

## Recommendation
<What Claude recommended, and why.>

## Decision
<What the owner chose, in their words.>

## Consequences
<What changes in the specs, design or tasks, with links.>
```
