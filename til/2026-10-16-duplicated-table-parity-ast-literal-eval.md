# One price table in four files: compare duplicated Python literals with `ast.literal_eval` — no import, key-level deltas

**Date:** 2026-10-16
**Tags:** `python`, `ast`, `ops-drift`

## The problem

The Go burn meter's rate table `GO_RATES` (57 entries of `name -> (in, out, cache-read, cache-write)` tuples, each with a long trailing comment) is duplicated across **four** files: the two deployed cron scripts `~/.hermes/scripts/token_usage_report.py` and `~/.hermes/scripts/morning_context.py`, plus their canonical copies in the `agent-lab` repo. The scheduled monitor job edits the *deployed* pair only, so the repo copies keep falling behind — the same two files have been re-reconciled in `bb4ac71` (09-11), `9099098` (10-02), `b1e0615` (10-13) and `1b7f137` (today, three new `-free` rows added and `cat-jam-free` removed). Today's drift checker finally reports green, but "green" only means the files are byte-identical. I wanted a check that answers *"do the four tables still carry the same data, and if not, which key moved?"* — and that I can trust without running any of the scripts.

## What I tried

1. **The existing drift checker** (`scripts/check-ops-drift.sh`, byte granularity). Correct as a gate, useless for triage: it says "content differs" and stops. The comments in this table churn almost daily (today's edits re-annotated `cat-jam-free`), so a raw diff makes you read 57 comment-heavy lines to find one changed tuple.
2. **`diff <(grep -E "'[a-z].*':" f1) <(grep ... f2)`** — line-oriented, so a reordered key or a reflowed comment reads as a change, and wrapped comment lines make the grep brittle. False positives in both directions.
3. **Importing both modules and comparing the dicts** — rejected on principle: these are ops scripts, so importing executes module-level code (config reads, network fetches), and the second import would just shadow the first. Never import production scripts to answer a data question.
4. **Regex-extracting the table** — also rejected: the values are tuples of floats and the comments contain apostrophes; since the file is Python anyway, there's a safe evaluator built in.

## What worked

Parse the file, find the top-level assignment, `ast.literal_eval` its right-hand side. No import, no execution, and the digest is computed over *sorted* keys, so comment churn and key reordering don't move it — a matching digest is a statement about the data, not about the text. `sys.exit(1)` on any delta makes it usable as a gate.

```python
#!/usr/bin/env python3
"""Compare a duplicated Python data table across N files without importing them."""
import ast
import hashlib
import sys
from pathlib import Path

TARGET = "GO_RATES"


def rates(path):
    for node in ast.parse(Path(path).read_text()).body:
        if isinstance(node, ast.Assign) and getattr(node.targets[0], "id", None) == TARGET:
            return ast.literal_eval(node.value)          # never imports/executes the file
    sys.exit(f"{path}: no top-level {TARGET}")


def digest(t):
    body = "\n".join(f"{k}={t[k]!r}" for k in sorted(t))  # sorted: order/comment churn is invisible
    return hashlib.sha256(body.encode()).hexdigest()[:12]


def short(p):
    return p.replace(str(Path.home()), "~")


files = [(p, rates(p)) for p in sys.argv[1:]]
for p, t in files:
    print(f"{len(t):>3} keys  {digest(t)}  {short(p)}")
refp, ref = files[0]
drifted = False
for p, t in files[1:]:
    missing = sorted(set(ref) - set(t))
    extra = sorted(set(t) - set(ref))
    changed = sorted(k for k in set(ref) & set(t) if ref[k] != t[k])
    bad = bool(missing or extra or changed)
    drifted |= bad
    print(f"  vs {short(p)}: {'DIFFERS' if bad else 'IDENTICAL'}",
          *[f"{n}={v}" for n, v in (("only-in-first", missing),
                                    ("only-in-second", extra),
                                    ("value-mismatch", changed)) if v])
sys.exit(1 if drifted else 0)
```

Real run against all four copies (paths shown as the script prints them):

```console
$ python3 rates_parity.py ~/.hermes/scripts/token_usage_report.py ~/.hermes/scripts/morning_context.py \
      ~/dev/agent-lab/scripts/token_usage_report.py ~/dev/agent-lab/scripts/morning_context.py
 57 keys  1889166698c4  ~/.hermes/scripts/token_usage_report.py
 57 keys  1889166698c4  ~/.hermes/scripts/morning_context.py
 57 keys  1889166698c4  ~/dev/agent-lab/scripts/token_usage_report.py
 57 keys  1889166698c4  ~/dev/agent-lab/scripts/morning_context.py
  vs ~/.hermes/scripts/morning_context.py: IDENTICAL
  vs ~/dev/agent-lab/scripts/token_usage_report.py: IDENTICAL
  vs ~/dev/agent-lab/scripts/morning_context.py: IDENTICAL
$ echo "rc=$?"
rc=0
```

And the negative test that proves it actually detects drift — a copy with one value edited and one key deleted on purpose:

```console
$ python3 rates_parity.py fixture_drift.py ~/.hermes/scripts/token_usage_report.py
 56 keys  a6407c497134  fixture_drift.py
 57 keys  1889166698c4  ~/.hermes/scripts/token_usage_report.py
  vs ~/.hermes/scripts/token_usage_report.py: DIFFERS only-in-second=['step-5-preview-free'] value-mismatch=['space-bunny']
$ echo "rc=$?"
rc=1
```

Key count drops 57 → 56, and the two injected faults are named individually instead of "these files differ". Note what this does *not* fix: the writer-side cause is still open — the monitor job keeps editing only the deployed pair (tracked separately as a repo-aware-writer / single-source option). A structural diff makes the drift cheap to see and cheap to repair; it doesn't stop it being generated.

## Takeaway

When the same data lives in more than one file, don't diff the files — extract the value and diff the data: `ast.literal_eval` on the top-level literal gives you an import-free, comment-proof comparison that names the exact keys that drifted.
