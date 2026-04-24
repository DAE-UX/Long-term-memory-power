# LTM Recall Workflow

## Progressive disclosure

Use the cheapest recall tier that answers the question. Escalate only when needed.

### Tier 1 — Cheap scan (budget ~50 tokens)

Read `python_cmd` from `ltm/config.json`, then run:

```bash
<python_cmd> ltm/bin/ltm.py files --limit 10
<python_cmd> ltm/bin/ltm.py sessions --limit 5
```

If `ltm.py` is unavailable, read `ltm/runtime/active-context.json` and `ltm/runtime/last-recall.md` directly.

This is usually enough to orient the agent.

### Tier 2 — Focused recall (budget ~200 tokens)

Use when the user references specific prior work ("continue the auth refactor", "what did we decide yesterday?"):

```bash
<python_cmd> ltm/bin/ltm.py search "term" --days 5
<python_cmd> ltm/bin/ltm.py checkpoints --days 3
<python_cmd> ltm/bin/ltm.py decisions --days 5
```

Optionally supplement with git context:

```bash
git status --short
git log --oneline -5
```

### Tier 3 — Deep recall (budget ~500 tokens)

Use only when investigating a specific session:

```bash
<python_cmd> ltm/bin/ltm.py show <session_id>
```

### Fallback

If `ltm.py` is unavailable, read the last 20 lines of `ltm/store/events.jsonl` directly. If writing directly to LTM ledgers in degraded mode, explicitly read `ltm-memory-format.md` for record formats.

## Hard rules

- Tier budgets are approximate. Hard enforcement is via record limits, snippet truncation, max stdout size (4KB), and runtime artifact caps.
- Never inject raw entire ledger files into context.
- If runtime artifacts are >1 hour old, regenerate them before using: `<python_cmd> ltm/bin/ltm.py regenerate`.

## Runtime artifact caps

- `last-recall.md`: max 400 words. If exceeded, drop oldest checkpoint summaries.
- `active-context.json`: max 8KB (preferred 2KB). If exceeded, truncate arrays.
- `recent_files` in active-context: max 15 entries.
- `open_threads` in active-context: max 10 entries.

## Token budget status

- `pass` = under preferred limits
- `warn` = between preferred and hard ceiling
- `fail` = over hard ceiling
