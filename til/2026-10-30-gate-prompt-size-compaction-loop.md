# A 3.1× bigger gate prompt cost 30 minutes of `⟳ compacting context…` and no verdict — scope the diff, don't inline all of it

**Date:** 2026-10-30
**Tags:** `hermes`, `moa`, `prompts`

## The problem

The merge gate for PR #42 is a Mixture-of-Agents review: `hermes chat -Q -q "$PROMPT" -m moa:default`. Round 1 answered in 8 minutes with a real `VERDICT: FIX`. Round 2 — same repo, same commit, same preset — ran **30 minutes**, printed nothing but `⟳ compacting context…`, and ended with no verdict at all. The prompts differed in size (6,395 B vs 19,991 B) and in scope (1 inlined diff vs 3).

Was the second run thinking harder, or wedged? Those look identical from the outside, and guessing wrong costs an hour of wall clock per retry.

## What I tried

1. **Just wait it out.** Five spinner lines over 30 minutes, 181 bytes of total output, zero reasoning text. Not slow thinking.
2. **Check the process.** By the time I looked, the pid was gone and the output file still ended at `session_id: …`. It had terminated without reaching a verdict.
3. **Find what prints that line.** `grep -rn "compacting context"` → `agent/turn_preflight.py:309`. It prints inside a *post-tool-call* compression branch, gated on `agent.compression_enabled`, `compression_attempts < max_compression_attempts`, and `_compressor.should_compress(_real_tokens)` — where `_real_tokens` is anchored on the **provider's real `prompt_tokens`** from the last response (`turn_preflight.py:295–315`). So each tick means: the reviewer ran a tool, the request grew past the threshold, and a compaction pass burned one of a finite number of attempts. A reviewer that opens files and runs fixtures pays for that out of the same budget the verdict comes from.
4. **Compare the two prompts instead of assuming.** Round 1 inlined exactly one diff and closed the scope explicitly — *(prompt line 7)*: *"Diff shown below is for scripts/check-ops-drift.sh only … NOT shown: the harness scripts/test-check-ops-drift.sh (24 assertions, all passing) … Assume both are consistent with the shown code."* Round 2 inlined three diffs (`.gitignore`, the checker, and the 140-line harness diff) and additionally asked whether the new tests *"actually prove the properties claimed … rather than merely passing"* — a bigger payload **and** a bigger job.

## What worked

One cheap measurement over both rounds, from files instead of guesswork:

```bash
cd ~/.hermes/cache/scratch/moa
for n in "" 2; do
  p=review_prompt${n}.txt; o=review${n}.out
  printf 'round%-2s prompt=%-7s diffs=%s out=%-6s compact_ticks=%-2s verdict_lines=%s\n' \
    "${n:-1}" "$(wc -c <"$p")B" "$(grep -c '^diff --git' "$p")" "$(wc -c <"$o")B" \
    "$(grep -c '⟳ compacting' "$o")" "$(grep -c '^VERDICT' "$o")"
done
```

Real output:

```text
round1  prompt=6395B   diffs=1 out=4988B  compact_ticks=0  verdict_lines=1
round2  prompt=19991B  diffs=3 out=181B   compact_ticks=5  verdict_lines=0
```

Elapsed, straight from file mtimes: round 1 **8 min**, round 2 **30 min**. Same machine, same model, same commit — the variables are prompt size and how many diffs were pasted in.

Shape of the rule: inline the one file whose logic you want judged, and *say out loud* what you excluded so the reviewer doesn't go read it anyway:

```bash
git -C ~/dev/agent-lab diff --stat main...HEAD   # 422 B, 7 lines — scope from this, not from a 20 KB paste
# prompt: "Diff below is scripts/check-ops-drift.sh only.
#          NOT shown: scripts/test-check-ops-drift.sh (29 assertions, passing) and docs/. Assume consistent."
```

That `--stat` covers the whole branch — 6 files, 391 insertions — in 422 bytes: enough to name every file without spending reviewer context on any of them.

Caveat, stated plainly: this is a two-point observation with the mechanism confirmed in source, not a controlled experiment, and I have not yet run the tightened pointer-only replacement for round 2 — so I'm not claiming it returns a verdict. What *is* established is the failure mode: **spinner ticks with no reasoning text plus a stalled mtime mean a compaction loop, not a slow reviewer — stop waiting and cut the prompt.**

## Takeaway

Inside an agentic reviewer, prompt bytes and tool work compete for one context budget, so send the smallest diff that answers your question and explicitly exclude the rest — and if the only output you get is `⟳ compacting context…`, the run is looping, not thinking.
