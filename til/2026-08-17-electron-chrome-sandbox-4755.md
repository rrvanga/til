# Electron's chrome-sandbox must be setuid root:root 4755 — gate on file state, not on sudo

**Date:** 2026-08-17
**Tags:** `electron`, `sandbox`, `systemd`, `permissions`, `linux`

## The problem

Chromium/Electron refuse to start unless `chrome-sandbox` is owned by `root:root` with mode `4755` (setuid). Fixing it needs `sudo` — but on this host sudo requires a password, and a systemd **user** unit (or unattended update script) has no TTY to answer the prompt. The unit fails mid-update and lands in `failed` state; the app stays broken and there's no automatic retry.

## What I tried

First, the naive version — just put the fix in the unit:

```bash
# ExecStartPre in a systemd user unit — fails: no TTY for the sudo prompt
sudo chown root:root /usr/lib/chromium/chrome-sandbox && sudo chmod 4755 /usr/lib/chromium/chrome-sandbox
```

Confirmed the premise on this box: `sudo -n true` fails instantly when no passwordless sudo is configured — and a systemd user unit is exactly that environment.

```
$ sudo -n true 2>&1; echo "exit=$?"
sudo: a password is required
exit=1
```

The unconditional `sudo` version was worse: it prompted (or failed) on *every* run, even when the file was already correct — so updates started failing on a machine whose sandbox had never been broken.

## What worked

Gate on the **file's actual state**, and only invoke sudo when the state is wrong. `stat -c '%U:%G:%a'` reports `owner:group:mode` in one call; compare it against the required `root:root:4755`:

```bash
for SB in /usr/lib/chromium/chrome-sandbox /opt/google/chrome/chrome-sandbox; do
  if [ "$(stat -c '%U:%G:%a' "$SB")" = "root:root:4755" ]; then
    echo "OK   $SB  -> $(stat -c '%A %U:%G' "$SB")  (skip: no sudo needed)"
  else
    echo "FIX  $SB  -> sudo chown root:root $SB && sudo chmod 4755 $SB"
  fi
done
```

Real output on this host (both installed sandboxes already correct → zero sudo calls):

```
OK   /usr/lib/chromium/chrome-sandbox  -> -rwsr-xr-x root:root  (skip: no sudo needed)
OK   /opt/google/chrome/chrome-sandbox  -> -rwsr-xr-x root:root  (skip: no sudo needed)
```

When a unit has already failed, clear the marker once the file is fixed — otherwise it stays flagged `failed` forever:

```
$ SYSTEMD_PAGER= systemctl --user reset-failed
reset-failed: exit=0 (clears failed-state markers, harmless if none)
```

## Takeaway

If a fix needs sudo on a host without passwordless sudo, never run it unconditionally and never from a TTY-less systemd unit — gate the sudo call on file state (`stat ... = root:root:4755`, skip if true), and `systemctl --user reset-failed` the unit afterwards instead of leaving it in a permanent failed state.