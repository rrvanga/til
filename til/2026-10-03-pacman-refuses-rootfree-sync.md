# Pacman v7 refuses a root-free `-Sy` even with a private `--dbpath` — unprivileged update checks are local-DB-only now

**Date:** 2026-10-03
**Tags:** `arch`, `pacman`, `updates`, `unprivileged`, `verification`

## The problem

A daily health probe runs `checkupdates` (the classic root-free way to list pending Arch updates) — and it cannot work on this host, for two independent reasons:

1. `checkupdates` is not part of core pacman — it ships in **pacman-contrib**, which isn't installed here (`pacman -Q pacman-contrib` → `not found`; `command -v checkupdates` → nothing).
2. Even with it installed, its core trick — copy the sync DBs to a user-writable temp dir and sync *there* — is dead on modern pacman.

## What I tried

Replicated the checkupdates workflow without root: copy `/var/lib/pacman/sync/*.db` into a `mktemp -d` dir, add an empty `lck` file, then:

```bash
pacman -Sy --dbpath "$TMP" --logfile /dev/null
```

Result on pacman **v7.1.0** (`pacman --version` → `Pacman v7.1.0`):

```
error: you cannot perform this operation unless you are root.
```

`-Sy` now hard-requires root regardless of `--dbpath`. The temp dir only ever contained the *copied* old DBs (their mtimes came from `cp`, not from any sync), so the follow-up `pacman -Qu --dbpath "$TMP"` printed 0 — a **false green**, not a fresh verdict.

## What actually works without root

- `paru -Qu` / `yay -Qu` — read-only, works fine, and is truthful for **AUR** packages (live AUR RPC): on this box it reported `google-chrome 144 → 154`, `yay 12.5.7 → 13.0.1`, `accounts-qml-module 0.7-7 → 0.7-8`.
- For **official-repo** packages, `-Qu` reads the local sync DBs and inherits their staleness — gate on DB mtimes first (`stat -c '%y' /var/lib/pacman/sync/*.db`; here they were 8 days old). See the 2026-09-11 `stale-pacman-sync-db` entry: an empty result on stale DBs is meaningless.
- The only true verdict is root `sudo pacman -Syu`.

## Takeaway

"Use `checkupdates`" is not portable advice on 2026-era Arch: the binary ships separately (pacman-contrib) and the root-free temp-DB sync it relied on is blocked on pacman ≥7. An unattended update probe should: check `paru -Qu` for AUR, verify sync-DB freshness for repos, and escalate "DB stale / updates pending" to a human to run `sudo pacman -Syu` — never present a 0-count from a stale or unsynced DB as "all green".