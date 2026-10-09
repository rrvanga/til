# A `cmp -s` drift checker reports "differ" with no magnitude — one intentional deployed-only line reads as a permanent WARN

**Date:** 2026-10-09
**Tags:** `bash`, `verification`, `monitoring`

## The problem

Reconciling the Go quota-meter scripts (agent-lab #34) after a deploy, `check-ops-drift.sh` reported:

```
WARN: morning_context.py content differs (one side edited; reconcile via repo then re-deploy)
WARN: token_usage_report.py content differs (one side edited; reconcile via repo then re-deploy)
```

Those two WARNs are exactly what a wholesale stale deploy would produce. I needed to know whether the deployed copies were actually out of sync, or whether this was the one *known, intentional* deployed-only line (a live free-preview row we never committed).

## What I tried

1. Read the checker. The whole comparison is one line — `if cmp -s "$REPO_FILE" "$HERMES_FILE"` — so **any** byte difference, from a single added character to a full rewrite, collapses into the same undifferentiated WARN. The tool answers *same/different*, never *how different* or *which side*.
2. Ran it against the live tree: two WARNs, everything else `OK`.
3. Measured the actual delta instead of trusting the label — diffed repo `HEAD` against the deployed file in both directions.

## What worked

A real diff showed the entire divergence is a single *additive* line per script:

```bash
$ diff <(git show HEAD:scripts/token_usage_report.py) ~/.hermes/scripts/token_usage_report.py
76a77
>     'step-5-preview-free': (0.0, 0.0, 0.0, 0.0),      # free preview 10-08; ...

$ # direction/magnitude, both scripts:
$ for s in token_usage_report.py morning_context.py; do
>   d() { diff <(git show HEAD:scripts/$s) ~/.hermes/scripts/$s; }
>   echo "$s: only-in-repo=$(d | grep -c '^<') only-in-deployed=$(d | grep -c '^>')"
> done
token_usage_report.py: only-in-repo=0 only-in-deployed=1
morning_context.py: only-in-repo=0 only-in-deployed=1
```

`only-in-repo=0` is the tell: no repo line is missing from the deploy, so nothing is stale — the deploy is a *superset* by one deliberate row. That classifies the WARN as expected, not a defect. (The clean fix is the tracked follow-up: commit the deployed-only row so the checker goes green.)

## Takeaway

A drift checker built on `cmp -s` tells you *something* changed, never *what* — so always follow a WARN with a bounded `diff <(git show HEAD:path) deployed` to read magnitude and direction (`^<` vs `^>`); otherwise one known-intentional line turns every run into noise you learn to ignore.
