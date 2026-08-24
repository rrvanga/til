# Verify every error string and output example in AI-generated docs against the source

**Date:** 2026-08-24
**Tags:** `ai-agents`, `verification`, `documentation`, `grep`

## The problem

OpenCode drafted `docs/UPDATE.md` (a Hermes update runbook, PR #14) and seeded it with three plausible-looking but unverified details. A multi-model MOA review flagged them, but flagged them unevenly: two were genuine fabrications, one was a real output wrongly accused of being conjured. Only source-level checks could separate truth from plausible fiction — in both directions.

## What I tried

Ran the MOA review gate, then checked each finding against the actual tree and filesystem instead of trusting either the author or the reviewer:

1. The doc's error string `ERROR: update interrupted, working tree dirty` — grep the whole checkout for it.
2. The doc's backup example `hermes-backup-20260824_150000.tar.gz.gpg (63.1 MB, 14 kept)` — ls the real backup directory.
3. The doc's `→ Fetching from origin...` line, which the reviewer called invented — read the emitting source.

## What worked

```bash
$ grep -rn "update interrupted" ~/.hermes/hermes-agent --include="*.py" | wc -l
0                                   # invented: the error string exists nowhere

$ grep -rn "Fetching from origin" ~/.hermes/hermes-agent/hermes_cli/update_cmd.py
2998:            print("→ Fetching from origin...")   # no-upstream fallback path
3009:        print("→ Fetching from origin...")       # non-default branch path
                                    # real: reviewer's "invented" accusation was wrong

$ ls ~/.hermes-backups/  # 9 archives, not 14; largest is 70413898 bytes
hermes-backup-20260813_165305.tar.gz.gpg
...
hermes-backup-20260816_133014.tar.gz.gpg
                                    # invented: wrong name, wrong size, wrong count
```

The `→ Fetching from origin...` line turned out to be *exactly* the code path this install exercises (origin-only clone, no `upstream` remote → the `else` fallback at line 2998 runs), which is why it read as plausible. `grep`/`ls`/`read` settled all three: source beats both the generated text *and* the reviewer's guess.

## Takeaway

Treat every error string, output sample, and number in AI-generated docs as unverified until grep/ls/read proves it — and apply the same source check to reviewers' accusations, because generative plausibility cuts both ways.