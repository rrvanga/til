# archify validate without `--repo-root` exits at the evidence gate — the real render diagnostics never surface

**Date:** 2026-09-25
**Tags:** `archify`, `validation`, `diagnostics`, `verification`

## The problem

Issue #26's `agent-setup.architecture.json` kept "passing" or failing on a single diagnostic while the real layout problems stayed invisible. The validator never reached the render stage, so the 27 actual layout violations were invisible until the right flag was added.

## What I tried

1. Re-reading the failing output — it showed one diagnostic (`repository-evidence/source-required`, later `root-required`) and stopped. That single early diagnostic *is* the exit point; everything after it (the render checks) is never run without the repo root.
2. Before the evidence references were added, the same missing flag produced a misleading 1-diagnostic PASS — a green that masked the same 27 real errors.

## What worked

Passing `--repo-root .` (the matching Git checkout) lets the validator verify the declared source evidence and proceed to rendering, where the real diagnostics appear. Counting them with a one-liner over the JSON output:

```bash
node /home/crashcrosster/dev/archify/archify/bin/archify.mjs validate \
  docs/experiments/archify/agent-setup.architecture.json \
  --json --quality showcase --repo-root . > /tmp/full.json
node -e '
const j = JSON.parse(require("fs").readFileSync("/tmp/full.json","utf8"));
console.log("ok:", j.ok, "| stage:", j.stage);
const codes = {};
for (const d of j.diagnostics || []) codes[d.code] = (codes[d.code]||0)+1;
for (const [k,v] of Object.entries(codes).sort((a,b)=>b[1]-a[1])) console.log("  ", k, "×"+v);
'
# ok: false | stage: render
#    composition/label-route-clearance ×18
#    clean-flow/endpoint-side-direction ×6
#    clean-flow/edge-through-node ×1
#    composition/short-interior-segment ×1
#    layout/constraint ×1
```

27 diagnostics, now actionable (18 `label-route-clearance` + 6 `endpoint-side-direction` + 1 each of the rest).

## Takeaway

`archify validate --quality showcase` is staged: without `--repo-root` it stops at the evidence gate and can even report green — always pass `--repo-root` against the matching checkout, or you'll fix one phantom error while the real layout violations stay hidden.