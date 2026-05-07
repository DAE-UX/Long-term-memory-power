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
| "Run STM lifecycle." | Trigger stm-lifecycle hook (compress → distill → graduate) |
| "Show STM status." | Run `stm.py status` |
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

## License & Support

**License:** MIT — see [LICENSE](../LICENSE)

**Author:** AWS

**Issues & feedback:** [github.com/DAE-UX/STM-Introspection-power](https://github.com/DAE-UX/STM-Introspection-power/issues)
