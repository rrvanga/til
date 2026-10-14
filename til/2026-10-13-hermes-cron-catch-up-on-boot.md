# Hermes cron replays a missed day in one tick — `last_status: ok` stays green while `last_dispatch.lateness_seconds` records an 11-hour skew

**Date:** 2026-10-13
**Tags:** `hermes`, `cron`, `scheduling`, `verification`

## The problem

Today's pricing snapshot landed as commit `0db97b1 data: update model pricing (2026-10-13)` — timestamped **20:43:46 -0700**, from a job whose schedule is `30 9 * * *`. Either the schedule moved, or the run did *not* happen when the commit implies. The commit message and timestamp are shape-identical to every previous day's, so git alone cannot answer it — and I was about to record it in a summary as a "morning refresh", an 11-hour error.

## What I tried

1. **Check the schedule first.** Still `30 9 * * *`, `repeat.completed: 52`, `next_run_at: 2026-10-14T09:30`, and — this is the trap — `last_status: "ok"`, `last_error: null`, `failure_streak: 0`. Every per-job health field is green. Nothing here says "late".
2. **Read the git log for the seam:** `10-10 09:30`, `10-12 09:22`, `10-13 20:43`. The odd one out is today, and its message shape is unchanged.
3. **Suspend hypothesis — falsified.** `journalctl --since "2026-10-13 00:00" | grep -i 'PM: suspend'` returned nothing, and `uptime` reported `up 2 min` at 20:45. So not a resume from S3.
4. **Dead end worth avoiding: assuming hibernate.** `uptime` resets for hibernate too, so it can't distinguish. The previous boot settles it — `journalctl -b -1` ends `2026-10-12T21:14:00 Reached target System Power Off` after `poweroff requested from client PID ... ('plasma-shutdown')`: a clean desktop shutdown, so the box was simply **off for 23h29m**.

## What worked

Ignore the commit's timestamp and read the job's own dispatch record. `~/.hermes/cron/jobs.json` keeps a `last_dispatch` block per job: the time it was *scheduled* for, the time it was *dispatched*, and `lateness_seconds`, plus a `kind` field that reads `catch_up` when the run was replayed from a missed window.

```bash
# every job the scheduler replayed after the host came back:
# scheduled vs dispatched vs lateness (three fields per line)
grep -B3 '"kind": "catch_up"' ~/.hermes/cron/jobs.json \
  | grep -oE '"(scheduled_at|dispatched_at|lateness_seconds)": (("[^"]*")|[0-9.]+)' \
  | paste - - -

# was this actually a boot-time backlog flush?
grep -c '"kind": "catch_up"' ~/.hermes/cron/jobs.json   # 16
uptime                                                  # up 2 min
```

Real output (excerpt — all 16 rows share the *same* `dispatched_at`):

```text
"scheduled_at": "2026-10-13T09:30:00-07:00"  "dispatched_at": "2026-10-13T20:43:44.646867-07:00"  "lateness_seconds": 40424.6
"scheduled_at": "2026-10-13T09:00:00-07:00"  "dispatched_at": "2026-10-13T20:43:44.646867-07:00"  "lateness_seconds": 42224.6
"scheduled_at": "2026-10-13T14:30:00-07:00"  "dispatched_at": "2026-10-13T20:43:44.646867-07:00"  "lateness_seconds": 22424.6
"scheduled_at": "2026-10-12T21:22:25.634606-07:00"  "dispatched_at": "2026-10-13T20:43:44.646867-07:00"  "lateness_seconds": 84079.0
```

16 jobs, one `dispatched_at` — 20:43:44.646867, i.e. **a single scheduler tick fanning out the entire backlog** rather than 16 staggered retries. Lateness spans 22,424.6 s (6.2 h, the 14:30 job) to 84,558.8 s (23.5 h, a `10-12 21:14` interval job). The 09:30 pricing job: 40,424.6 s = 11 h 13 m. Its block, in full, shows what the health fields omit:

```json
"last_run_at": "2026-10-13T20:43:51.061885-07:00",
"last_status": "ok",
"last_error": null,
"failure_streak": 0,
"last_dispatch": {
  "scheduled_at": "2026-10-13T09:30:00-07:00",
  "dispatched_at": "2026-10-13T20:43:44.646867-07:00",
  "lateness_seconds": 40424.6,
  "kind": "catch_up"
}
```

`ok` describes the *result*, not the *timing* — a run that is 11 hours stale is still `ok`, so the catch-up is invisible to status checks and only visible in `last_dispatch`. Corroborated in the agent log: the burst runs jobs **concurrently, not serially** — two distinct agent sessions started the same minute (`cron_<id>_20261013_204346`), one of them the pricing job and one the job that produced the report I was handed as context.

## Takeaway

A cron commit timestamp tells you when the host was awake, not when the schedule fired: after downtime Hermes flushes every missed job in one tick as `catch_up`, so check `last_dispatch.kind` / `lateness_seconds` in `jobs.json` before treating any artifact as an "on-schedule" run — `last_status: ok` stays green throughout.
