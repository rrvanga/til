# A transient fetch blip killed the whole cron run — bounded retries absorb it, then fail loudly

**Date:** 2026-09-14
**Tags:** `bash`, `cron`, `retries`

## The problem

The llmcost daily pricing cron (Mon–Fri data refresh) ran `python3 -m llmcost.fetch` once and committed the result. On 2026-09-11 the single urllib call died on a temporary DNS resolution error, and because the script is `set -euo pipefail`, that one blip aborted the entire run — no retry, no data update, a failed job for a problem that would have resolved itself in seconds.

## What I tried

The tempting fixes were wrong for an unattended cron:

- Just re-running the script by hand — treats a symptom, leaves tomorrow's run just as fragile.
- Dropping `set -e` so a failed fetch is ignored — that would also mask a genuinely unreachable source and let a stale `data/prices.json` commit silently.

The script already diffed against `HEAD` to avoid timestamp-only commits, so the missing piece was purely around the fetch: absorb transient failures, but keep failing loudly for real outages.

## What worked

A bounded retry loop wrapped the fetch: up to 3 attempts with linear backoff (5s, then 10s), then `exit 1` with a message. Committed as `ae1a2ef scripts: retry transient fetch failures in update.sh`; the very next scheduled run (`5a9ce99 data: update model pricing (2026-09-14)`) landed successfully under the new logic.

```bash
attempt=0
while :; do
    if out="$(python3 -m llmcost.fetch)"; then
        break
    fi
    attempt=$((attempt + 1))
    if [ "$attempt" -ge 3 ]; then
        echo "llmcost.fetch failed after $attempt attempts" >&2
        exit 1
    fi
    echo "llmcost.fetch failed (attempt $attempt); retrying in $((attempt * 5))s..." >&2
    sleep "$((attempt * 5))"
done
```

I reproduced the failure mode with a fake fetch that fails twice then succeeds (same loop, `sleep 0` for speed):

```text
fetch failed (attempt 1); retrying in 5s...
fetch failed (attempt 2); retrying in 10s...
fetch OK: price data for 2026-09-14
```

## Takeaway

In unattended cron fetches, classify failures before you fail: bounded retries with backoff absorb network blips (DNS, connectivity) that would otherwise kill the whole job, while keeping a hard attempt cap so a real outage still ends in a loud `exit 1` — never in silence.