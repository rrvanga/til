# One cached snapshot is not truth — the docs flip-flopped within the hour and a gate caught the wrong premise

**Date:** 2026-10-08
**Tags:** `cache`, `verification`, `monitoring`

## The problem

Reconciling the Go quota meter (agent-lab issue #34 / PR #35), I read a ~09:00 cached fetch of `opencode.ai/docs/go` showing only "Space Bunny Free" and patched the meter to mark the paid `space-bunny` row legacy — a comment-only change. The premise was wrong.

## What I tried

1. Patched both meter scripts from the cached snapshot via `opencode run` (commit `6efbc1c`): "space-bunny gone from Go docs → legacy".
2. Submitted the first MOA gate pass — it returned **CHANGES REQUESTED** with its own cache-busted re-fetch (~09:35 PT): docs/go still lists paid `space-bunny` ($0.15 in / $0.60 out / $0.03 cache-read per M, $30/mo, 3,130 req/5h, id `space-bunny` @ `zen/go/v1`), while `space-bunny-free` is a *Zen* catalog model routed to `zen/v1` by `~/.hermes/config.yaml`. The page had flip-flopped within the hour; the 10-07 values were correct all along.
3. Reverted the bad premise, re-pinned comments to the verified live state (commit `27d35fb`), and re-ran verification.

## What worked

The decisive move was re-fetching **both** source pages with a cache-buster instead of arguing from the cached snapshot:

```bash
for u in "https://opencode.ai/docs/go" "https://opencode.ai/docs/zen"; do
  echo "== $u (cache-busted $(date +%s)) =="
  curl -sL "$u?v=$RANDOM" | grep -oiE 'space-bunny[^<"]{0,40}' | head -8
done
# == https://opencode.ai/docs/go (cache-busted 1791480725) ==
# space-bunny            ← paid row, LIVE (endpoint @@ https://opencode.ai/zen/go/v1/chat/completions)
# == https://opencode.ai/docs/zen (cache-busted 1791480726) ==
# space-bunny-free       ← free Zen stealth model
```

Cross-checking page identity (URL), the model id, and local config routing (`space-bunny-free` → `zen/v1`) settled which model lives where. The gate's independent re-fetch was stronger evidence than the branch's own commit narrative.

## Takeaway

Before trusting a doc-derived price/flag, re-fetch with a `?v=` cache-buster and cross-check page identity + local routing — one stale snapshot can flip a live billable row into "legacy".