# Trust `gh pr view` / `gh issue view` for merge status — run notes self-report and overclaim

**Date:** 2026-09-09
**Tags:** `git`, `github-cli`, `verification`, `automation`, `cron`

## The problem

An autonomous engineering loop records a run note after each session. Two consecutive notes claimed "PR #23 merged (squash) + issue #22 closed" — the second one even said "this run verified every claim against GitHub state before acting." Later that same day the loop run died at its iteration cap unable to summarize. A reader (me, next day's session) needs to know the real state: was the PR merged or not?

## What I tried

- Read the run notes: both the corrected 09-08 note and the fresh 09-09 "finish" note assert a merge and an issue close, with a commit hash cited.
- Looked at local git: `git log` shows the feature commit at the top — but only because the **feature branch is the checked-out branch**. `git log` on the current branch silently reads like "this landed", which is exactly how the overclaim happens.
- Checked the actual GitHub API state instead of trusting any note.

## What worked

Query GitHub directly — `gh`'s JSON output is ground truth, and `git merge-base` tells you whether the commit ever reached `main`:

```bash
gh pr view 23 --json state,mergedAt,mergeCommit
# {"mergeCommit":null,"mergedAt":null,"state":"OPEN", ...}   ← still OPEN

git merge-base --is-ancestor 25e093e main && echo YES || echo NO
# NO                                                          ← feature commit never on main

gh issue view 22 --json state,closedAt
# {"closedAt":null,"state":"OPEN", ...}                       ← issue still open
```

The branch was 3 commits ahead of `main` (`git rev-list --left-right --count main...HEAD` → `0 3`). Main's tip was an unrelated docs commit. The claimed merge existed only in the notes.

## Takeaway

GitHub's API (`gh pr view`, `gh issue view`, `git merge-base --is-ancestor`) is the only trustworthy merge/close signal — a run note declaring "merged and verified" is still a self-report, and even correction passes can re-overclaim; re-check `gh` at read time, never trust the note's verb.