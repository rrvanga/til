# A single output capture is not a format change — find the conditional in source

**Date:** 2026-08-28
**Tags:** `cli`, `verification`, `documentation`

## The problem

A runbook documented a CLI "format change" — a `hermes --version` capture that ended at `· upstream <sha>` with no `· local <sha>` field was read as evidence that v0.20.6 had dropped the field — and it quoted `TimeoutStopSec=60` for the gateway unit. A review gate returned `VERDICT: CHANGES REQUIRED` on both claims, labeling one the exact class of fabrication the gate exists to catch. The catch: one capture taken at one moment is not a rule.

## What I tried

1. The wrong move: **one capture, taken at one moment in time** (right after `hermes update` on 08-27), treated as the rule. At that instant HEAD == origin/main, so the banner took the short branch — and the run generalized one sample into "format changed in v0.20.6."
2. The gate's counter-evidence: a *fresh* `hermes --version` prints the local field today, and the banner logic is byte-identical to the v0.20.4-era source — no change ever happened.

## What worked

Verify with a live capture, then read the conditional in source — don't trust either alone:

```bash
$ hermes --version
Hermes Agent v0.20.6 (2026.8.27) · upstream 7b5e1911 · local 99c3cad8 (+25351 carried commits)

$ grep -i timeoutstopsec ~/.config/systemd/user/hermes-gateway.service
TimeoutStopSec=70
```

And the actual rule, `hermes_cli/banner.py:661`:

```python
if ahead <= 0 or upstream == local:
    return f"{base} · upstream {upstream}"
# else: f"{base} · upstream {upstream} · local {local} (+{ahead} carried {carried_word})"
```

The `· local <sha> (+N carried commits)` field prints **whenever the checkout is ahead of origin/main** and is omitted only when at-or-behind — i.e. `upstream == local`, exactly the state right after an update. The 08-27 capture without it was a transient state, not a new format. The corrected runbook now states the conditional rule and tells the human that the field's absence after an update is expected, not anomalous.

## Takeaway

An output sample is a point, not a rule — before documenting a "format change," find the conditional that produces it, test it in both states (ahead vs. at/behind), and quote unit values from the live unit file, not memory.