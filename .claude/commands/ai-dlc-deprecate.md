---
description: Dedicated track for deprecating a feature/API — safely cut it through impact analysis, agreement, migration, and call-site cleanup
argument-hint: "[feature/API/flag/module to deprecate — if empty, asks what to deprecate]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git worktree list:*), Bash(git status:*), Bash(git diff:*), Bash(pytest:*), Bash(npm:*), Bash(make:*), Bash(cargo:*), Bash(go test:*), Bash(bun test:*)
---

# /ai-dlc-deprecate — Feature Deprecation Track

A dedicated command for **intentionally removing (cutting)** an existing feature/API/option.
It splits out **T3 (intent change / feature cut)** from `/ai-dlc-refactor` as a standalone
procedure — the "delete/modify some features for simplification" case belongs here.

**Premise**: a cut is inherently a behavior change, so it **always requires user approval**.
Until approval, this command only **analyzes and proposes** — it does not remove any code.

Deprecation target: $ARGUMENTS
(If empty, first ask "what to deprecate, and why".)

## Procedure

Place each stage's output under `aidlc-docs/inception/deprecate/<name>/`, with an approval gate
at each stage.

### Stage 1 — Impact Analysis
Output: `1-impact.md`
- **What is being deprecated**: exact symbol/endpoint/flag/file.
- **Who uses it**: full sweep of call sites (grep for internal usage), whether it is an external
  contract (public API / CLI / schema / config key / storage format), and dependent tests.
- **Why deprecate**: the **concrete complexity cost** that maintaining backward-compat incurs.
- **What is lost**: behaviors / usage scenarios that existed only in this feature.

### Stage 2 — Deprecation Decision Gate (🛑 approval required)
- Summarize Stage 1 as a table, present it to the user, and **get a decision**:
  - **Remove now** / **Remove after grace period (via a deprecation-warning phase)** /
    **Remove after providing a replacement** / **Keep (cancel the cut)**.
- If it is an external contract, default conservatively to recommending "grace period /
  replacement", but follow the user if they choose "cut now". Record the decision in the output
  and in `aidlc-docs/audit.md`.

### Stage 3 — Migration Plan
Output: `2-migration.md`
- A staged removal order matching the approved approach (small units, keeping tests green at
  every step).
- If there is a replacement, the call-site migration mapping; if a grace period, the warning
  message / deadline; the docs / CHANGELOG entries to update.
- Specify how tests killed by the removal are handled (delete, or replace with replacement tests).

### Stage 4 — Implementation (AI-DLC Construction)
- After Stages 1–3 are approved, proceed via the AI-DLC code-generation flow.
- Remove/migrate **only the approved scope**. Do not touch any additional cut that was not agreed.
- Clean up call sites, tests, and docs in the same change so no broken references remain.

## Operating rules
- **Append** user input/approvals to `aidlc-docs/audit.md` (never overwrite). Docs default to English.
- Application code goes in the workspace root; docs only under `aidlc-docs/`.
- If the deprecation scope is large and entangled with a separate redesign, coordinate with
  `/ai-dlc-refactor` (this track focuses on the cut).
