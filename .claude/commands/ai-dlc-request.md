---
description: Start a new development request in the AI-DLC adaptive workflow — from intent analysis through staged approval gates
argument-hint: "[one line on what to build/change — if empty, uses the request from the prior conversation]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git worktree add:*), Bash(git worktree list:*), Bash(git worktree remove:*), Bash(git status:*), Bash(git branch:*), Bash(git rev-parse:*), Bash(git pull --rebase:*), Bash(mkdir:*)
---

# /ai-dlc-request — AI-DLC Workflow Entry Point

Starts a new software development request in the **AI-DLC adaptive workflow**.
This is the "front door" for ordinary feature additions/changes.
(Use `/ai-dlc-refactor` for behavior-preserving redesign, `/ai-dlc-deprecate` for feature deprecation.)

Request: $ARGUMENTS
(If empty, target the request from the prior conversation context.)

## Procedure

Follow the ruleset's INCEPTION → CONSTRUCTION flow as-is. In summary:

1. **Load the ruleset.** From `.aidlc-rule-details/`, load:
   - `common/process-overview.md`, `common/session-continuity.md`,
     `common/content-validation.md`, `common/question-format-guide.md`.
   - Lightly load only the `*.opt-in.md` files under `extensions/` (load the full rules after opt-in).

2. **Show the Welcome message once.** Load `common/welcome-message.md` and show it only the first time.

2.5. **Create a track (concurrent multi-track).** Load and follow `common/concurrent-tracks.md`.
   - **Registry bootstrap.** If `aidlc-docs/aidlc-state.md` does not exist, create it and write the
     `## Track Registry` table header (`| ID | Title | Status | Branch | Worktree | Base | Started | Updated |`).
     If it exists, read the existing registry.
   - Assign a new **Track ID** (max existing + 1, e.g. `t9`).
   - Create the directory with `mkdir -p aidlc-docs/tracks/<id>`, then write `state.md` and `audit.md`
     directly from the templates below (not a copy — no `_TEMPLATE/` directory dependency):

     `state.md`:
     ```markdown
     # Track <id> — <title>
     
     > Per-track state. **Single writer = this track's worktree session.**
     > Never edit another track's state.
     
     ## Track Info
     - **ID**: <id>
     - **Title**: <one line from request>
     - **Kind**: feature | fix | refactor | deprecate
     - **Status**: active
     - **Branch**: feat/<id>
     - **Worktree**: .claude/worktrees/<id>
     - **Base commit**: <git rev-parse --short HEAD>
     - **Started**: <ISO 8601>
     
     ## Scope
     <what this track will build/change — one paragraph>
     
     ## Verify
     <exact command that proves this track works, e.g. `pytest -q`, `npm test`, `make check`>
     
     ## Merge Risk Notes
     > Filled when flipping to `merge-awaiting`. /ai-dlc-merge reads this.
     
     - **Shared files (watch)**: <files likely to overlap another active track>
     - **API / signature changes**: <renames, deletes, splits>
     - **Known concurrent edits**: <other track ids touching the same files>
     
     ## Progress
     - [ ] <step>
     - [ ] <step>
     ```

     `audit.md`:
     ```markdown
     # Track <id> — Audit Log
     
     > Per-track, **append-only**, single writer.

     ```
   - Add a row (`active`) to the **Track Registry** table in root `aidlc-docs/aidlc-state.md`.
   - **Worktree gate**: before actual code generation (Code Gen Part 2), you MUST run
     `git worktree add .claude/worktrees/<track> -b feat/<track>`. No code changes on `main`.
     (Design/doc stages may proceed even before the worktree exists.)

3. **Record the request.** Append the user's **verbatim** text to **that track's**
   `aidlc-docs/tracks/<id>/audit.md` (not the root audit.md; never overwrite).

4. **Workspace Detection (always).** Run `inception/workspace-detection.md`.
   - If the Track Registry in root `aidlc-docs/aidlc-state.md` has an **active track**, this may be
     a resume rather than a new request → say so and **suggest `/ai-dlc-resume`**.
   - Determine brownfield/greenfield by whether code exists.

5. **Reverse Engineering (brownfield + only when no artifacts exist).** `inception/reverse-engineering.md`.

6. **Requirements Analysis (always, adaptive depth).** `inception/requirements-analysis.md`.
   - If ambiguous, create **a question file in English** (`question-format-guide.md` format, A/B/C/D + Other).
   - Do not move to the next stage before passing the answer gate.
   - Present extension opt-in questions here too.

7. **Subsequent conditional stages** (User Stories / Workflow Planning / Application Design /
   Units Generation → Construction's Functional/NFR/Infra Design → Code Generation →
   Build & Test), run adaptively to the request's complexity. Each stage ends with a
   **2-option approval gate** (request changes / continue), and the user's response is recorded
   in audit.md.

## Operating rules

- **Append** all user input/approvals to **that track's** `aidlc-docs/tracks/<id>/audit.md`
  (no summarizing, ISO 8601). The root `aidlc-docs/audit.md` gets only a **one-line summary at merge time**.
- Question/plan/design docs default to **English**.
- Application code is generated only **inside the track worktree** (no code changes in the root
  `main` tree). Docs only under `aidlc-docs/`.
- After design approval, Construction (code+tests) **proceeds autonomously**, stopping only when
  genuine human judgment is needed.
- Reflect progress **immediately** in **the track's** `aidlc-docs/tracks/<id>/state.md` and each
  plan checkbox (root `aidlc-docs/aidlc-state.md` holds only Track Registry rows, updated only at
  create/close).
