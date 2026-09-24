# archify validate is staged — a schema-clean file still fails the showcase layout gate

**Date:** 2026-09-23
**Tags:** `archify`, `diagrams`, `validation`, `verification`

## The problem

The archify prototype handoff said "one mechanical schema fix remains — remove `variant` from 11 components, then 9/9 checks pass." When I opened the file, the components already had **no** `variant` keys. So why did the previous run fail, and what is actually left to fix?

## What I tried

First, stop trusting the handoff and validate the file on disk:

```bash
node ~/dev/archify/archify/bin/archify.mjs validate architecture \
  docs/experiments/archify/agent-setup.architecture.json --quality showcase --json
```

The summary claimed 11 identical schema errors (`/components/* must NOT have additional properties {variant}`) — but the current file produced **zero** schema errors. The run instead exited 1 at a different gate, reporting `stage: "render"` with 29 diagnostics. Then I tried the obvious "fix": on a scratch copy, dropped the hand-authored `fromSide: "bottom"` + `via` routes from c3/c13 to let the renderer auto-route.

## What worked

The JSON output carries a `stage` field — that is the lever for diagnosing which gate you actually failed (I saved output via the CLI, then counted codes with a small node script; a `jq -r '.diagnostics[].code'` pipe does the same):

```bash
$ node archify.mjs validate architecture agent-setup.architecture.json --quality showcase --json > /tmp/v.json; echo exit=$?
exit=1
$ node count-diag.js /tmp/v.json
ok: false | stage: render | schemaVersion: 1
diagnostics total: 29
  clean-flow/endpoint-side-direction 8
  composition/label-route-clearance 7
  clean-flow/edge-through-node 6
  layout/constraint 5
  composition/micro-segment 2
  composition/ambiguous-corridor 1
```

Schema validation (1.0) passed silently — the failure is the *second* gate: clean-flow + composition + layout quality (showcase). Fixing what I hypothesized caused them (auto-routing instead of authored `via`) removed only one diagnostic (29 → 28); `clean-flow/edge-through-node` and `composition/label-route-clearance` stayed flat. The real problem is the component geometry (desktop/opencode/kanban columns at 2px clearance, labels crossing boxes), which no routing tweak in the connections array can fix — positions and `labelDy`/`labelAt` must change.

## Takeaway

Treat a staged validator's `stage` + `code` counts as the source of truth, not a handoff's single-class summary — and when geometry checks fail, re-tune the layout, not just the routes.