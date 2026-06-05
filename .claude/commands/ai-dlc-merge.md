---
description: Sequentially merge completed tracks from one place — rebase·verify·merge·cleanup the merge-awaiting queue onto main one by one (prevents tangling)
argument-hint: "[optional: track ID filter (e.g. t3,t5, comma-separated) — if empty, queues all merge-awaiting tracks]"
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(git status:*), Bash(git worktree list:*), Bash(git worktree remove:*), Bash(git log:*), Bash(git diff:*), Bash(git merge-base:*), Bash(git rev-list:*), Bash(git branch:*), Bash(git rebase:*), Bash(git merge:*), Bash(git -C:*), Bash(git pull --rebase:*), Bash(git checkout:*), Bash(git clean:*), Bash(git stash:*), Bash(git add:*), Bash(git commit:*), Bash(pytest:*), Bash(npm:*), Bash(make:*), Bash(cargo:*), Bash(go test:*), Bash(bun test:*)
---

# /ai-dlc-merge — Completed-track sequential merge orchestrator

When you develop several tracks **concurrently** in worktrees and each merges to main on its own,
they overlap and tangle (branches on a stale base, contention on shared files registry/audit,
cross-integration breakage — e.g. after merging t3, a leftover-reference crash patched by a t8
hotfix). This command runs **from a single place only (the main working tree)** and treats **all
merge-awaiting tracks as one queue, merging them in turn**. Because it is a single execution point,
shared-file contention serializes naturally without locks, and each track is **rebased again onto
the main that already reflects the previous merge**, so tangling surfaces and is resolved in place
before the merge.

Scope filter: $ARGUMENTS (if empty, all merge-awaiting tracks)

## Core premise: the main working tree need not be "clean", only "normal noise"

Running several tracks concurrently means the main working tree always has **active tracks' doc
changes** floating in it (uncommitted/untracked docs under `aidlc-docs/tracks/<id>/`, shared
registry edits, etc.). This is normal and is **not a target for a hard clean**. The merge does not
require wiping all this noise — instead it **classifies changes by owning track**, blocks only
unidentified (unregistered / non-active) changes, and **reflects** an active track's changes
**when that track is merged**. (The previous version was "abort if there are any uncommitted
changes", but in a concurrent-track environment that always tripped and was unusable — that rule is
replaced by the triage below.)

## Execution premises (blocking)
- **Run from the main working tree, as a single instance only.** Abort if inside a worktree or if
  `/ai-dlc-merge` is already running. (The concurrency guard *is* this premise — "runs from one
  place only" — no separate lock.)
- Uncommitted changes are **classified by the Stage 0a triage**. If there are no unidentified
  changes, proceed (no hard clean needed).
- Docs default to English. Application code goes in the workspace root; docs only under `aidlc-docs/`.
- **Append** user input/approvals to the global `aidlc-docs/audit.md` (never overwrite).

---

## Stage 0a — Working-tree triage (block only unidentified changes)

Before starting the merge, **classify by owner** the uncommitted state of the main working tree.

1. Collect all changed/untracked paths with `git status --porcelain`.
2. Map each path to an owner:
   - `aidlc-docs/tracks/<id>/**` → **owned by track `<id>`**.
   - Shared root files (`aidlc-docs/aidlc-state.md`, `aidlc-docs/audit.md`) → **shared**.
   - Everything else (workspace-root application code, anything outside `aidlc-docs/`, files
     outside a track directory) → **foreign (unidentified)**.
3. Decide against the registry (root `aidlc-docs/aidlc-state.md`):
   - **If there is any foreign path → abort.** Present the list and ask the user to handle it
     (commit/revert/attribute to a track). The merge orchestrator never swallows changes of
     unknown origin.
   - If an owning track is **not in the registry**, or is there but its **Status is not `active`**
     (merged·abandoned·paused etc.) → **abort** and report. ("Changes from a non-active track" is a
     working-tree-pollution signal.)
   - If the remaining changes are **only shared files + active-track docs** → **pass**. This noise
     is normal and is not wiped. Each active track's changes are reflected at the stage that merges
     it (Stages 1·4).
4. Append the triage result (including pass/block reason) as one line to `aidlc-docs/audit.md`.

> One-line summary: **noise from active tracks / shared files is OK; unidentified / non-active changes STOP.**

---

## Stage 0b — Build the merge queue + user confirmation (🛑 the sole approval gate)

1. **Collect candidates.** Read the `active` rows from the Track Registry in root
   `aidlc-docs/aidlc-state.md`, and for each track:
   - **Merge-ready signal** (priority): if `**Status**:` in `aidlc-docs/tracks/<id>/state.md` is
     `merge-awaiting`, it is explicitly awaiting merge. (No contention, since a track only marks its
     own file.)
   - **Heuristic fallback**: with no explicit signal — worktree exists + the feat branch has
     commits ahead of main (`git rev-list main..feat/<id>` ≥ 1) + Stage Progress in state.md is all
     `[x]` (especially Build & Test) → put it forward as a **candidate**, marked "inferred".
2. Gather **per-track readiness evidence**: number of commits ahead, base commit,
   `git merge-base feat/<id> main`, list of changed files, summary of state.md's verification section.
   **Also read the `## Merge Risk Notes` section in each candidate track's `state.md`**:
   - If it has info on shared files, API/signature changes, or known concurrent tracks, reflect it
     in building the queue.
   - This info is written by the track author and **complements** the automatic
     `git diff --name-only` overlap analysis (automatic analysis is file-level, but Risk Notes
     capture function/signature-level risk).
   - It is fine if Risk Notes are empty — automatic diff analysis alone suffices in most cases.
3. **Pre-gate (auto-exclude)** — drop these from the queue and report the reason:
   - 0 commits ahead, or no worktree → exclude as "nothing to merge".
   - (Apply any project-specific pre-gates here — e.g. exclude bases before a certain commit.)
4. **Cross-overlap analysis.** For each pair of queue candidates, compute the intersection of
   changed files (`git diff --name-only main...feat/<id>`). Surface overlapping track pairs and
   files as a table.
5. **Auto-decide order (dependency/overlap based).** Place independent tracks (0 file overlap with
   other candidates) first; place overlapping tracks later by base-age (older base first), pushing
   conflicts to the back of the queue to minimize them. Present the decided order and the
   **rationale** (why this order).
6. **Present the whole queue to the user for a single approval**:
   ```
   Merge queue (proposed order):
     1. t4  [clean]         feat/t4  ↑3  no overlap
     2. t2  [inferred]      feat/t2  ↑5  overlap: t5 (config/main.py)
     3. t5  [merge-awaiting] feat/t5 ↑7  overlap: t2 (config/main.py)
   Excluded: t1 (base before a structural cutoff — manual cherry-pick)
   Working-tree noise (not a merge target, left as-is): t6 docs (active), t8 docs (active)
   ```
   - Once the user approves, **proceed autonomously thereafter** (stop only on the stop conditions below).
   - Append the approval/selection to `aidlc-docs/audit.md`.
   - If there are no merge-awaiting tracks, end with "no tracks to merge".

---

## Stages 1..N — Per-track sequential merge loop (in the approved order)

For each track `T` (worktree `W`, branch `feat/T`) in order:

### 0) Tidy track-T-owned working-tree noise (just before merging)
- Track T's **authoritative docs are feat/T** (written by the single writer of worktree W). The
  uncommitted/untracked T-owned docs floating in the main working tree are **superseded** by what
  the feat/T merge brings in — this is the actual mechanism of "reflect that track when merging".
- So that `git merge feat/T` does not collide with untracked files, **wipe only the main-tree
  leftovers under the T-owned path (`aidlc-docs/tracks/T/`)** (e.g. tracked: `git checkout -- <path>`,
  untracked: `git clean -fd aidlc-docs/tracks/T/`). **Never touch another track's noise.**
- ⚠️ **Information-loss prevention (judgment)**: before wiping, check whether the main-tree
  leftovers contain anything that exists only there and not in feat/T (`git -C W show feat/T:<path>`
  comparison or diff). If feat/T is authoritative, wipe as-is; if the leftover side has missing
  content, **stop** → ask the user which side to adopt.

### 1) Rebase onto the latest main (the core of tangle prevention)
- `git -C W rebase main` — put feat/T onto the current main that **reflects the previous track's merge**.
- **On conflict**: analyze the conflicting files/hunks.
  - Resolve mechanical/obvious conflicts directly (import ordering, a same-direction fix to a
    rename/signature change the previous merge introduced, etc.) and `git -C W rebase --continue`.
  - For a **semantic conflict** (two tracks change the same logic with different intent), **stop**
    → present the conflict to the user and get a decision.
- **⚠️ Post-conflict-resolution cross-logic verification (MANDATORY, only when a conflict occurred):**
  for every file that had a conflict, **before** continuing the rebase:
  1. **Read the whole resolved file** and **list the identifiers** referenced by all new code the
     rebased track added (function calls, variable references, prop passing, import usage).
  2. **Confirm each identifier is actually defined/imported in the merged result file**:
     - function/variable: does a definition exist in the file?
     - import: is it included in the import statements?
     - prop: does it exist in the interface of the target component?
     - type/interface field: can the definition be found?
  3. **A pattern to watch especially** — if the conflict was in a file the previous track (the main
     side) refactored (rename, signature change, function split), confirm the rebased track's code
     was **adjusted to the refactored API**. E.g. main renamed `pinnedDate()`→`pinnedStart()`; if
     the rebased track's `isToday()` definition still references `pinnedDate()`, it must be
     rewritten on `pinnedStart()`.
  4. If any identifier fails the above, fix it in the file and re-check.
     **Never assume "git removed the conflict markers, so it's done".**
     git only merges both sides as text; it **does not verify logical consistency**.

### 2) Re-run verify (analyze cause and fix on failure)
- Actually **re-run** the track's verification in W (mandatory, since the rebase may have changed code):
  - Run the verification command specified in the track's `state.md` `## Verify` section, as-is.
  - E.g. `pytest -q`, `npm test`, `make check`, `cargo test`.
- **On failure**: analyze the cause. If it is **cross-integration breakage** caused by the previous
  merge (stale references, missed renames, etc.), fix it directly in W, commit a fixup, and re-run
  verify. If it is a failure needing **judgment**, like a regression in the track's own logic, stop
  → report to the user.

### 3) Merge into main
- Since the rebase put feat/T linearly on top of main, it merges without conflict.
- In the main working tree, `git merge --no-ff feat/T` (`--no-ff` by default for traceability; put
  the track ID/summary in the merge commit message). Since T-owned noise was tidied in Stage 1-0,
  it proceeds without untracked collisions.

### 4) Close out the docs + reflect into shared files (write shared files once, only at this point)
- In root `aidlc-docs/aidlc-state.md` Track Registry, change **only T's row** `active` → `merged`
  and record the merge sha in Branch/Updated. **Preserve other active tracks' rows/registrations**
  as-is (even if their uncommitted registry edits float in the working tree — those are owned by
  those tracks).
- In `aidlc-docs/tracks/T/state.md`, set `**Status**:` → `merged → main <sha> (date)`.
- Append a **one-line summary** to the global `aidlc-docs/audit.md` (existing
  `- YYYY-MM-DD — **Tn merged** …` format).
- **Close commit**: put the above changes in a single commit (`docs(Tn): close track — merged (<sha>) …`),
  but **limit the staging scope to T-related + shared files** — i.e. stage only `aidlc-docs/tracks/T/`
  and the shared `aidlc-docs/aidlc-state.md`/`aidlc-docs/audit.md` (explicit paths with `git add`).
  Keep other tracks' uncommitted noise out of this commit.

### 5) Cleanup (full cleanup)
- `git worktree remove W` (after confirming the tree is clean).
- `git branch -d feat/T` (`-d` succeeds since it is merged).
- Clean up leftover artifacts in `.claude/worktrees/T` (`__pycache__`, etc.).

### 6) On to the next track
- The next track's rebase goes onto the main that includes the T merge just landed → overlaps/tangles
  surface there and are resolved by the same procedure. Repeat until the queue is empty.

---

## Stop conditions (when to hand off to a human during autonomous progress)
- Stage 0a triage finds **foreign (unidentified) changes** or **non-active/unregistered-track changes**.
- In Stage 1-0, the main-tree leftovers contain content absent from feat/T, so an **authoritative
  judgment is needed**.
- A **semantic conflict** in the rebase (not mechanically resolvable).
- A verify failure judged to be **a track-logic regression rather than cross-integration breakage**.
- A track caught by a pre-gate (auto-excluded, but the user requests forcing — guide them).
- The worktree/branch state differs from expectations (e.g. uncommitted changes on feat/T, detached HEAD).

Each stop holds **only the current track**; the rest of the queue continues after the user's
decision (already-merged tracks are not rolled back).

## Final report
- Summarize, in a table, the merged tracks (+sha), held tracks (+reason), and excluded tracks (+reason).
- The cleaned-up worktrees/branches, updated registry rows, and the list of appended audit lines.
- Also state the **working-tree noise left as-is** (uncommitted docs of other still-active tracks) —
  make clear it is intended leftover.

## Operating rules
- **The working tree being "only normal noise" is OK.** Uncommitted changes of active tracks /
  shared files are not wiped. Block only unidentified / non-active changes (Stage 0a). The hard-clean
  requirement is retired.
- **Single writer.** Each track's state.md is written only by that track. The root registry/audit
  is written by this command only at merge time, serially (this is the only concurrent-write point,
  and it is safe because it is a single execution).
- **Reflect shared files after merge.** Only after merging T, change T's row to `merged` in
  `aidlc-docs/aidlc-state.md` and include it in the close commit. Preserve other active tracks'
  rows/noise, and limit the close commit's staging to T-related + shared files.
- `aidlc-docs/audit.md` is **append-only** (never overwrite the whole thing — it causes duplicates).
- Never roll back an already-merged track. On failure, stop **at the current track** and report.
- The new track status value `merge-awaiting` convention: after Build & Test completes, a track
  marks its own state.md Status as `merge-awaiting` to enroll in this queue (the registry stays
  `active` until the merge).
- **The queue criterion is the Status flag alone — not the presence of commits.** If additional work
  (implementation · code-review · critic fixes · re-review) lands on a `merge-awaiting` track, that
  track sets its own state.md Status back to `active` *before* editing and restores `merge-awaiting`
  only after Build & Test is re-green (authority: `common/concurrent-tracks.md`). Since this command
  builds the queue purely on Status, a track left `merge-awaiting` while work is in flight risks
  picking up a half-finished track.
