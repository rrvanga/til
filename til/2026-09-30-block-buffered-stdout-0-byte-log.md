# A live background process can sit on a 0-byte log — block-buffered stdout flushes only at exit; check /proc, not output size

**Date:** 2026-09-30
**Tags:** `background-processes`, `buffering`, `verification`, `monitoring`

## The problem

A cron gate launch (background `hermes chat` with stdout redirected to a log file) looked "silently dead": the log stayed 0 bytes the whole run, so the run was written up as a false alarm. The process was alive the entire time — stdio had just never flushed. When stdout is not a TTY, C stdio switches from line-buffering to **full (block) buffering**: output sits in the ~4–8 KB buffer until it fills or the process exits.

## What I tried

Assuming 0 bytes meant death, and hunting for why the process "exited silently". Wrong direction — the log size was never a liveness signal. Reproduced it in isolation to confirm the mechanism.

## What worked

Check `/proc/<pid>` for liveness, never the log's byte count. Reproducing the trap:

```bash
python3 til_buffered_demo.py > til-buffered.log 2>&1 &
PID=$!
sleep 1
wc -c < til-buffered.log          # 0   <- output already printed, stuck in buffer
grep -E '^(Name|State)' /proc/$PID/status   # State: S (sleeping) -- ALIVE
[ -d /proc/$PID ] && echo ALIVE   # ALIVE
wait $PID
wc -c < til-buffered.log          # 105 <- both lines landed at process exit
```

Real output from the run:

```
== while RUNNING (stdout redirected to file, no -u) ==
log bytes: 0
state: Name:	python3 State:	S (sleeping)
liveness: /proc/357817 exists -> ALIVE (log still 0 bytes)
== after process EXIT ==
log bytes: 105
phase: start (this line sits in the stdio buffer)
phase: end (both lines flushed when the process exits)

== contrast: python3 -u, 1s in ==
log bytes: 50
```

The contrast line is the fix when you *do* want live logs: `python3 -u` (or `stdbuf -oL` for line-buffered, or `script`/pty) makes output land immediately. For a launcher that only cares whether the child died, `wait $PID` or a `/proc/$PID` poll is authoritative.

## Takeaway

An empty log from a redirected background process proves nothing — block-buffered stdout only flushes at exit; judge liveness by `/proc/<pid>` or `process wait`, and add `-u`/`stdbuf -oL` if you need the log to grow live.