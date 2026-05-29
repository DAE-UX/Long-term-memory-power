---
name: "stm-power"
displayName: "STM Introspection"
description: "Agent self-improvement through structured observation capture. Records what works and what doesn't, builds consensus across sessions, graduates validated insights into actionable learning proposals, and supports validation, repair, and in-place updates. Works alongside LTM Power or standalone."
author: "AWS"
keywords:
  - "memory"
  - "introspection"
  - "stm"
  - "observations"
  - "what worked"
  - "what didn't work"
  - "learning"
  - "feedback"
  - "reflect"
  - "consensus"
  - "improve"
  - "self-improvement"
  - "remember what works"
---

# STM Introspection

Agent self-improvement through structured observation capture. Records what works and what doesn't during agent sessions, builds consensus across sessions, and graduates validated insights into actionable learning proposals.

Works alongside LTM Power or standalone.

## Getting started

Say **"Remember what works."**

## How it works

The agent captures point-by-point observations about what worked and what didn't. Observations are structured JSONL with scope, topic, stance, claim, evidence, and situational cues. When enough observations agree, the system graduates consensus insights into actionable learning files.

Pipeline: **capture → compress → distill → graduate → project integration.**

## Relationship to LTM

LTM tracks **what happened** — files changed, sessions, checkpoints, decisions.
STM tracks **what the agent learned from what happened** — the evaluative layer.

When both are installed, STM reads LTM's session IDs to link observations to sessions. When LTM is absent, STM manages its own session IDs. The dependency is one-way: STM → LTM. LTM never reads from STM.

## Onboarding

### Step 1: Validate Python

```bash
python3 --version   # try first
python --version    # fallback
```

Require 3.9+. Store as `python_cmd`. If unavailable, follow degraded path in `stm-bootstrap.md`.

### Step 2: Check existing introspection

If `stm/` exists, check `stm/config.json` for `"created_by": "stm-power"`:
- Healthy → report and offer recall/purge.
- Damaged → repair via `stm-failure-recovery.md`.

### Step 3: Bootstrap

If no `stm/`, read `stm-bootstrap.md` and execute.

## Commands

| You say | What happens |
|---------|-------------|
| "Remember what works." | Bootstrap STM |
| "That approach worked well." | Record `user:feedback` observation with `supports` stance |
| "That was wrong." / "That didn't work." | Record `user:feedback` observation with `contradicts` stance |
| "Reflect on this session." | Trigger stm-reflect hook |
| "What topics have you observed?" | Run `stm.py topics` |
| "What have you learned?" | Run `stm.py recall` for summary |
| "What learnings apply here?" | Run `stm.py learnings --cue "<situation>"` |
| "Search observations." | Run `stm.py search "<term>" [--scope <scope>] [--days N]` |
| "Run STM lifecycle." | Trigger stm-lifecycle hook (compress → distill → graduate) |
| "Show STM status." | Run `stm.py status` |
| "Share introspection with the team." | Run `stm.py sharing --mode shared` |
| "Keep introspection local." | Run `stm.py sharing --mode local` |
| "Forget all observations." | Run STM `purge-all --confirm` only |
| "Forget all memory." | Run both STM and LTM `purge-all --confirm` (if LTM installed) |
| "Remove STM from this project." | Run `stm.py teardown --confirm` |
| "Remove all memory." | Teardown STM first, then LTM (order matters) |
| "Validate STM." | Run `stm.py validate` |
| "Repair STM." | Run `stm.py repair` |
| "Update STM." | Update tooling to latest version |

## When to Load Steering Files

- Bootstrapping → `stm-bootstrap.md`
- Writing stm.py → `stm-script-source.md`
- Capture rules → `stm-capture-policy.md`
- Purge/reset → `stm-purge-and-reset.md`
- Validation → `stm-validation-policy.md`
- Recovery → `stm-failure-recovery.md`
- Updating STM tooling → `stm-update.md`

## Tool Trace

STM can passively capture tool invocations via a `postToolUse` hook. This creates a lightweight runtime artifact (`stm/runtime/tool-trace.jsonl`) that grounds the auto-reflect hook in actual events rather than relying solely on agent memory.

- Trace entries are capped (default 50 per session) and auto-rotated.
- The auto-reflect hook references the trace summary to identify patterns.
- When LTM is present, LTM's event log provides richer grounding and the tool-trace serves as a fast-access supplement for the current session only.
- Trace data is always gitignored (runtime artifact).

## Sharing Mode

STM supports two sharing modes:

- **local** (default): Store files (`observations.jsonl`, `clusters.jsonl`) are gitignored. Introspection is private to the developer.
- **shared**: Store files are committed to the repo. All collaborators see and contribute to the same introspection data.

### Conflict Avoidance in Shared Mode

When sharing is enabled:
1. Observations use append-only JSONL — git merges are line-based and rarely conflict.
2. Each observation carries a unique ID and session ID — concurrent writes from different developers produce distinct records.
3. The `distill` command handles duplicate detection via topic+scope grouping — overlapping observations from different users strengthen consensus rather than conflicting.
4. Runtime files remain gitignored — they are per-machine state.
5. `config.json` and `manifest.json` are committed — changes to config should be coordinated (standard PR workflow).

### Choosing During Bootstrap

During bootstrap, the agent asks: "Should introspection data be shared with the team (committed to repo) or kept local (gitignored)?" The choice sets `sharing_mode` in config and adjusts `.gitignore` accordingly.

## Learnings Integration

Graduated learnings are the output of the STM pipeline — validated insights with cue patterns. To prevent repeated mistakes:

1. The `learnings` command lists all graduated learnings with their cue patterns.
2. The workspace steering file instructs the agent to run `stm.py recall --cue "<situation>"` before making decisions in scopes where prior observations exist.
3. When LTM is present, the agent can cross-reference LTM's session context with STM's cue patterns for richer pre-task recall.

## Scope Maturity

When a scope accumulates enough graduated learnings (default: 5), the `status` command reports it as "mature." This signals that the scope's learnings may be worth consolidating into a steering file or project documentation for permanent integration.

## LTM Integration

When LTM is installed alongside STM:
- STM reads LTM's session IDs (no duplicate session management).
- The reflect hook reads LTM's event log for grounding observations in actual file changes.
- LTM's `capture-turn` hook already records tool activity — STM's tool-trace supplements this with outcome tracking for the current session.
- The agent should prefer LTM events for "what happened" and STM observations for "what we learned from it."

## License & Support

**License:** MIT — see [LICENSE](../LICENSE)

**Author:** AWS

**Issues & feedback:** [github.com/DAE-UX/STM-Introspection-power](https://github.com/DAE-UX/STM-Introspection-power/issues)
