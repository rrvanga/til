# archify: force orthogonal lanes with truthful `fromSide`/`toSide` and an explicit `via` detour

**Date:** 2026-09-28
**Tags:** `archify`, `diagrams`, `validation`, `routing`

## The problem

The agent-lab architecture diagram (issue #26) failed archify's `showcase` quality gate with 27 validator diagnostics: grid auto-routing drew diagonal first/final segments out of hermes's right side and stacked hub-edge labels, so edges weren't orthogonal and crossings/corridor checks blew up.

## What I tried

- Left routing to the grid algorithm with no hints — that was the bug: it picks whatever side is geometrically closest, which fights the orthogonal-lane rules.
- Declared truthful `fromSide`/`toSide` pairs on the 8 hub edges (mostly `top`/`bottom`, matching where the components actually sit) so the router emits clean orthogonal lanes at y=182/234/390.
- For the long `github→hermes` back-route I pinned `via: [[1116,390],[276,390]]` — without it, the edge routed through the opencode node even after the side hints.
- Tightened over-wide labels ("provider pool · cheapest adequate" → "cheapest adequate") to clear the label-route gate.

## What worked

Validation now passes at showcase quality end-to-end (this is real output from this session's run):

```bash
cd ~/dev/agent-lab
node ~/dev/archify/archify/bin/archify.mjs validate architecture \
  docs/experiments/archify/agent-setup.architecture.json \
  --quality showcase --repo-root . --json
```

```json
{
  "ok": true,
  "checks": [
    { "name": "orthogonal_arrows", "ok": true },
    { "name": "label_route_clearance", "ok": true },
    { "name": "relationship_crossings", "ok": true },
    { "name": "relationship_corridors", "ok": true },
    { "name": "route_rhythm", "ok": true }
  ],
  "composition": { "status": "pass", "summary": { "errors": 0, "warnings": 0 },
    "metrics": { "properCrossings": 0, "ambiguousCorridors": 0,
                 "maxBends": 2, "minLabelRouteClearance": 6 } }
}
```

The IR change that did the heavy lifting (verified in the committed file):

```json
{ "id": "github-to-hermes", "from": "github", "to": "hermes",
  "label": "review results", "fromSide": "bottom", "toSide": "bottom",
  "via": [[1116, 390], [276, 390]] }
```

## Takeaway

Tell the grid router which side each edge truly leaves and enters from (and pin long back-routes with explicit `via` points) — auto-routing happily draws diagonals that fail the orthogonality and corridor checks, and the coordinates are grid-derived, so re-derive them whenever the layout changes.