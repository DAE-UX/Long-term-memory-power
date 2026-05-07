# STM Validation Policy

> **TL;DR:** Health check dimensions, validation rules, and overall state classification for STM introspection data.

## Health Check

Read `python_cmd` from `stm/config.json`.

```bash
<python_cmd> stm/bin/stm.py health
```

### Dimensions

1. **Store integrity:** All JSONL lines are valid JSON.
2. **Schema compliance:** Observations have required fields, valid enums.
3. **Referential integrity:** Supersedes targets exist, cluster observation IDs exist, compressed_from targets exist.
4. **Compression state:** No orphaned compression candidates.
5. **Cluster state:** No graduated clusters with score < 0.
6. **Session state:** `current-session.json` is valid (either STM's own or LTM's is readable).
7. **LTM link:** If `ltm_available`, verify `ltm/runtime/current-session.json` is readable. If not, report `ltm_link: degraded`.
8. **Application tracking:** Graduated clusters with `application_success_rate < 0.5` and `application_count >= 3` are flagged for re-evaluation.

### Overall States

- **healthy** — all checks pass.
- **degraded** — non-critical issues (LTM link degraded, missing optional files).
- **broken** — store integrity or schema compliance failures.

## Validate

```bash
<python_cmd> stm/bin/stm.py validate
```

Checks structure, schema, and referential integrity. Reports all issues found. Exits non-zero if any issues exist.

## Selftest

```bash
<python_cmd> stm/bin/stm.py selftest [--quick]
```

Creates temporary fixtures in a temp directory. Verifies:
- JSONL read/write
- Schema validation (rejects bad input)
- Deduplication
- Distill grouping and scoring
- Compression marking
- Teardown path validation (rejects absolute paths, traversal)
