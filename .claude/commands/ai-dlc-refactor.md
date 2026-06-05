---
description: AI-DLC refactoring/redesign track — drive behavior-preserving redesign safely through tier gates
argument-hint: "[which module/file/concern to redesign — if empty, asks first]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git worktree list:*), Bash(git worktree add:*), Bash(git status:*), Bash(git diff:*), Bash(git branch:*), Bash(pytest:*), Bash(npm:*), Bash(make:*), Bash(cargo:*), Bash(go test:*), Bash(bun test:*)
---

# /ai-dlc-refactor — AI-DLC Refactoring Track

A command for **redesigning an existing implementation into a more efficient, maintainable
structure**. It reinforces the "behavior-preserving redesign" and "intent-change / feature-cut
approval gate" that the main AI-DLC workflow (feature-delta-centric) does not cover.

Target (scope): $ARGUMENTS
(If empty, first ask the user "which module/file/concern to redesign".)

---

## 0. Operating principles

- This command **reinforces, not replaces, AI-DLC**. It follows this ruleset's audit logging,
  approval gates, and content-validation principles as-is.
- Place doc outputs only under **`aidlc-docs/inception/refactor/<name>/`**. Application code goes
  in the workspace root (never inside `aidlc-docs/`).
- Record all user input/approvals as **append** to `aidlc-docs/audit.md` (never overwrite).
- If a question file is needed, write it in **English** (project default).
- After design approval, T1/T2 implementation **proceeds autonomously** (autonomy-in-construction).
  Only T3 stops.

---

## 1. Core concept: Change Tier

Every change item is classified into one of the 3 tiers below and recorded in `2-tier-ledger.md`.
**Do not touch code without classifying it.**

| Tier | Definition | Handling |
|------|------------|----------|
| **T1 — Behavior-preserving** | Externally observable behavior / interface / output unchanged. Internal structure only. | **Proceed autonomously** after design approval |
| **T2 — Safe extension** | Preserves existing behavior + adds a new path/option (superset). | **Proceed autonomously + report afterward** |
| **T3 — Intent change / feature cut** | Feature removal, semantic change, dropping backward-compat, shrinking a contract (signature/schema/output). | **🛑 Stop. No code change before user approval** |

When in doubt, **always raise to the higher tier** (conservative). Remember that "delete/modify
some features for simplification" is almost always **T3**.

### T3 gate behavior (propose only, then stop)
- A T3 candidate is recorded in the ledger **only as a "cut/change candidate"; never reflected
  in code.**
- For each T3 item, also record:
  - what you intend to cut/change
  - **the reason** (why maintaining backward-compat grows complexity — concrete cost)
  - what is gained / lost by the cut
  - affected call sites / users / tests
- If there is any T3 item, **present them all to the user and stop** before proceeding to Stage 3
  (redesign). Do not move on until the user decides "approve cut / keep / hold" per item.

---

## 2. Stages

Each stage produces an output and proceeds **after user approval** (2-option: request changes / continue).

### Stage 1 — Baseline + characterization tests (pin current behavior)
Output: `aidlc-docs/inception/refactor/<name>/1-baseline.md` + characterization test code
- Summary of the target scope's current structure (key files / entry points / dependencies).
- **List of observable behaviors to preserve** (external contract): public function signatures,
  CLI/output formats, file/DB schemas, side effects, error behavior.
- Current test coverage status (if any, which paths it covers).
- **Characterization tests first**: before starting the redesign, **write/confirm** tests that
  pin the behavior to preserve. If existing tests already cover those paths, use them; where
  there are gaps, add new tests that capture current behavior as-is.
  - These tests record "this is how it is now", not "this is correct" — a safety net to keep
    **before/after green** during refactoring.
  - Keep these tests green throughout Stage 4 implementation. A red is a signal that the change
    is **T3 (behavior change)**, not T1 — stop and raise it to the ledger.

### Stage 2 — Tier Ledger (classify + discuss cuts)
Output: `aidlc-docs/inception/refactor/<name>/2-tier-ledger.md` (template in §3)
- Break every planned change into items and classify as T1/T2/T3.
- Map **which Stage 1 characterization test protects each T1/T2 item**. For an item with no
  protecting test, go back to Stage 1 and add the test first (characterization-tests-first
  principle).
- If there are T3 items → **stop here**, present the cut agenda to the user (multiple-choice/table).
  Reflect the user's decision in the ledger, record it in audit.md, then go to Stage 3.

### Stage 3 — Redesign (target structure + migration)
Output: `aidlc-docs/inception/refactor/<name>/3-redesign.md`
- Target structure and an **equivalence argument** (why each T1 item preserves behavior).
- New behavior spec for approved T3 changes.
- Staged migration order (small units, keeping tests green after each step).

### Stage 4 — Implementation (AI-DLC Construction)
- After Stages 1–3 are approved, proceed via the AI-DLC code-generation flow.
- T1/T2: autonomous implementation + confirm tests pass at each step. For T2, report afterward
  what was added.
- T3: implement **only approved items**. Never touch an unapproved cut.
- Put code summary docs in `aidlc-docs/construction/<unit>/code/`.

---

## 3. `2-tier-ledger.md` template

```markdown
# Tier Ledger — <name>

Scope: <target module/file>
Date: <YYYY-MM-DD>

## T1 — Behavior-preserving (autonomous)
| # | Change item | Behavior preserved | Verification method | Rationale |
|---|-------------|--------------------|---------------------|-----------|
| 1 | (e.g.) split functions in module X | public API unchanged | existing tests suffice | path covered |
| 2 | (e.g.) remove duplication in Y | output unchanged | characterization test needed | test gap |

## T2 — Safe extension (autonomous + report afterward)
| # | Added item | Impact on existing behavior | Verification method |
|---|------------|-----------------------------|---------------------|
| 1 |            | none (superset)             |                     |

## T3 — Intent change / feature cut (🛑 approval required)
| # | Cut/change | Reason (complexity cost) | Gained | Lost | Scope of impact | User decision |
|---|------------|--------------------------|--------|------|-----------------|---------------|
| 1 |            |                          |        |      |                 | approve/keep/hold |

## Stop points
- [ ] T3 items presented to user
- [ ] User decision reflected + recorded in audit.md
```

---

## 4. Start procedure

1. Confirm the scope ($ARGUMENTS) — if none, ask the user.
2. Append this invocation to `aidlc-docs/audit.md`.
3. Start from Stage 1 (Baseline). After producing each stage's output, wait for approval.
4. On finding a T3, stop immediately and discuss with the user.
