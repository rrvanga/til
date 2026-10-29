# A sync tool that writes *into* a git worktree must refuse on a dirty tree — `cp` leaves nothing for the reflog to recover

**Date:** 2026-10-29
**Tags:** `bash`, `git`, `ops-drift`, `fail-closed`

## The problem

`check-ops-drift.sh` grew a reconcile mode that copies a drifted deployed script **back into the repo** (`--reconcile --toward=repo --apply`) so the canonical copy is fixed in one step. The write target is a git worktree, and `cp` doesn't know what uncommitted state is: any half-finished local edit sitting at the destination is silently replaced by the deployed bytes, and because it was never committed there is no reflog, no stash, nothing to recover. Worse, the tool then re-scans and prints a clean bill of health — the evidence of the loss is erased along with the loss. The chicken-and-egg part: the one thing the guard must not do is turn a *no-op* run into a failure, and a `--force` escape hatch is genuinely needed for rescue cases, but a flag whose only purpose is to skip a safety check will get pasted everywhere unless the parser refuses it in every other mode.

## What I tried

1. **Trust `cmp` and the operator.** That's what the tool did before: the pre-scan already knows the file differs and already prints "one side edited". Fine for reading, useless as protection — the diff is visible only on the *deployed* side; the side about to be clobbered is invisible to the operator in that output.
2. **Refuse whenever `git status --porcelain` is non-empty, in both directions.** Rejected: `--toward=deployed` writes to a non-versioned dir whose source of truth *is* the repo. The repo is the recovery path there, so a dirty tree is irrelevant (or even desirable — that's how you preview a change before committing).
3. **Hard-fail when the directory isn't a git work tree.** Rejected: the checker is also used against a deployed dir with no `.git` anywhere up the tree; `git status` exits 128 there, which must be a WARN, not a dead end.

Placed the check **after** the drift scan and after the early `DRIFT_COUNT -eq 0 → exit 0` branch, so it costs one `git` call only when there is actually something to copy, and it never fails a run that was going to do nothing anyway.

## What worked

Gate the copy loop on a clean worktree, only for the direction that writes into the repo; `WARN`-and-continue when the target isn't a work tree; and reject `--force` with exit `64` unless it is *exactly* `--reconcile --toward=repo --apply`:

```bash
if [ "$TOWARD" = "repo" ] && [ "$APPLY" -eq 1 ]; then
  GIT_STATUS="$(git -C "$REPO_RESOLVED" status --porcelain 2>/dev/null)"
  GIT_RC=$?
  if [ "$GIT_RC" -ne 0 ]; then
    echo "WARN: cannot verify repo worktree cleanliness (not a git work tree): $REPO_RESOLVED" >&2
  elif [ -n "$GIT_STATUS" ]; then
    if [ "$FORCE" -eq 1 ]; then
      echo "WARN: repo worktree is dirty: $REPO_RESOLVED (continuing because --force was given)" >&2
    else
      echo "ERROR: repo worktree is dirty: $REPO_RESOLVED; commit or stash first, or pass --force" >&2
      exit 1
    fi
  fi
fi
```

Demonstrated end-to-end against a throwaway git repo + fake deployed dir, using the tool's env overrides (`REPO_SCRIPTS_DIR`, `HERMES_SCRIPTS_DIR`, `HERMES_CRON_JOBS`) so nothing live was touched:

```bash
P=$(mktemp -d); mkdir -p "$P/repo/scripts" "$P/deployed"; cd "$P/repo"
printf '#!/usr/bin/env bash\necho deployed-canonical\n' > scripts/demo.sh
git init -q . && git add -A && git commit -qm init
printf '#!/usr/bin/env bash\necho drifted-deployed-copy\n' > "$P/deployed/demo.sh"
printf '{"jobs":[{"script":"demo.sh"}]}\n' > "$P/jobs.json"
export REPO_SCRIPTS_DIR="$P/repo/scripts" HERMES_SCRIPTS_DIR="$P/deployed" HERMES_CRON_JOBS="$P/jobs.json"

# 1. uncommitted local edit at the destination -> must refuse, must not copy
printf '#!/usr/bin/env bash\necho half-finished-edit-never-reviewed\n' > scripts/demo.sh
check-ops-drift.sh --reconcile --toward=repo --apply ; echo "rc=$?"
sha256sum "$P/deployed/demo.sh"          # unchanged == nothing was written either way

# 2. committed edit -> the promotion is now reviewable, so exec it
git commit -qam "reviewed edit"
check-ops-drift.sh --quiet --reconcile --toward=repo --apply ; echo "rc=$?"
cmp -s "$P/deployed/demo.sh" scripts/demo.sh && echo "deployed == repo"

# 3. the deliberate override, and the flag's blast-radius limit
check-ops-drift.sh --quiet --reconcile --toward=repo --apply --force ; echo "rc=$?"
check-ops-drift.sh --force ; echo "rc=$?"
```

Real output (`$P` abbreviated; fix-hint lines elided):

```
$ check-ops-drift.sh --reconcile --toward=repo --apply
WARN: demo.sh content differs (one side edited; reconcile via repo then re-deploy)
      fix: ... --reconcile --toward=repo --apply     # deployed copy -> repo
      fix: ... --reconcile --toward=deployed --apply # repo -> deployed
ERROR: repo worktree is dirty: $P/repo/scripts; commit or stash first, or pass --force
rc=1
$ sha256sum "$P/deployed/demo.sh"
1c865aaa4b987316...        # identical before and after: the refusal copied nothing

$ check-ops-drift.sh --quiet --reconcile --toward=repo --apply   # after committing
WARN: demo.sh content differs (one side edited; reconcile via repo then re-deploy)
APPLIED: cp $P/deployed/demo.sh $P/repo/scripts/demo.sh
rc=0
deployed == repo

$ check-ops-drift.sh --quiet --reconcile --toward=repo --apply --force
WARN: repo worktree is dirty: $P/repo/scripts (continuing because --force was given)
APPLIED: cp $P/deployed/demo.sh $P/repo/scripts/demo.sh
rc=0

$ check-ops-drift.sh --force
Usage: check-ops-drift.sh [--quiet] [--help] or --reconcile --toward=repo|deployed [--apply]
rc=64
```

Proof that the override is genuinely destructive — the uncommitted line I had in the worktree is gone from disk *and* from git, and the tool's own write leaves the worktree dirty relative to `HEAD`:

```
$ git status --porcelain
 M scripts/demo.sh
$ git show HEAD:scripts/demo.sh | tail -1
echo half-finished-edit-never-reviewed
$ tail -1 scripts/demo.sh
echo drifted-deployed-copy
$ grep -rl forced-through . || echo "gone: not on disk, not in git"
gone: not on disk, not in git
```

Harness went 18 → 24 assertions, 0 failed (`bash scripts/test-check-ops-drift.sh`), including "refused apply leaves the drifted file byte-unchanged" and both `--force` usage errors.

## Takeaway

Put the dirty-worktree veto on the direction that writes *into* version control, not on the tool: refuse with exit 1 when `git status --porcelain` is non-empty, `WARN`-and-continue when the target isn't a work tree at all, and let `--force` exist only for the one invocation that is otherwise unrecoverable — because a copy into a worktree destroys uncommitted work that neither `git reflog` nor `stash` can bring back.
