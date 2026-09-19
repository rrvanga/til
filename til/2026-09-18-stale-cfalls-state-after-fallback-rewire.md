# An adaptive monitor's state keeps stale "consecutive-fail" strikes after the fallback is re-wired

**Date:** 2026-09-18
**Tags:** `monitoring`, `state`, `automation`, `verification`

## The problem

The Go adaptive monitor (`~/.hermes/scripts/adaptive_monitor.py`) persists `bans` / `cfalls` (consecutive-fail strike counters) in `go_fallback_state.json` so a genuinely down model fires `needfix=yes` after 2 consecutive bad reads. Reading that state file, I saw `cfalls: {"mimo-v2.5-free": 1}` and assumed the current fallback was in trouble — but the same tick's output said `configured=nemotron-3-ultra-free`. One of those two views was lying.

## What I tried

Cross-checked the state file against the tick that wrote it (cron output `81a8bc241220/2026-09-18_19-32-52.md`): that run printed `fallback=nemotron-3-ultra-free`, `configured=nemotron-3-ultra-free`, `needfix=no` — so the configured fallback was healthy, and the `mimo` strike was a **fossil** from the earlier wiring (mimo was the fallback on 09-11, then superseded). The code confirmed the mechanism: `cfalls` is only incremented/popped for `cfalls[configured]`; a re-wire to a different model never prunes the old model's key, and `if changed or cfalls: save_state(...)` writes it forever. I then ran the monitor out-of-band to observe the behavior live.

## What worked

Running the real monitor and diffing the state file before/after (catalog lines elided):

```bash
$ cat ~/.hermes/scripts/go_fallback_state.json      # written by the 19:32 tick
{"bans": {}, "cfalls": {"mimo-v2.5-free": 1}}
# ^ but that tick printed configured=nemotron-3-ultra-free (healthy) — stale entry

$ python3 ~/.hermes/scripts/adaptive_monitor.py      # out-of-band tick
fallback=ALL-DEAD
configured=nemotron-3-ultra-free
needfix=no
free=deepseek-v4-flash-free;ling-3.0-flash-fin-free;mimo-v2.5-free;...   # elided
paid38=...   # elided (included union-alpha, new model)
zen70=...    # elided
exit=0

$ cat ~/.hermes/scripts/go_fallback_state.json      # after
{"bans": {}, "cfalls": {"mimo-v2.5-free": 1, "nemotron-3-ultra-free": 1}}
```

Two live observations: (1) the fresh strike landed on the **configured** model (`ultra: 1`) while the orphaned `mimo: 1` survived untouched; (2) `fallback=ALL-DEAD` vs the healthy `fallback=ultra` five minutes earlier — a single probe is flaky, which is exactly why the threshold is 2 consecutive ticks. I restored the state file afterwards (my run was diagnostic, not a scheduled tick).

## Takeaway

In a self-healing monitor's state file, only the counter for the *currently configured* model is live signal — every other `cfalls` key is stale archaeology; if you ever re-wire the fallback, pop the old model's strike counter in the same write, or the state file will mislead the next reader.