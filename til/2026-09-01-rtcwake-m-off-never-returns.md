# rtcwake -m off never returns — arm with -m no, verify the alarm, then power off yourself

**Date:** 2026-09-01
**Tags:** `rtcwake`, `power-management`, `linux`, `bash`, `fail-closed`

## The problem

Building a guarded overnight shutdown script (power off at night, RTC-wake at 06:45),
the natural first draft was `rtcwake -m off -t <epoch>` followed by a
verify-then-poweroff chain: if the kernel didn't accept the alarm, the machine should
stay on. The review gate flagged that the verification lines after `-m off` were dead
code that could never run.

## What I tried

The first draft's logic looked sound:

```bash
rtcwake -m off -t "$epoch"     # set alarm AND power off
cat /sys/class/rtc/rtc0/wakealarm   # confirm kernel accepted it
# if empty -> abort, stay on
```

The problem is the mode itself. `rtcwake --list-modes` shows the available modes,
and the man page spells out what `off` actually does:

```
$ rtcwake --version
rtcwake from util-linux 2.42.2
$ rtcwake --list-modes
freeze mem disk off no on disable show
```

man rtcwake(8):

> **off** — ACPI state S5 (Poweroff). This is done by calling '/sbin/shutdown'...

`-m off` hands control to `/sbin/shutdown` internally and never returns — so
anything you write after it (the "verify, else stay on" guard) can never execute.
Verification after a poweroff-invoking call is a fail-open lie: the script *looks*
guarded but isn't. Checking `/proc/driver/rtc` confirms what an unarmed RTC looks
like (`alarm_IRQ: no`, `alrm_time: 00:00:00`).

## What worked

Split the one-shot call into three observable steps: arm **only**, verify, then power
off explicitly — the pattern now committed in `agent-lab` as
`scripts/nightly-shutdown.sh` (issue #5):

```bash
# 1. arm only — rtcwake -m no returns immediately
set_alarm() { rtcwake -m no -t "$epoch"; }

# 2. verify the kernel accepted it (non-empty wakealarm)
verify_alarm() { alarm=$(cat "$RTC_SYSFS/wakealarm"); [ -n "$alarm" ]; }

# 3. only after verification: power off explicitly
systemctl poweroff
```

man rtcwake(8) on the `no` mode: *"Don't suspend, only set the RTC wakeup time."*
It returns, so the arm result is actually checkable. The sysfs read-back is the
kernel's own acknowledgment — write, then read, then act.

## Takeaway

If a tool call can end the process mid-flight (`rtcwake -m off`, `shutdown`, `reboot`),
never put fail-safe logic after it — the process won't be alive to run it; split the
operation so each step returns and is verifiable before the irreversible one.