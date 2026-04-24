# ltm-power — Local Long-Term Memory for Kiro

A Kiro power that gives your AI coding agent project-local long-term memory.

**Author:** AWS

## What it does

ltm-power scaffolds a memory system inside your repo. Once set up, the agent automatically tracks what files changed each turn, and you can ask it to recall past work, save checkpoints, or forget sessions — without re-explaining your project every time.

The memory lives in an `ltm/` folder at your project root. It uses plain JSONL files for storage and a single Python script for queries. No external services, no databases, no background processes.

## Install

1. Open Kiro.
2. Open the Powers panel.
3. Select "Add power from GitHub."
4. Enter: `https://github.com/DAE-UX/Long-term-memory-power`
5. Say **"Remember this project."** to bootstrap.

## How it works

1. **Say "Remember this project."** The power creates the `ltm/` folder, a capture hook, workspace steering files, and a CLI tool.
2. **Work normally.** After each agent turn that writes files, a lightweight script records what happened. No extra prompts, no credit cost.
3. **Resume later.** Say "Pick up where we left off" or "What do you remember?" The agent reads compact recall artifacts instead of searching your whole project.
4. **Save milestones.** Say "Save a checkpoint" after meaningful progress. The agent writes a rich summary with decisions, open threads, and next actions.

## Features

- **Automatic capture** — a hook records file changes after each agent turn (~100ms, zero credits).
- **Cheap recall** — the agent queries memory in ~50 tokens instead of grepping your codebase for ~10,000 tokens.
- **Progressive disclosure** — three tiers of recall depth. Most questions are answered by the cheapest tier.
- **Session tracking** — work is grouped into sessions with automatic rollover after inactivity.
- **Checkpoints** — durable milestone summaries with decisions, files, threads, and next actions.
- **Purge and reset** — forget the last session, forget everything, or fully remove LTM from the project.
- **Self-contained** — the memory system works even if the power is uninstalled.

## Commands

| You say | What happens |
|---------|-------------|
| "Remember this project." | Bootstrap the LTM scaffold |
| "Pick up where we left off." | Recall recent work |
| "What did we work on yesterday?" | Focused recall with search |
| "Save a checkpoint." | Write a milestone summary |
| "Forget the last session." | Remove only the latest session |
| "Forget everything." | Clear all memory, keep structure |
| "Remove LTM from this project." | Full teardown, no orphaned files |
| "Validate memory." | Run health check |
| "Repair memory." | Fix missing or damaged files |

## Requirements

- **Kiro IDE** with powers support.
- **Python 3.9+** for automatic capture and CLI queries. Without Python, the system runs in degraded mode where the agent handles memory operations directly.
- **Git** for automatic file tracking. Without git, file recall depends on explicit checkpoints.

## Privacy

- Memory is local to your machine. Nothing is uploaded.
- Sensitive file paths (`.env`, `secrets/`, `*.pem`, `*.key`, etc.) are filtered from capture.
- Secret patterns in text (API keys, tokens, credentials) are redacted before storage.
- Redaction is best-effort in v1. Do not rely on it as a security boundary.

## Limitations (v1)

- macOS and Linux only. Windows may work but is not tested.
- Single-root workspaces only.
- No vector search or semantic matching.
- Recall quality improves with checkpoints.

## License

MIT
