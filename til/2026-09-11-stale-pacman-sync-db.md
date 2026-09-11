# `pacman -Qu` returning empty doesn't mean you're up to date — check the sync DB age first

**Date:** 2026-09-11
**Tags:** `arch`, `pacman`, `updates`, `verification`, `read-only-checks`

## The problem

A monthly system-health pass ran `pacman -Qu` and got zero pending updates, which read as "all good". A rollback-release distro a month past its last repo sync reporting zero updates is suspicious — Arch moves fast, so a month of releases should show plenty of pending packages.

## What I tried

- Ran `pacman -Qu` again → still empty. On its own that output is meaningless: `-Qu` only compares installed package versions against the **locally mirrored sync DBs** (`/var/lib/pacman/sync/*.db`), not the live repos. Stale DB → empty result even when the real repositories are thousands of commits ahead.
- Checked the DB freshness instead — read-only, no root needed:

```bash
stat -c '%y %n' /var/lib/pacman/sync/*.db
# 2026-08-11 14:37:24 ... /var/lib/pacman/sync/core.db
# 2026-08-11 16:15:32 ... /var/lib/pacman/sync/extra.db
# 2026-08-11 11:48:01 ... /var/lib/pacman/sync/multilib.db
```

One month old. Equivalent one-liner for "older than N days":

```bash
find /var/lib/pacman/sync -name '*.db' -mtime +7
```

## What worked

The `stat` mtime check is the ground truth: `pacman -Qu` output is only meaningful when the sync DBs are recent, and the empty output had been masking a month of unapplied updates (kernel, glibc, mesa, browsers — whatever landed since Aug 11).

Fix after a stale DB is `sudo pacman -Syu`, **not** `sudo pacman -Sy`: syncing only the DB and then installing is a partial upgrade which can break package dependencies (e.g. new lib versions with old consumers). One `-Syu` refreshes the DB *and* upgrades everything in a single pass.

## Takeaway

Treat `pacman -Qu` (or `checkupdates`) results as trustworthy only after confirming the sync DBs are fresh — `stat -c '%y' /var/lib/pacman/sync/*.db` is the cheap read-only check to run first. An empty update list on a rolling-release distro is the classic false-green: the check passed, the system isn't.