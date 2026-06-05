# Concurrent Multi-Track Development

## Why this rule exists

When you run multiple AI-DLC sessions against one repo, they collide on two things:

1. **Shared mutable files** — a status file, an audit log. Two sessions read-then-write and
   clobber each other.
2. **Stale-base merges** — each branch was cut from an old `main`; merging them independently
   tangles (one merge breaks another's assumptions, cross-integration regressions surface
   *after* the fact).

This rule removes both hazards with one idea + one orchestrator. It is the **AI-DLC-native
integration** of the [agent-tracks](https://github.com/inventor71/agent-tracks) philosophy —
the same "partition, don't lock" principle applied across every phase of the AI-DLC workflow.

## Core principle: partition, don't lock

> **Every state/audit file has exactly one writer.** No file locks.

File locks are the wrong tool for a git-backed doc edited by a human+agent loop (stale locks
when an agent dies, no clean cross-worktree "wait"). Instead, **eliminate shared mutable
state** by giving each track its own files. The only file multiple tracks ever touch is the
lightweight **Track Registry**, edited just twice per track (create + close) — rare enough to
resolve with `git pull --rebase`.

## What a "track" is

- A **track** = one feature/refactor/deprecate effort, developed in its own git worktree.
- Each track has a stable **Track ID**: `t1`, `t2`, … (next id = max existing + 1). Refactor
  and deprecate efforts are tracks too (e.g. `r1`, `d1`) and follow the same partition rules.
  Prefix by kind is encouraged: `f` feature, `r` refactor, `d` deprecate.
- A track owns exactly one branch and one worktree.

## File layout

```text
.aidlc-docs/
├── registry.md               # Track Registry ONLY (thin index). Rarely edited.
├── log.md                     # GLOBAL timeline. Append-only, written ONLY at merge-time.
├── codekb/                    # Shared codebase knowledge. Single writer: CI (bootstrap exception: first track).
│   ├── summary.md
│   ├── architecture.md
│   ├── integration-map.md
│   ├── domain-entities.md
│   ├── business-rules.md
│   ├── nfr-design.md
│   ├── infrastructure-design.md
│   └── codekb-state.md
└── tracks/
    ├── _TEMPLATE/             # copy this to start a track
    │   ├── state.md
    │   └── audit.md
    ├── t1/
    │   ├── state.md           # t1's full stage progress / extension config / scope.
    │   ├── audit.md           # t1's append-only audit log.
    │   ├── inception/         # t1's requirements/ plans/ user-stories/ application-design/
    │   └── construction/      # t1's plans/ {unit}/{functional-design,nfr-*,code}/ build-and-test/
    │                          #   ALL of the above: SINGLE WRITER = the t1 worktree session.
    └── …
```

- **Per-track docs all live under `tracks/<id>/`**: `state.md`, `audit.md`, and every phase
  artifact (requirements, plans, functional/NFR design, build-and-test) in `inception/`+
  `construction/` subdirs mirroring the global layout. Author them in the worktree so `main`
  stays clean (no stray uncommitted docs when `/ai-dlc-merge` starts).
- **Per-track `state.md`**: everything that used to go in a shared `aidlc-state.md` feature-track
  section — stage progress checkboxes, extension config, construction scope, design notes,
  and a `## Merge Risk Notes` section (filled when transitioning to `merge-awaiting`) that
  records shared files / API changes / known concurrent tracks to help `/ai-dlc-merge`
  resolve conflicts beyond what `git diff --name-only` can see.
- **Per-track `audit.md`**: every user input / approval / AI action for this track, append-only,
  ISO 8601 timestamps, raw user input (never summarized) — same format as before, just scoped.
- **Root `registry.md`**: the **Track Registry** (table below) only. Do NOT add per-track
  detail here — it lives in `tracks/<id>/state.md`. The only edits to this file are Registry
  rows (at track create / merge / close).
- **Root `log.md`**: a global cross-track timeline. A track appends to it **only at merge**
  (fold a one-line summary), never mid-flight — so there is no concurrent append race.

## Track Registry (in root registry.md)

A single table is the authority for which tracks exist and where they live:

```markdown
## Track Registry
| ID | Title | Status | Branch | Worktree | Base | Updated |
|----|-------|--------|--------|----------|------|---------|
| t1 | … | active | feat/t1 | .worktrees/t1 | <sha> | 2026-… |
```

- `Status` ∈ `active` / `merged` / `abandoned`, plus the transient `merge-awaiting` a track sets on
  **its own** `tracks/<id>/state.md` when Build & Test passes (the standard hand-off — see
  `construction/build-and-test.md` Step 8). `merge-awaiting` enqueues the track for `/ai-dlc-merge`;
  the registry row stays `active` until that command flips it to `merged` at actual merge time.
- **`merge-awaiting` is revertible — the Status flag, not the commit log, is the queue.** If more
  work lands on a `merge-awaiting` track (extra implementation, code-review/critic fixes, a
  re-review), flip its `tracks/<id>/state.md` Status back to `active` **before editing**, and
  restore `merge-awaiting` **only after** Build & Test is re-run green. A commit by itself never
  removes a track from the queue; `/ai-dlc-merge` selects purely on this flag, so leaving it
  `merge-awaiting` while work is in flight risks merging a half-finished track.
- A registry row is written at track **creation** and flipped at **merge/close**. These are the
  only two cross-track edits; serialize them with `git pull --rebase` before committing.

## MANDATORY worktree gate

**No application code may be generated outside a worktree.** Enforced as a hard, blocking gate.

Before Code Generation **Part 2** (actual coding), the track MUST be on its own worktree branch:

```bash
git worktree add .worktrees/<track> -b feat/<track>
```

If the session is on the default branch (`main`) with uncommitted code changes, **refuse to
generate code** and create/switch to the worktree first. Inception/design docs (markdown in
`.aidlc-docs/tracks/<id>/`) may be authored before the worktree exists; code may not.

`/ai-dlc-status` flags a violation: uncommitted code changes in the `main` working tree.

### Worktree bootstrap

A fresh worktree has no dependencies installed (gitignored build output). Before verification
can run, bootstrap the worktree with the project's package manager:

```bash
# Example — adapt to your project's toolchain:
( cd .worktrees/t1 && npm ci )       # Node
( cd .worktrees/t1 && uv sync )       # Python
( cd .worktrees/t1 && bun install )   # Bun
```

Link environment files as needed (e.g. `ln -sf /path/to/.env .worktrees/t1/.env`). See
[agent-tracks](https://github.com/inventor71/agent-tracks) for the `track-setup.sh`
reference implementation — a single script that creates the worktree and runs
project-specific bootstrap steps.

> **Registry row ⇒ per-track record (no exceptions, even for lean hotfixes).** If a change gets a
> row in the Track Registry, it MUST have a `.aidlc-docs/tracks/<id>/state.md` — at minimum a few
> lines (title, status, branch/merge commit, what changed, verification). Do NOT ship a track as
> "registry row + global audit one-liner only"; that leaves `tracks/` inconsistent with the registry.
> A tiny fix/follow-up may skip the full AI-DLC stages, but never the per-track `state.md` stub.

## Track lifecycle

1. **Create.** Pick next id from the **union of live sources** — `git worktree list`,
   `git branch --list 'feat/*'`, `ls .aidlc-docs/tracks/`, and the registry table — NOT from a
   cached session-start registry snapshot (other sessions claim IDs concurrently). Guard:
   `git worktree list | grep -q <id>` and `[ -d .aidlc-docs/tracks/<id> ]` before claiming.
   `mkdir .aidlc-docs/tracks/<id>`, copy `_TEMPLATE/{state.md,audit.md}` (only when the directory
   is absent — `cp -r` into an existing dir silently nests a stray `_TEMPLATE/` inside it). Add a
   registry row (`active`). Create the worktree **before** any code generation. (Even a lean hotfix
   track creates `tracks/<id>/state.md` here — see the note above.)
2. **Work.** All progress, audit, and phase docs go under `tracks/<id>/` only (state/audit +
   `inception/`+`construction/` artifacts). Never touch another track's files or the root files
   (except a registry row update if the title/base changes).
3. **Merge / close.** Prefer **`/ai-dlc-merge`** — a single-runner orchestrator that merges all
   `merge-awaiting` tracks sequentially (rebase each onto the just-updated main → verify → merge →
   close), so concurrent tracks can't tangle the shared registry/log or land on a stale base.
   It flips the registry row to `merged`, appends a **one-line** summary to the global root
   `log.md`, and removes the worktree (`git worktree remove …`) + branch. (Manual single-track
   merge remains possible, but run the orchestrator from one place to keep merges serialized.)

   > **Refactor/review tracks — pre-merge re-sweep.** A refactor or review track's branch-point
   > snapshot goes stale while it's open: other tracks merge concurrently and new code lands on
   > `main` that may carry the SAME issue the track targets. Before merging, re-diff `main` since
   > the track's base commit against the track's own heuristics (e.g. speed patterns, security
   > rules, code-quality checks) and either fold qualifying new hits into the track or log them
   > as a recorded followup. A track's `state.md` should note this as an explicit pre-merge step.

## What is still global / shared

- The **Track Registry** (root `registry.md`).
- The **global timeline** (root `log.md`) — merge-time appends only.
- The **CodeKB** (`.aidlc-docs/codekb/`) — shared codebase knowledge. Single writer is CI (refreshes on every push to `main`). First-track bootstrap during inception RE if CodeKB absent. All tracks read-only.
- The rule files under `.aidlc-rule-details/` and `CLAUDE.md` — changing process itself is a track too.

## Quick checklist for any agent starting work

- [ ] Am I in a worktree for my track? If coding and on `main` → stop, create worktree.
- [ ] Does my track have `.aidlc-docs/tracks/<id>/{state.md,audit.md}`? If not → create from template + register.
- [ ] Am I about to edit root `registry.md`/`log.md` mid-flight? → Don't. Use my track files.
- [ ] Am I about to edit `codekb/` from a track? → Don't. Only CI writes it.
- [ ] Am I writing a phase doc to top-level `inception/`/`construction/`? → Don't. Put it under `tracks/<id>/`.
