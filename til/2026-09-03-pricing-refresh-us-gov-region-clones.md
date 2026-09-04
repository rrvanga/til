# A pricing refresh added 25 US-gov region clones — diff the snapshots, don't trust model_count

**Date:** 2026-09-03
**Tags:** `llmcost`, `data-refresh`, `json-diff`

## The problem

The `llmcost daily update` cron committed `data/prices.json` (fetched from the upstream litellm pricing source via `python3 -m llmcost.fetch`), and `model_count` jumped 3115 → 3154. A raw count bump tells you nothing about *what* actually changed — and `jq` isn't installed on this box, so the usual provider-drill-down was unavailable.

## What I tried

`jq -r '.models[].model' data/prices.json …` → `jq: command not found`. Rather than install a package mid-cron, I fell back to Python: pull the previous snapshot out of git, load both JSON files, and diff with `collections.Counter`.

## What worked

Git keeps the previous snapshot at `HEAD~1` — no backup needed:

```bash
git -C ~/dev/llmcost show HEAD~1:data/prices.json > /tmp/prices_before.json
python3 /tmp/prices_diff2.py /tmp/prices_before.json ~/dev/llmcost/data/prices.json
```

Output (trimmed):

```
added total: 40
  containing 'gov': 25
REMOVED: azure_ai/deepseek-v4-flash-0731

bedrock/us-gov-east-1/openai.gpt-oss-120b-1:0    1
bedrock/us-gov-west-1/anthropic.claude-opus-4-8  1
azure/us-gov/gpt-5.1                             1
bedrock_mantle/us-gov-west-1/xai.grok-4.3        1
us-gov.anthropic.claude-sonnet-5                 1
scaleway/deepseek-v4-flash-0731                  1
gemini-3.8-flash                                 1   # + gemini/gemini-3.8-flash, vertex_ai/gemini-3.8-flash
```

The script core is a 10-line Counter over `m["model"].split("/", 1)[0]`, plus set-difference of model names.

Three findings:

1. **62.5% of the refresh (25/40) were US Government region clones** — the same models re-listed under `bedrock/us-gov-east-1/*`, `bedrock/us-gov-west-1/*`, `azure/us-gov/*`, `bedrock_mantle/us-gov-*`, and bare `us-gov.anthropic.*`. No new model *families*, just region namespace expansion.
2. **The "1 removed" was a rename, not a loss**: `azure_ai/deepseek-v4-flash-0731` was removed and re-added as `azure_ai/DeepSeek-V4-Flash-0731` (case normalized) while `scaleway/deepseek-v4-flash-0731` appeared as a new provider row.
3. **Same model, many namespaces**: `gemini-3.8-flash` now exists in three rows (bare, `gemini/`, `vertex_ai/`) — raw counts overcount a single model.

## Takeaway

When an upstream-driven dataset grows by N, diff the previous git snapshot before believing the headline count — a +40 "new models" refresh was mostly region clones plus a rename, and the real new models were a handful.