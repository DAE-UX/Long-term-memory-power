# LTM Update

> **TL;DR:** Updates the LTM tooling (`ltm.py`, hooks, workspace steering) to the latest version from the power without affecting stored memory data.

## When to use

The user says "Update LTM", "update memory tooling", or equivalent.

## What gets updated

- `ltm/bin/ltm.py` — replaced with the latest version from `ltm-script-source.md`.
- `ltm/README.md` — regenerated from bootstrap template (step 5) with current hook filenames.
- `.kiro/hooks/ltm-postturn-capture.json` and/or `.kiro/hooks/ltm-postturn-capture.kiro.hook` — regenerated with current `python_cmd` (format depends on Kiro version; see step 5).
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
4. **Write new `ltm.py`:** Read `ltm-script-source.md`, write to `ltm/bin/ltm.py` using the chunked write procedure from `ltm-bootstrap.md` Phase 2. Verify with selftest.
5. **Regenerate hook:** Re-detect the consumer's Kiro version (per `ltm-bootstrap.md` step 7). Then apply the correct format:
   - If only `.kiro/hooks/ltm-postturn-capture.kiro.hook` exists and the user is now on Kiro 0.12.315+ → write the new `.json` (v2) file and remove the old `.kiro.hook`.
   - If only `.kiro/hooks/ltm-postturn-capture.json` exists → regenerate it.
   - If both exist → regenerate both.
   - If neither exists → scaffold per the version-detection logic.
   - Update the `hooks` list in `ltm/manifest.json` to reflect whichever file(s) now exist.
6. **Regenerate README:** Write `ltm/README.md` from the template in `ltm-bootstrap.md` step 5, substituting current `python_cmd` and hook filename(s).
7. **Regenerate workspace steering:** Write `.kiro/steering/ltm-operations.md` and `.kiro/steering/ltm-memory-format.md` from the templates in `ltm-bootstrap.md` step 8.
8. **Update config version:** Set `version` to the power's current version. Preserve all other fields.
9. **Update manifest:** Update `ltm_py_hash`, `version`, timestamp, and `hooks` list (to reflect current hook file(s) from step 5). Preserve other file lists.
10. **Run selftest:** `<python_cmd> ltm/bin/ltm.py selftest --quick`. If it fails, report and offer rollback.
11. **Run health:** `<python_cmd> ltm/bin/ltm.py health`. Report results.
12. **Report:** "LTM tooling updated to v{version}. All memory data preserved."

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
