# aidlc-workflows-concurrent

**AI-DLC workflow rules reworked around the concurrent-track philosophy.** The standard
[AWS AI-DLC](https://github.com/awslabs/aidlc-workflows) adaptive workflow (Inception →
Construction → Operations), with every phase made track-aware — so you can run multiple
AI-DLC sessions in parallel without them stepping on each other.

> This is the rule-details companion to
> [agent-tracks](https://github.com/inventor71/agent-tracks). agent-tracks gives you the
> **isolation + recombination** mechanism (worktrees, merge queue, partition discipline);
> this repo gives you the **full AI-DLC workflow** (requirements → design → code → test)
> rebuilt on top of that philosophy. Use them together, or use agent-tracks alone and
> bring your own process.

---

## What's different from upstream AI-DLC

The original [awslabs/aidlc-workflows](https://github.com/awslabs/aidlc-workflows) (v0.1.8,
MIT-0) runs one session at a time — state lives in shared root files, and there's no
concept of parallel tracks. This fork adds:

| Addition | What it does |
|---|---|
| **`common/concurrent-tracks.md`** | The core discipline: partition, don't lock. Every state file has exactly one writer. Track Registry + per-track docs + worktree gate. |
| **Track-aware phases** | Every Inception and Construction rule file references per-track paths (`.aidlc-docs/tracks/<id>/`) instead of shared root files. |
| **`merge-awaiting` hand-off** | `build-and-test.md` Step 8: when tests pass, set `merge-awaiting` → enqueues for `/ai-dlc-merge`. |
| **Workspace detection** | Checks the Track Registry for active tracks; suggests resume if found. |
| **Session continuity** | Reconstructs position from per-track `state.md` + `audit.md`. |

The upstream AI-DLC workflow structure (phase order, adaptive depth, approval gates) is
preserved. What changes is *where state lives* and *how tracks hand off to merge*.

## Quick start

Copy into your project alongside agent-tracks:

```
your-repo/
├── .aidlc-rule-details/   ← this repo (AI-DLC rule files, track-aware)
├── .claude/
│   ├── commands/
│   │   ├── ai-dlc-*        ← this repo (AI-DLC process commands)
│   │   └── track-*         ← agent-tracks (track mechanism commands)
│   ├── agents/             ← agent-tracks critic subagent
│   └── hooks/              ← this repo (worktree gate for aidlc-docs paths)
├── .tracks/                ← agent-tracks (track template + registry for bare track-* commands)
└── aidlc-docs/             ← created at runtime by /ai-dlc-request (Track Registry + per-track state)
```

Activate the worktree gate hook in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|NotebookEdit",
        "command": "python .claude/hooks/guard-main-edits.py"
      }
    ]
  }
}
```

Then point your project's `CLAUDE.md` at both:

```markdown
## AI-DLC workflow
Load and obey `.aidlc-rule-details/common/concurrent-tracks.md` at workflow start.
All per-track state lives under `.aidlc-docs/tracks/<id>/`.

## Parallel work
This repo uses agent-tracks for concurrent work. Never generate code outside a
track worktree.

## Slash commands
- `/ai-dlc-request` — start new AI-DLC work (the front door)
- `/ai-dlc-merge` — sequential merge orchestrator
- `/ai-dlc-status` — read-only progress dashboard
- `/ai-dlc-resume` — continue from breakpoint
- `/ai-dlc-refactor` — behavior-preserving redesign
- `/ai-dlc-deprecate` — safe feature removal
- `/track-*` — track lifecycle (from agent-tracks)
```

## Relationship to agent-tracks

| Concern | agent-tracks | aidlc-workflows-concurrent |
|---|---|---|
| Worktree isolation | ✅ `track-setup.sh`, worktree gate | ✅ enforces via rule prose |
| Serialized merge | ✅ `/track-merge` orchestrator | ✅ `/ai-dlc-merge` (AI-DLC-aware) |
| Track lifecycle | ✅ create → work → merge-awaiting → merged | ✅ same, integrated into AI-DLC phases |
| Requirements / Design | — (bring your own) | ✅ full AI-DLC: requirements → design → code → test |
| Per-track state | `.tracks/<id>/` | `.aidlc-docs/tracks/<id>/` |
| Slash commands | `track-*` (mechanism) | `ai-dlc-*` (process) |
| Guard hook | ✅ `.tracks/`-based variant | ✅ `aidlc-docs/`-based variant |
| Critic subagent | ✅ `critic` agent + command | — (use agent-tracks') |

The two repos share the same philosophy and conventions (`t1`/`t2` ids, `merge-awaiting`
hand-off, single-writer partition). agent-tracks is the mechanism; aidlc-workflows-concurrent
is a full process built on it.

## Origin

Customized from [awslabs/aidlc-workflows](https://github.com/awslabs/aidlc-workflows)
**v0.1.8** (MIT-0) by integrating the [agent-tracks](https://github.com/inventor71/agent-tracks)
concurrent-track discipline into every phase of the workflow. The customization was
extracted from a production project that ran 60+ concurrent tracks.

## License

MIT — see [LICENSE](LICENSE).

The original AI-DLC workflow rules are MIT-0 (AWS Labs). Modifications are MIT.
