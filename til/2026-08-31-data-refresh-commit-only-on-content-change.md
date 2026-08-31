# A daily data-refresh script commits only when content changed, not when the timestamp changed

**Date:** 2026-08-31
**Tags:** `git`, `automation`, `cron`, `data-engineering`, `bash`

## The problem

`llmcost` re-fetches LLM pricing every weekday, but a naive "regenerate and commit" cron would create a useless commit every single day — because the `generated_at` timestamp at the top of `data/prices.json` legitimately changes on every run, even when no price actually moved. A daily cron that commits noise trains you to ignore the log, and then the day a real price change lands you miss it.

## What I tried

Naive comparison first: just diff the new file against `HEAD` with `git diff --quiet` — always non-zero, because `generated_at` differs every run. The repo's `scripts/update.sh` avoids that by stripping the timestamp line **from both sides** before comparing, so the only thing that can trigger a commit is real content (a new model, a price change).

## What worked

The whole check is three commands; today's run demonstrated it live:

```bash
cd ~/dev/llmcost
old="$(git show HEAD:data/prices.json | grep -v '"generated_at"')"
new="$(grep -v '"generated_at"' data/prices.json)"
[ "$old" = "$new" ] && { git restore data/prices.json; exit 0; }   # silent no-op
```

Real output from this session (working tree already clean after today's commit):

```
== raw compare INCLUDING generated_at (differs every run) ==
generated_at lines in HEAD: 1, in working tree: 1
== compare EXCLUDING generated_at (the real signal) ==
MATCH -> no real change since last commit, so: git restore, exit 0 (silent)
```

Today's commit (`0541e17`, 2026-08-31) was a *real* one: 29 new models added (model_count 2985 → 3014) and one price hike — `bedrock_mantle/openai.gpt-5.6-sol` went 4.4 → 5.5 in / 22.0 → 33.0 out. The commit message even says which day it's for, and the `update.sh` header notes the point explicitly: *"Silent (exit 0, no output) when nothing changed — so the cron watchdog stays quiet."*

## Takeaway

When a generated dataset carries a legitimate per-run timestamp, compare the two versions with that volatile field stripped — then a cron can commit only when reality changed, and stay silent the rest of the time.