# archlinux-keyring-wkd-sync's exit code is its error count — a run colliding with suspend "fails" with 101 yet needs zero action

**Date:** 2026-10-07
**Tags:** `arch`, `systemd`, `gpg`, `suspend`, `exit-codes`, `verification`

## The problem

`journalctl -p err` flagged `archlinux-keyring-wkd-sync.service` as failing on 2026-10-06 18:51: `Main process exited, code=exited, status=101/n/a` → `Failed with result 'exit-code'`. With the 07:00/07:30 ops reports watching everything, a keyring sync unit red on the error log looked like something to chase — including a possibly stale `pacman` sync DB.

## What I tried

1. Read the unit: `Type=oneshot` with `Restart=on-failure`, `RestartSec=5minutes`, `StartLimitBurst=3` — so a failure is *designed* to retry.
2. Dumped the failing window and saw the whole thing hit in one second: the weekly timer fired **exactly as the machine entered S3**. The kernel log and the service log interleave at 18:51:23:
   - `gpg: error retrieving '<uid>' via WKD: Server indicated a failure` — ~101 times, because the WKD fetches died mid-suspend.
3. Counted the errors and read the script (`/usr/bin/archlinux-keyring-wkd-sync`, owned by `archlinux-keyring 20260909-1`) to find out what 101 meant. The "Skipping key …" lines looked alarming at first glance, but the script shows they are the **normal filtering branch** (UIDs not ending in `@archlinux.org`/`@master-key.archlinux.org`, or revoked/invalid-fingerprint keys) — not an error.

## What worked

The script ends with `exit ${#errors[@]}` — **the exit code IS the number of refresh errors**, so status 101 means exactly 101 failed WKD fetches, no more, no less:

```bash
# suspend and service start collide in the same second
$ journalctl --since "2026-10-06 18:51" --until "2026-10-06 18:52" --no-pager | grep -E "Preparing to enter system sleep state S3|Starting Refresh"
Oct 06 18:51:23 slick kernel: ACPI: PM: Preparing to enter system sleep state S3
Oct 06 18:51:23 slick systemd[1]: Starting Refresh existing keys of archlinux-keyring...

# exit code = error count, verifiable in the journal
$ journalctl -u archlinux-keyring-wkd-sync.service --since "2026-10-06 18:51:20" --until "2026-10-06 18:53:00" --no-pager | grep -cE "gpg: error retrieving"
101
$ journalctl -u archlinux-keyring-wkd-sync.service --since "2026-10-06 18:51:20" --until "2026-10-06 18:58:30" --no-pager | grep -E "Main process exited|Scheduled restart|Deactivated"
Oct 06 18:51:23 slick systemd[1]: archlinux-keyring-wkd-sync.service: Main process exited, code=exited, status=101/n/a
Oct 06 18:56:23 slick systemd[1]: archlinux-keyring-wkd-sync.service: Scheduled restart job, restart counter is at 1.
Oct 06 18:57:51 slick systemd[1]: archlinux-keyring-wkd-sync.service: Deactivated successfully.
```

`RestartSec=5minutes` retried after wake and the run finished clean (`Deactivated successfully`) — the "failure" was entirely a suspend artifact, self-healed on the retry. The next scheduled fire is visible in `systemctl list-timers` (weekly, `Persistent=true` catches up).

## Takeaway

Read the service script before chasing a systemd failure: some units deliberately encode a count as the exit status (here `exit ${#errors[@]}`), and a media-suspend collision is a benign, self-healing transient — distinguish it from genuinely stale states (like a root `pacman -Syu` still pending) by checking whether the retry succeeded and what the error count actually was.