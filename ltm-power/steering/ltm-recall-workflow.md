# LTM Recall Workflow

Use the cheapest tier that answers the question. Escalate only when needed.

## Tier 1 — Cheap scan (~50 tokens)

```bash
<python_cmd> ltm/bin/ltm.py files --limit 10
<python_cmd> ltm/bin/ltm.py sessions --limit 5
```

Fallback: read `ltm/runtime/active-context.json` and `ltm/runtime/last-recall.md`.

Usually enough to orient the agent.

## Tier 2 — Focused recall (~200 tokens)

Use when user references specific past work:

```bash
<python_cmd> ltm/bin/ltm.py search "term" --days 5
<python_cmd> ltm/bin/ltm.py checkpoints --days 3
<python_cmd> ltm/bin/ltm.py decisions --days 5
```

Optional git supplement: `git status --short` and `git log --oneline -5`.

## Tier 3 — Deep recall (~500 tokens)

Use only for specific session investigation:

```bash
<python_cmd> ltm/bin/ltm.py show <session_id>
```

## Fallback (no ltm.py)

Read last 20 lines of `ltm/store/events.jsonl`. For degraded-mode writes, follow formats in `ltm-memory-format.md`.

## Hard rules

- Tier budgets are approximate. Enforced via record limits, snippet truncation, 4KB max stdout, and artifact caps.
- **NEVER** inject raw entire ledger files into context.
- If runtime artifacts are >1 hour old, regenerate first: `<python_cmd> ltm/bin/ltm.py regenerate`.

## Artifact caps

| Artifact | Limit |
|----------|-------|
| `last-recall.md` | 400 words |
| `active-context.json` | 8KB hard, 2KB preferred |
| `recent_files` | 15 entries |
| `open_threads` | 10 entries |

## Token budget status

`pass` = under preferred, `warn` = between preferred and hard ceiling, `fail` = over.

---

Read `python_cmd` from `ltm/config.json`.
