# STM Update

> **TL;DR:** Updates the STM tooling (`stm.py`, hooks, workspace steering) to the latest version from the power without affecting stored observation data.

## When to use

The user says "Update STM", "update introspection tooling", or equivalent.

## What gets updated

- `stm/bin/stm.py` — replaced with the latest version from `stm-script-source.md`.
- `.kiro/hooks/stm-*.kiro.hook` — regenerated with current `python_cmd`.
- `.kiro/steering/stm-*.md` — regenerated from bootstrap templates.
- `stm/manifest.json` — updated with new `stm_py_hash` and timestamp.
- `stm/config.json` — `version` field updated. All user settings preserved.

## What is NOT touched

- `stm/store/*.jsonl` — all observation and cluster data preserved.
- `stm/runtime/*` — runtime artifacts preserved.
- `.gitignore` block — preserved as-is.

## Update procedure

1. **Verify STM exists:** Check `stm/config.json` has `"created_by": "stm-power"`. If not, abort.
2. **Read current config:** Load `stm/config.json`. Preserve all user settings (`scopes`, `consensus_threshold`, `graduation_output_path`, etc.).
3. **Read `python_cmd`:** Use the existing `python_cmd` from config. If it no longer works, re-detect. If LTM is present, check for `python_cmd` divergence and offer to sync.
4. **Write new `stm.py`:** Read `stm-script-source.md`, write to `stm/bin/stm.py`. Verify SHA-256 hash.
5. **Regenerate hooks:** Write all `.kiro/hooks/stm-*.kiro.hook` files with current `python_cmd`.
6. **Regenerate workspace steering:** Write `.kiro/steering/stm-observations.md` and `.kiro/steering/stm-memory-format.md` from the templates in `stm-bootstrap.md` step 8.
7. **Update config version:** Set `version` to the power's current version. Preserve all other fields.
8. **Update manifest:** Update `stm_py_hash`, `version`, and timestamp. Preserve file lists.
9. **Run selftest:** `<python_cmd> stm/bin/stm.py selftest --quick`. If it fails, report and offer rollback.
10. **Run health:** `<python_cmd> stm/bin/stm.py health`. Report results.
11. **Report:** "STM tooling updated to v{version}. All observation data preserved."

## Rollback

If the update fails:
1. Restore previous `stm.py` from git (`git checkout -- stm/bin/stm.py`).
2. Or run `<python_cmd> stm/bin/stm.py repair` to fix any issues.
3. Observation data is never at risk — the update only touches tooling files.

## Safety rules

- **NEVER** modify or delete files in `stm/store/`.
- **NEVER** clear runtime artifacts during update.
- **ALWAYS** preserve user config settings.
- **ALWAYS** verify the new script with selftest before reporting success.

Read `python_cmd` from `stm/config.json`.
