# A failed monitor run didn't write `union-alpha`'s price — the next convergence pass did

**Date:** 2026-09-22
**Tags:** `cron`, `monitoring`, `verification`

## The problem

2026-09-18 19:32:52 the Go adaptive monitor cron died with `APIConnectionError` (0 tool calls, `max_retries_exhausted`) — yet both rate scripts carried a `union-alpha` `# GUESS ≈ omen-alpha` entry stamped 2026-09-18, and llmcost's `prices.json` also gained a `union-alpha` row in a commit authored 9 seconds later. Who wrote the pricing if the detecting run never ran a single tool?

## What I tried

- Read the failed run's request dump: `reason: max_retries_exhausted` + `APIConnectionError "Connection error."` — proves 0 tool calls; that run touched nothing itself.
- `git log -S 'union-alpha'` on llmcost: `d81501d` (09-18) added `openrouter/stealth/union-alpha` at **$0.0/$0.0** — a free promo in OpenRouter's catalog — and `cc527bd` (09-22) removed it when OpenRouter delisted the model. That commit came from a separate no_agent script (`llmcost daily update`, ran 19:33:03 in the same gateway-backfill window) — pure coincidence of timing, no causal link to the dead monitor.
- The actual author: the monitor's own skill-reference addendum logs an "autonomy run" completing the dead tick's intended action by hand. This works *because* a failed run never updates its previous-output-hash, so the next convergence pass re-sees the same change-diff and finishes it. The pre-edit backups' mtime (2026-09-18 19:34:17) places the edit ~90s after the crash.

## What worked

Attribution came from the request dump + `git -S` history + the state log — not from the cron output, which echoes the full prompt and reads as if the dead run did everything:

```bash
$ grep -o '"reason": "[^"]*"' ~/.hermes/sessions/request_dump_cron_81a8bc241220_20260918_193232_20260918_193252_527240.json
"reason": "max_retries_exhausted"

$ git -C ~/dev/llmcost log --all -S 'union-alpha' --oneline -- data/prices.json
cc527bd data: update model pricing (2026-09-22)
d81501d data: update model pricing (2026-09-18)

$ git -C ~/dev/llmcost show d81501d:data/prices.json | grep -A3 -m1 '"model": "openrouter/stealth/union-alpha"'
      "model": "openrouter/stealth/union-alpha",
      "provider": "openrouter",
      "input_usd_per_mtok": 0.0,   # scriptable catalog price vs the sibling-analogy GUESS (0.20, 0.66)
      "output_usd_per_mtok": 0.0,
```

## Takeaway

A run marked failed with 0 tool calls cannot be the author of anything — but monitor-style jobs guarantee the same change-diff is re-seen by the next healthy pass, so trace attribution to *that* pass (request dump + state log) before blaming a ghost, and prefer a scriptable price source over a sibling-analogy guess.