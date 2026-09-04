# A midnight-spanning night-window predicate admits daytime when start ≤ end — validate the shape, not just the format

**Date:** 2026-09-04
**Tags:** `bash`, `time-handling`, `fail-closed`

## The problem

`nightly-shutdown.sh` (agent-lab, issue #19) decides the night window with `[ "$now" -ge "$start" ] || [ "$now" -lt "$end" ]` — an OR union that is only correct when the window spans midnight (start > end). A config like `NIGHT_START=12:00 NIGHT_END=00:00` passed the old checks yet put every hour from noon onward "in window": the box was eligible for a daytime poweroff at 13:00.

## What I tried

Reading the predicate alone didn't show the failure mode, so I rebuilt its exact logic in a 20-line harness and drove `now` across the day. I also probed the shell-arithmetic corner around the times themselves: `$((0800))` for 08:00 errors out because a leading-zero literal is parsed as octal in `$(( ))`.

## What worked

Real harness output from this session:

```bash
$ bash /tmp/til_demo.sh
== 1) predicate with START=12:00 END=00:00 (start <= end) ==
  1300 IN window
  1700 IN window
  2359 IN window
== 2) octal trap ==
  $((10#${NIGHT_START//:/})) -> 800
/tmp/til_demo.sh: line 18: 0800: value too great for base (error token is "0800")
== 3) valid_hhmm guard ==
  23:59 OK
  20:00 OK
  03:7 REJECTED
  25:00 REJECTED
  08:00 OK
```

The fix (agent-lab PR #20, squash `49d754c`) closes the hole fail-closed at config load: a strict `valid_hhmm()` `case` guard (`[0-2][0-9]:[0-5][0-9]`, then numeric `10#` bounds ≤ 23:59), a requirement that start > end (the window must span midnight), and a floor of start ≥ 20:00 so an early start like noon can never admit daytime hours. Any invalid config exits 2 before a poweroff path can run; harness test T10 sabotages the start bound to prove the regression actually bites.

## Takeaway

A "spans midnight" window predicate is a *shape* (start > end), not a range — validate start > end and a sane start floor before trusting an OR union, and always parse HH:MM with `10#` so `08:00` doesn't blow up as octal.