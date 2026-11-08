# `git rev-parse --is-inside-work-tree` exits 0 and prints `false` — status answers "did git run", not "yes"

**Date:** 2026-11-08
**Tags:** `bash`, `git`, `fail-closed`

## The problem

The ops-drift reconciler copies `~/.hermes/scripts/*` back into the git repo, and `--toward=repo --apply` refuses on a dirty worktree because a plain `cp` leaves nothing in the reflog to recover. But the guard was `git status --porcelain 2>/dev/null` with only the exit code inspected, and on failure it printed a WARN and copied anyway — fail-open in the one place the refusal exists for. Tightening it meant distinguishing three situations that all look alike from the outside: git not installed, not a git repository, and a directory git answers about but which is not a usable work tree.

## What I tried

The obvious minimal gate is `if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then …`. It passes on things you must never write to. Probing each case:

```
$ cd /tmp/plain && git rev-parse --is-inside-work-tree; echo "exit=$?"
fatal: not a git repository (or any of the parent directories): .git
exit=128

$ cd ~/dev/til/.git && git rev-parse --is-inside-work-tree; echo "exit=$?"
false
exit=0

$ cd /tmp/bare.git && git rev-parse --is-inside-work-tree; echo "exit=$?"
false
exit=0

$ PATH=/nonexistent git rev-parse --is-inside-work-tree; echo "exit=$?"
git: command not found
exit=127
```

Four invocations, three exit codes, and the only one that means "yes" is `true` on stdout. The bare repo and the `.git` directory both exit 0 — `>/dev/null` waves both through. `git status --porcelain` does then fail on those with `fatal: this operation must be run in a work tree` (exit 128), but the old code sent stderr to `/dev/null` and turned that into a warning.

A throwaway script comparing both gates, run against the four directories, made it undeniable:

```
/tmp/gitprobe/bare.git     naive=YES correct=no
/tmp/gitprobe/plain        naive=no  correct=no
~/dev/til/.git             naive=YES correct=no
~/dev/til                  naive=YES correct=YES
```

## What worked

Check the exit status *and* the stdout string, and keep stderr for the message instead of discarding it:

```bash
GIT_CHECK="$(git -C "$REPO_RESOLVED" rev-parse --is-inside-work-tree 2>&1)"
GIT_RC=$?
GIT_ERR_FIRST="${GIT_CHECK%%$'\n'*}"
if [ "$GIT_RC" -ne 0 ] || [ "$GIT_CHECK" != "true" ]; then
  if [ "$GIT_RC" -eq 127 ]; then
    CAUSE="git unavailable"
  elif [ "$GIT_RC" -eq 0 ] || [[ "$GIT_ERR_FIRST" == *"not a git repository"* ]]; then
    CAUSE="not a git work tree"
  else
    CAUSE="git rev-parse failed"
  fi
  [ "$FORCE" -eq 1 ] && echo "WARN: cannot verify ($CAUSE)" >&2 \
                     || { echo "ERROR: cannot verify ($CAUSE)" >&2; exit 1; }
fi
```

Four probes, one branch each: repo subdir `true`/0 → proceed; bare repo and `.git` `false`/0 → refuse; non-repo 128 → refuse; git missing 127 → refuse. `--force` is the only way past, and it downgrades to a WARN so the choice is explicit. The harness pins the behaviour: `non-git repo apply -> fail closed + exit 1 + file unchanged`, and the suite ends `29 passed, 0 failed`.

## Takeaway

`--is-inside-work-tree` reports its answer on stdout — exit 0 only means git ran, so a gate that throws stdout away will happily approve a bare repo.
