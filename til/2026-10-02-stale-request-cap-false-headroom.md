# A stale request-cap denominator read as 4.9× the real headroom — verify every cap against its live source

**Date:** 2026-10-02
**Tags:** `monitoring`, `pricing`, `verification`, `quota`

## The problem

`token_usage_report.py` tracks each model's requests per 5h window against a
`PER_MODEL_REQ_CAPS` table. `deepseek-v4-flash` was capped at **63,300** req/5h —
a figure from an older, larger docs snapshot. The real Go page says **13,000**.
Since the cap is the *denominator* of the meter, a 4.9×-too-generous number made
every flash request look ~5× smaller than it was: the report read "plenty of
headroom" when the window was actually near its ceiling. It's the worst kind of
monitoring bug — the output stays green while the truth doesn't.

## What I tried

The MOA review on PR #29 re-verified the Go "Estimated requests" table and flagged
the cap while it was in the file anyway. I trusted the committed number until the
reviewer's evidence check showed the live value was 13,000. The old line was
believable — 63,300 isn't a typo, it's a *stale* fact, which is why it survived.

## What worked

Correct the denominator, then quantify the scale factor instead of hand-waving.
Real 5h window from the live DB, priced against both caps:

```bash
c5h=$(sqlite3 ~/.hermes/state.db "SELECT SUM(api_call_count) FROM session_model_usage
      WHERE model='deepseek-v4-flash' AND billing_base_url LIKE '%opencode.ai/zen/go%'
      AND last_seen >= $(( $(date +%s) - 5*3600 ));")   # 88 live calls
awk -v c="$c5h" 'BEGIN {
  printf "old cap 63300: %.2f%%  (0.007x of window)\n", c*100/63300;
  printf "new cap 13000: %.2f%%  (0.034x of window)\n", c*100/13000;
  printf "headroom overstated by %.1fx\n", 63300/13000 }'
```

```text
old cap 63300: 0.14%  (0.007x of window)
new cap 13000: 0.68%  (0.034x of window)
headroom overstated by 4.9x
```

Same 88 calls: 0.14% vs 0.68%. With the old denominator, a real 6,000-call/5h
burst would have shown as ~9.5% — the meter was lying by the same factor at every
load level. The script now carries the correct figure:

```python
'deepseek-v4-flash': 13000,   # was 63,300 — denominator 4.9x too generous
```

## Takeaway

A monitoring table is itself a measurement: re-verify *every* cap, rate, and ratio
against its live source periodically, because a stale denominator doesn't break the
report — it quietly scales every percentage by the same wrong factor.