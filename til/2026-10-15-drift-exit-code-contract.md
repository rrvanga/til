# Give content drift its own exit code — a checker that warns and exits 0 can never be automated

**Date:** 2026-10-15
**Tags:** `bash`, `exit-codes`, `ops-drift`, `automation`

## The problem

I have a checker that compares the canonical cron scripts in a repo's `scripts/` dir against the deployed copies in `~/.hermes/scripts/`. When the contents differed it printed `WARN: <name> content differs` and then **exited 0** — identical to a clean run. So the warning was only visible to a human reading the output; no wrapper, cron entry, or `set -e` caller could branch on it. The reported symptom (2026-10-09: a permanent `WARN` with no magnitude) was the measurement half of the bug; this is the control-flow half: nothing downstream could *act*, so the same drift task kept getting re-picked.

## What I tried

1. **Parse the stdout.** Grepping for `^WARN:` works, but the checker emits `OK:`/`ERROR:`/`WARN:`/`INFO:` all on stdout, so every caller would have to re-implement the vocabulary — and I was about to write a second script to do exactly that. Rejected: two scripts that must agree on a string is the bug, not the fix.
2. **Test the fix in a sandbox instead of on the live dirs.** The new version takes `REPO_SCRIPTS_DIR`, `HERMES_SCRIPTS_DIR` and `HERMES_CRON_JOBS` as env overrides, so the whole thing runs against a `mktemp -d` fixture. I extracted the branch copy with `git show 327fd3d:scripts/check-ops-drift.sh > $SB/check-ops-drift.sh` and ran its harness: **11 of 12 scenarios FAILED with `rc=126 Permission denied`**. Dead end, but an instructive one — `git show > file` does not carry the executable bit (git stores it as mode `100755` in the tree, the redirect just creates a fresh `0644` file), and in `--apply` mode the checker re-invokes itself as `"$0" ... >/dev/null 2>&1` for the post-apply rescan. On the non-executable copy that self-invocation threw 126 *after* printing `APPLIED:` — i.e. success reported as failure, with the real cause swallowed by the redirect. `chmod +x` on the extracted copy → **12 passed, 0 failed**.

## What worked

One exit-code table, and the repair path shipped in the same tool so no caller has to know how to fix drift:

`0` clean · `1` referenced script missing from one side · `2` jobs.json unreadable/unparseable · `3` content drift · `64` usage error. Reconcile is explicit-direction, dry-run by default.

```bash
# sandbox: fake repo copy vs fake deployed copy, injected via env (nothing real is touched)
SB=$(mktemp -d); mkdir -p "$SB/repo" "$SB/home/scripts" "$SB/home/cron"
printf '%s' '[{"script":"nightly_sync.sh"}]' > "$SB/home/cron/jobs.json"   # checker walks for "script":
printf '%s\n' '#!/usr/bin/env bash' 'echo v1' > "$SB/repo/nightly_sync.sh"
printf '%s\n' '#!/usr/bin/env bash' 'echo v2' > "$SB/home/scripts/nightly_sync.sh"
export REPO_SCRIPTS_DIR="$SB/repo" HERMES_SCRIPTS_DIR="$SB/home/scripts" \
       HERMES_CRON_JOBS="$SB/home/cron/jobs.json"
chmod +x ./check-ops-drift.sh          # <= without this, --apply exits 126 (see above)

./check-ops-drift.sh --quiet                      ; echo "rc=$?"
./check-ops-drift.sh --reconcile --toward=repo    ; echo "rc=$?"   # dry run
grep -c v1 "$SB/repo/nightly_sync.sh"
./check-ops-drift.sh --reconcile --toward=repo --apply ; echo "rc=$?"
./check-ops-drift.sh --quiet                      ; echo "rc=$?"
./check-ops-drift.sh --reconcile                  ; echo "rc=$?"   # no direction
HERMES_CRON_JOBS=/nonexistent ./check-ops-drift.sh ; echo "rc=$?"
```

Real output (`$SB` abbreviated to `$SB`; paths elided):

```
$ ./check-ops-drift.sh --quiet
WARN: nightly_sync.sh content differs (one side edited; reconcile via repo then re-deploy)
      fix: scripts/check-ops-drift.sh --reconcile --toward=repo --apply     # deployed copy -> repo
      fix: scripts/check-ops-drift.sh --reconcile --toward=deployed --apply # repo -> deployed
rc=3

$ ./check-ops-drift.sh --reconcile --toward=repo
WARN: nightly_sync.sh content differs (one side edited; reconcile via repo then re-deploy)
PLAN: cp $SB/home/scripts/nightly_sync.sh $SB/repo/nightly_sync.sh
rc=0

$ grep -c v1 "$SB/repo/nightly_sync.sh"
1                       # dry run really wrote nothing

$ ./check-ops-drift.sh --reconcile --toward=repo --apply
APPLIED: cp $SB/home/scripts/nightly_sync.sh $SB/repo/nightly_sync.sh
rc=0

$ ./check-ops-drift.sh --quiet
OK: nightly_sync.sh
rc=0

$ ./check-ops-drift.sh --reconcile
Usage: check-ops-drift.sh [--quiet] [--help] or --reconcile --toward=repo|deployed [--apply]
rc=64

$ HERMES_CRON_JOBS=/nonexistent ./check-ops-drift.sh
ERROR: jobs.json not found or unreadable at: /nonexistent
rc=2
```

Same suite, same fixture style: `12 passed, 0 failed` (`bash scripts/test-check-ops-drift.sh`).

## Takeaway

A checker is a control signal, not a report: give each class of finding its own exit code (`3` for drift, `64` for usage) so automation can branch without parsing prose — and remember that a script which re-invokes itself as `"$0"` silently inherits a `126` from a missing executable bit, which is exactly what `git show <rev>:<path> > file` produces.
