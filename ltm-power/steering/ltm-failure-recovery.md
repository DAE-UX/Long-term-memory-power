# LTM Failure Recovery

## Degraded mode

If `ltm.py` fails or Python is unavailable, the agent reads/writes JSONL files directly. More expensive in credits but functional.

**Rules:** Same capture policy applies — no chain-of-thought, no transcripts, no secrets. Use formats from `ltm-memory-format.md`. Apply redaction patterns.

## Recovery scenarios

### Python unavailable
Agent performs all operations directly. Hooks disabled. User must explicitly request checkpoints and recall.

### Runtime artifacts missing
1. Read recent entries from ledger files.
2. Generate `active-context.json` and `last-recall.md` per format specs.
3. If `ltm.py` available: `<python_cmd> ltm/bin/ltm.py regenerate`.

### Ledger files corrupted
Run: `<python_cmd> ltm/bin/ltm.py repair`

- Recreates missing directories and files.
- Removes only incomplete trailing lines (not valid JSON).
- Preserves complete records even if schema-invalid — `validate` reports those.
- Emits health report after repair.

### Hooks disabled
System works without hooks — no automated capture. Agent captures manually when instructed. Recall and checkpoints work normally.

### Hook fires but capture fails
Errors go to stderr (warning to user). Events may be lost if failures repeat. `health` detects stale timestamps and reports `degraded` capture coverage.

---

Read `python_cmd` from `ltm/config.json`.
