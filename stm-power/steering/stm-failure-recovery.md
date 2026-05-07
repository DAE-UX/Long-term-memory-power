# STM Failure Recovery

> **TL;DR:** Recovery procedures for common STM failure modes. Prioritizes data preservation over clean state.

## Python Unavailable

If Python becomes unavailable after bootstrap:
- Agent performs all operations directly via file writes.
- Hooks are disabled (they call the Python CLI).
- Use formats from `stm-memory-format.md` for direct writes.
- Same capture policy applies — the agent is responsible for validation.

## Runtime Artifacts Missing

```bash
<python_cmd> stm/bin/stm.py repair
```

Recreates missing directories (`stm/store/`, `stm/runtime/`, `stm/bin/`) and files (`status.json`, `current-session.json`, `merge-candidates.json`, `errors.jsonl`). Does not overwrite existing files.

## Store Files Corrupted

```bash
<python_cmd> stm/bin/stm.py repair
```

Removes only incomplete trailing lines from JSONL files. Preserves all complete records. Logs removed lines to `errors.jsonl`.

## LTM Becomes Unavailable After Bootstrap

If `ltm_available` was `true` at bootstrap but LTM is later removed:
1. STM's next `record` call fails to find `ltm/runtime/current-session.json`.
2. STM falls back to its own session management automatically.
3. `stm/config.json` → `ltm_available` is updated to `false`.
4. Existing observations retain their LTM `sess_*` IDs (still valid as identifiers).
5. New observations get STM `stm_sess_*` IDs.

No data loss. No user action required.

## LTM Installed After STM

If LTM is installed into a project that already has STM:
1. STM's next `record` call detects `ltm/runtime/current-session.json` exists.
2. STM switches to reading LTM's session ID.
3. `stm/config.json` → `ltm_available` is updated to `true`.
4. Existing observations retain their `stm_sess_*` IDs.
5. New observations get LTM `sess_*` IDs.

The transition is automatic.

## Manual Recovery

If `stm.py` is unavailable and repair is needed:
1. Check `stm/store/observations.jsonl` — each line should be valid JSON.
2. Remove any trailing incomplete line.
3. Recreate missing runtime files as empty JSON (`{}` for `.json`, `[]` for `merge-candidates.json`, empty for `.jsonl`).
4. Reinstall the power to regenerate `stm/bin/stm.py`.

Read `python_cmd` from `stm/config.json`.
