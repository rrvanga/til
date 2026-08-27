# An auto-refresh cron job commits to whatever branch is checked out — not always main

**Date:** 2026-08-26
**Tags:** `git`, `automation`, `cron`

## The problem

The agent-lab engineering loop finished 2026-08-24 with a handoff: "fix the doc, re-run the review gate, merge PR #14". Two days later, PR #14 is still open, `main` hasn't moved, the branch has an unexpected extra commit — and the automatic *"docs: refresh architecture diagram (auto)"* job, which normally lands on `main`, appears on the feature branch instead. Where did the auto commit go, and why?

## What I tried

Checked the repo instead of trusting the handoff notes: `git status -sb` for branch drift, `git log --graph --oneline --all` for where commits actually live, and `git reflog` for what really happened after the handoff.

## What worked

The commit timestamps and parentage told the whole story:

```bash
$ git status -sb | head -1
## feat/update-policy...origin/feat/update-policy [ahead 1]

$ git log -3 --format='%h %ci %d %s'
f0a6220 2026-08-24 11:00:55 -0700  (HEAD -> feat/update-policy) docs: refresh architecture diagram (auto)
e1c50c6 2026-08-24 09:12:04 -0700  (origin/feat/update-policy) docs: add Hermes update policy runbook (issue #4)
ab74fa3 2026-08-22 11:00:33 -0700  (origin/main, origin/HEAD, main) docs: refresh architecture diagram (auto)
```

The 11:00 auto-refresh job committed `f0a6220` *on top of the feature branch* (its parent is `e1c50c6`, the PR commit), not on `main` — because the repo had been left checked out on `feat/update-policy`. A scheduled job that runs `git add -A && git commit` commits to **whatever branch is checked out**; it doesn't know or care that its siblings usually land on `main`. The result: the branch is `[ahead 1]` of origin (the refresh was never pushed), `main` still lacks the refresh, PR #14's diff silently picked up an unrelated diagram change — and the reflog confirmed the handoff's planned amend/rebase/merge never executed at all. The same worktree was also left mid-state (4 unstaged deletions, 2 untracked notes files).

## Takeaway

Before leaving a repo to scheduled jobs that commit, switch it to `main` (or give each job its own `git worktree`) — and when a handoff says "amend and merge", trust the reflog and the PR state, not the notes: an unchanged commit hash and an open PR mean the fix never ran.