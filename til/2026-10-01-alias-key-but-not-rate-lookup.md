# An alias applied to the aggregation key — but not the rate lookup — silently prices the lane at the default rate

**Date:** 2026-10-01
**Tags:** `python`, `pricing`, `aliases`, `verification`, `monitoring`

## The problem

The MOA gate on PR #29 (`feat/28-reconcile-ops-drift`) flagged the shared-pool cost
scripts: `token_usage_report.py` and `morning_context.py` reported `unknown` models
and mis-priced cost lanes. The SQL `GROUP BY model` returns gateway alias strings
(`default`, `agent-main`) that are not keys in the `GO_RATES` table of canonical
model names.

## What I tried

The aggregation-side fix was already in place: `m = MODEL_ALIASES.get(model, model)`
so totals roll up under canonical names. But the **rate lookup** on the same line
still used the raw string — `rate = GO_RATES.get(model) or GO_DEFAULT_RATE`. So the
alias only fixed the label, not the math: any alias row fell through to the default
rate while being displayed under the canonical model name, which made the error
invisible.

## What worked

Alias first, then look up the canonical name. Demonstrated with a miniature of the
exact pattern:

```python
# per_model_cost / _pm_costs — fixed order
for model, c, i, o, r, w in rows:
    m = MODEL_ALIASES.get(model, model)      # 1. alias FIRST
    rate = GO_RATES.get(m) or GO_DEFAULT_RATE  # 2. rate lookup on canonical name
    costs[m] = costs.get(m, 0.0) + (i or 0)/1e6*rate[0] + (o or 0)/1e6*rate[1] \
        + (r or 0)/1e6*rate[2] + (w or 0)/1e6*rate[3]
```

Real output, same rows, lookups on raw vs aliased name:

```text
=== BEFORE (bug) ===
costs = {'deepseek-v4-flash': 0.63}  (rate used: (0.15, 0.6, 0.3, 0.3) = DEFAULT)
=== AFTER (fix) ===
costs = {'deepseek-v4-flash': 0.41000000000000003}  (rate used: (0.1, 0.4, 0.1, 0.2))
```

## Takeaway

When a key needs aliasing, alias the variable once at the top of the loop and use
that same variable for **every** lookup — aliasing only the bucket key hides the bug
by mis-pricing under a correct-looking name.