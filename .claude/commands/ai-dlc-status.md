---
description: Current AI-DLC progress dashboard — phase/stage, pending approvals, skipped items, ledger (read-only)
argument-hint: "[optional: scope to a track ID (e.g. t9) — if empty, all active tracks]"
allowed-tools: Read, Glob, Grep, Bash(git status:*), Bash(git diff --stat:*), Bash(git worktree list:*)
---

# /ai-dlc-status — Progress dashboard (read-only)

Summarizes the current AI-DLC work state at a glance. **It modifies no files.**

Scope: $ARGUMENTS (if empty, all `active` tracks)

## Items to collect

0. **Track Registry.** From the `## Track Registry` table in root `aidlc-docs/aidlc-state.md`, read
   the track list (id/title/status/branch/worktree). If a scope argument is given, only that track;
   otherwise collect the items below per track for all `active` tracks.

1. **Track state.** From **each track's** `aidlc-docs/tracks/<id>/state.md`:
   - current phase (INCEPTION / CONSTRUCTION / OPERATIONS).
   - per-stage list of done `[x]` / skipped / in-progress.
   - enabled/disabled in Extension Configuration.

2. **Next action & gates.** From the recent entries of **each track's** `aidlc-docs/tracks/<id>/audit.md`:
   - the last completed step and what to do next.
   - whether a gate is **awaiting approval**.
   - whether there is an unanswered question file (detect empty `[Answer]:` tags in
     `aidlc-docs/inception/**/...questions.md`, `**/*-questions.md`).

3. **Plan progress.** Count the plan checkboxes of the in-progress stage as a `done/total` ratio.
   - `aidlc-docs/construction/<unit>/code/...`, `.../plans/...`.

4. **Refactor/Deprecate ledger.** If present, show the T1/T2/T3 counts from
   `aidlc-docs/inception/refactor/<name>/2-tier-ledger.md` and separately the
   **unresolved T3 items** (awaiting user decision).

5. **Working tree & worktree violations.** Via `git worktree list`, confirm each track's worktree
   exists; via `git status` (summary) and `git diff --stat`, the size of uncommitted changes.
   **Violation detection**:
   - If the `main` working tree has uncommitted **code** changes (worktree-gate violation), flag ⚠️.

## Output format

```
# AI-DLC Status — <project>

Tracks (registry): <id:status …>
⚠️ worktree violation: <uncommitted code on main — omit if none>

── <id> · <title> ── [<branch> @ <worktree>]
Phase: <…>   |   Stage: <…>   |   Unit: <…>
Progress:
  ✅ Done: <stage list>
  ⏳ In progress: <stage> (plan N/M)
  ⏭️  Skipped: <stage list + reason>
🚦 Pending gates: <awaiting approval / unanswered question — "none" if none>
🧩 Extensions: <enabled / disabled>
🔧 Refactor/Deprecate ledger (if any): T1 n · T2 n · T3 n  (unresolved T3: <list>)
➡️  Next action: <one line>

── <next active track> ──
…

📦 Working tree (overall): <number of changed files, +/- lines>
```

- Print one block per `active` track (only that track if a scope argument is given).
- If a given artifact does not exist, mark that line "N/A".
- If there is no `active` track in the Registry, advise "No AI-DLC work in progress — start with `/ai-dlc-request`".
