---
description: Resume AI-DLC work from its stopping point, based on aidlc-state.md
argument-hint: "[optional: a specific unit/stage name — if empty, the next step the state points to]"
allowed-tools: Read, Glob, Grep, Bash(git worktree list:*), Bash(git worktree add:*), Bash(git status:*), Bash(git rev-parse:*)
---

# /ai-dlc-resume — Resume from the stopping point

Continues the AI-DLC work in progress from a previous session, **based on the state file**.

Resume target: $ARGUMENTS
(If empty — if a track argument is given, that track; otherwise the single `active` track in the
Registry. If multiple are active, first ask which track to resume.)

## Procedure

1. **Load session-continuity rules.** From `.aidlc-rule-details/`, load
   `common/session-continuity.md`, `common/process-overview.md`, `common/concurrent-tracks.md`.

2. **Select the track + assess state.** Read the following and reconstruct the current position:
   - The **Track Registry** in root `aidlc-docs/aidlc-state.md` — confirm the track (`<id>`) to
     resume, its branch and worktree.
   - **That track's** `aidlc-docs/tracks/<id>/state.md` — done/skipped/in-progress stages,
     Extension Configuration.
   - The **recent entries** in **that track's** `aidlc-docs/tracks/<id>/audit.md` — last user
     input / approval / next action. (The root `audit.md` is for merge summaries, so read
     progress context from the track audit.)
   - The plan-file checkboxes of the in-progress stage (`aidlc-docs/construction/<unit>/...`,
     `aidlc-docs/inception/.../plans/...`) — up to which step is `[x]`.
   - **Worktree check**: via `git worktree list`, whether you are in that track's worktree. If
     it is a coding stage but there is no worktree, advise creating it first (code only inside a
     worktree).
   - If it is `/ai-dlc-refactor` work, the stop points in
     `aidlc-docs/inception/refactor/<name>/2-tier-ledger.md` (especially unresolved T3 items).

3. **Present a resume-point summary.** To the user in 2–4 lines:
   - which phase/stage/unit you are in now,
   - the last completed item and **what to do next**,
   - if there is a gate awaiting user input (unanswered question / unapproved stage), state it.

4. **Check the gate.** If the next stage is **awaiting approval** or has an **unanswered question**,
   do not auto-proceed — re-present that question/approval and stop. Otherwise continue per the workflow.

5. **Record.** Append this resume invocation and the user's response to **that track's**
   `aidlc-docs/tracks/<id>/audit.md`.

## Notes

- If the state is contradictory (e.g. state says done but plan checkboxes are incomplete) or
  ambiguous, **do not guess** — confirm with the user what to do next.
- If there is no `active` track in the Registry, there is nothing to resume — advise starting
  fresh with `/ai-dlc-request`.
