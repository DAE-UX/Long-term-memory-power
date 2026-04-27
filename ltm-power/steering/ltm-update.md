# LTM Update

> **TL;DR:** Updates the LTM tooling (`ltm.py`, hooks, workspace steering) to the latest version from the power without affecting stored memory data.

## When to use

The user says "Update LTM", "update memory tooling", or equivalent.

## What gets updated

- `ltm/bin/ltm.py` — replaced with the latest version from `ltm-script-source.md`.
- `.kiro/hooks/ltm-postturn-capture.kiro.hook` — regenerated with current `python_cmd`.
- `.kiro/steering/ltm-operations.md` — regenerated from bootstrap template.
- `.kiro/steering/ltm-memory-format.md` — regenerated from bootstrap template.
- `ltm/manifest.json` — updated with new `ltm_py_hash` and timestamp.
- `ltm/config.json` — `version` field updated. Existing settings preserved.

## What is NOT touched

- `ltm/store/*.jsonl` — all memory data preserved.
- `ltm/runtime/*` — runtime artifacts preserved (regenerated after update if needed).
- `ltm/reports/*` — reports preserved.
- `ltm/snapshots/*` — snapshots preserved.
- `.gitignore` block — preserved as-is.

## Update procedure

1. **Verify LTM exists:** Check `ltm/config.json` has `"created_by": "ltm-power"`. If not, abort.
2. **Read current config:** Load `ltm/config.json`. Preserve all user settings (`exclude_paths`, `sensitive_path_patterns`, `session_timeout_minutes`, etc.).
3. **Read `python_cmd`:** Use the existing `python_cmd` from config. If it no longer works, re-detect.
4. **Write new `ltm.py`:** Read `ltm-script-source.md`, write to `ltm/bin/ltm.py`. Verify SHA-256 hash.
5. **Regenerate hook:** Write `.kiro/hooks/ltm-postturn-capture.kiro.hook` with current `python_cmd`.
6. **Regenerate workspace steering:** Write `.kiro/steering/ltm-operations.md` and `.kiro/steering/ltm-memory-format.md` from the templates in `ltm-bootstrap.md` step 8.
7. **Update config version:** Set `version` to the power's current version. Preserve all other fields.
8. **Update manifest:** Update `ltm_py_hash`, `version`, and timestamp. Preserve file lists.
9. **Run selftest:** `<python_cmd> ltm/bin/ltm.py selftest --quick`. If it fails, report and offer rollback.
10. **Run health:** `<python_cmd> ltm/bin/ltm.py health`. Report results.
11. **Report:** "LTM tooling updated to v{version}. All memory data preserved."

## Rollback

If the update fails (selftest or health fails):
1. The previous `ltm.py` can be restored from git (`git checkout -- ltm/bin/ltm.py`).
2. Or run `<python_cmd> ltm/bin/ltm.py repair` to fix any issues.
3. Memory data is never at risk — the update only touches tooling files.

## Safety rules

- **NEVER** modify or delete files in `ltm/store/`.
- **NEVER** clear runtime artifacts during update.
- **ALWAYS** preserve user config settings.
- **ALWAYS** verify the new script with selftest before reporting success.
