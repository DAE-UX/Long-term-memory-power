# stm-power — STM Introspection for Kiro

A Kiro power that gives your AI coding agent structured self-improvement through observation capture, consensus building, and graduated learning proposals.

**Author:** AWS

## What it does

stm-power adds an introspection layer to your project. The agent records what works and what doesn't as structured observations, builds consensus across sessions, and graduates validated insights into actionable learning files.

The introspection data lives in an `stm/` folder at your project root. It uses plain JSONL files for storage and a single Python script for operations. No external services, no databases, no background processes.

## Install

1. Open Kiro.
2. Open the Powers panel.
3. Select "Add power from GitHub."
4. Enter the repository URL.
5. Say **"Remember what works."** to bootstrap.

## How it works

1. **Say "Remember what works."** The power creates the `stm/` folder, hooks, workspace steering files, and a CLI tool.
2. **Work normally.** The agent notices what works and what doesn't, recording structured observations inline during work.
3. **Reflect.** Say "Reflect on this session" to generate end-of-session observations grounded in what actually happened.
4. **Graduate insights.** Say "Run STM lifecycle" to compress observations, build consensus, and graduate validated insights into learning files.

## Pipeline

**capture → compress → distill → graduate → project integration**

- **Capture:** Agent records observations with scope, topic, stance, claim, evidence, and situational cues.
- **Compress:** Groups of raw observations are summarized, preserving 1-2 exemplar cases.
- **Distill:** Observations are grouped by topic, scored for consensus, and merge candidates identified.
- **Graduate:** Clusters meeting consensus threshold are synthesized into actionable learning files.

## Features

- **Structured observations** — scope, topic, stance, claim, evidence, and situational cues.
- **Consensus building** — observations are scored and grouped. Insights graduate when N observations agree.
- **Reversible assertions** — later observations can flip consensus. The system doesn't lock in conclusions.
- **Boundary-aware** — each observation declares its scope. Lessons from one domain don't bleed into another.
- **LTM integration** — works alongside LTM Power for session linking, or standalone with its own session management.
- **Application tracking** — graduated learnings track whether they work when applied.
- **Self-contained** — the introspection system works even if the power is uninstalled.

## Commands

| You say | What happens |
|---------|-------------|
| "Remember what works." | Bootstrap STM |
| "That approach worked well." | Record positive feedback |
| "That didn't work." | Record negative feedback |
| "Reflect on this session." | Generate end-of-session observations |
| "What topics have you observed?" | List observation topics |
| "Run STM lifecycle." | Compress, distill, and graduate |
| "Show STM status." | View current state |
| "Forget all observations." | Clear observation data |
| "Remove STM from this project." | Full teardown |
| "Validate STM." | Run health check |
| "Repair STM." | Fix missing or damaged files |
| "Update STM." | Update tooling to latest version |

## Requirements

- **Kiro IDE** with powers support.
- **Python 3.9+** for CLI operations. Without Python, the system runs in degraded mode where the agent handles operations directly.

## Works with LTM Power

When both STM and LTM are installed:
- STM reads LTM's session IDs to link observations to sessions.
- The reflect hook reads LTM's event log to ground observations in what actually happened.
- Combined recall: LTM gives "here's what you were working on," STM gives "here's what you've learned about this kind of work."

When LTM is absent, STM manages its own sessions. The transition is automatic in both directions.

## Privacy

- All data is local to your machine. Nothing is uploaded.
- Observation data is stored as plain JSONL files.
- Store and runtime files are gitignored by default.

## Limitations (v1)

- macOS and Linux only. Windows may work but is not tested.
- Single-root workspaces only.
- No vector search or semantic matching — uses keyword overlap for cue matching.
- Consensus quality improves with compression. Raw-only clusters cannot graduate.

## License

MIT
