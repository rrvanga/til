# A background job writes its real verdict after the wrapper reports "exited" — verify with pgrep first

**Date:** 2026-09-07
**Tags:** `background-processes`, `verification`, `automation`, `bash`

## The problem

The MOA review gate (`hermes chat ... -m moa:default &`) twice "failed silently": the terminal wrapper reported the background command as exited with a 0-byte output file within ~10–40s. Turns out the wrapper was lying — the orphaned child kept running and wrote the full verdict later. Trusting the wrapper's status meant we nearly relaunched the gate (or worse, declared the review dead) while the real verdict was still on its way.

## What I tried

- Restarting the review on the wrapper's early "exited"/0-byte status → wasted runs.
- Assuming the output file's size at check time was final → wrong, verdict landed seconds later.

## What worked

Verify liveness (`kill -0` / `pgrep`) before believing any wrapper status, then wait for the complete output file — the file is the source of truth, not the wrapper's exit report. Demonstrated with a 2-second sleeper:

```bash
$ bash /tmp/orphan_demo.sh
Foreground: launched bg pid 22050; out file = 0 bytes
kill -0: process STILL RUNNING (wrapper 'exited' would be a false signal)
    PID STAT     ELAPSED CMD
  22050 S          00:00 bash /tmp/orphan_demo.sh
After wait: out file = 33 bytes
full verdict written at 19:41:29
```

The 0-byte file at "launch+0s" looks exactly like the wrapper's false "exited" signal — but `kill -0` proved the process was alive, and waiting produced the full 33-byte verdict.

## Takeaway

Never trust a wrapper's "exited" status or a momentarily-empty output file for a background job that may keep writing — `pgrep` first, then poll/`wait` for the complete verdict before acting on it.