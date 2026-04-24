# LTM Failure Recovery

## Degraded mode

If `ltm.py` fails or Python is unavailable, the agent reads and writes JSONL files directly using its file tools. This is more expensive in credits but functional.

**Rules in degraded mode:**
- Follow the same capture policy: no chain-of-thought, no full transcripts, no secrets.
- Use the same JSONL field formats documented in `ltm-memory-format.md`.
- Apply the same redaction patterns.
- Read `ltm/runtime/active-context.json` and `ltm/runtime/last-recall.md` for recall.

## If Python is unavailable

The agent performs all capture and query operations directly. Automatic capture via hooks is disabled. The user must explicitly ask for checkpoints and recall.

## If runtime artifacts are missing

The agent regenerates them from ledger data:
1. Read recent entries from `events.jsonl`, `sessions.jsonl`, `checkpoints.jsonl`, `open_threads.jsonl`.
2. Generate `active-context.json` and `last-recall.md` following the format specs.
3. If `ltm.py` is available, run `<python_cmd> ltm/bin/ltm.py regenerate` instead.

## If ledger files are corrupted

Run repair: `<python_cmd> ltm/bin/ltm.py repair`

Repair behavior:
- Recreate missing directories and files with empty/placeholder content.
- Remove only incomplete trailing lines from JSONL files (lines that are not valid JSON).
- Preserve complete JSON records even if they fail schema validation — `validate` reports those records.
- Emit health report after repair.

## If hooks are disabled

The system works without hooks — just without automated capture. The agent can perform capture manually when instructed. Recall and checkpoints work normally.

## If the hook fires but capture fails

The capture script writes errors to stderr (shown as a warning to the user). Events may be silently lost if the script fails repeatedly. The `health` command detects this through stale event timestamps and reports `degraded` capture coverage.

## Repair command reference

```bash
<python_cmd> ltm/bin/ltm.py repair
```

This command:
- Recreates missing `ltm/` subdirectories.
- Recreates missing JSONL files as empty files.
- Recreates missing runtime files with placeholder content.
- Truncates incomplete trailing lines in JSONL files.
- Does NOT delete complete records, even if they have invalid fields.
- Reports what was repaired to stdout.

Read `python_cmd` from `ltm/config.json`.
