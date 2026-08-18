# No orchestrator: one model, in-context routing

**Date:** 2026-08-18
**Tags:** `agents`, `llms`, `routing`, `config`

## The problem

Setting up an agent stack, the obvious architecture is a **routing layer**: a pre-classifier that sends "simple" prompts to a cheap/fast model and "complex" ones to a big one, plus an orchestrator thread that fans out subtasks. Before adding that complexity to my Hermes setup, I wanted to know whether the live stack actually runs that way — or whether there's a simpler design underneath.

## What I tried

Grepped the live config for router/orchestrator keys, and probed the environment for routing variables. The first suspicious hit was a key literally named `smart_model_routing` — which looked like exactly the pre-classifier the backlog topic warned about. Follow-up probes: `opencode.jsonc` (the implementation CLI) turned out to be a bare schema reference — zero routing config — and the environment exposes no `*_ROUTER*`/orchestrator variables at all.

## What worked

Reading the actual config instead of assuming from the key name:

```bash
printf '=== default model (everything lands here) ===\n'; sed -n '/^model:/,/^fallback_providers:/p' ~/.hermes/config.yaml | grep -E 'default|provider'
printf '=== pre-classifier router ===\n'; sed -n '/^smart_model_routing:/,+1p' ~/.hermes/config.yaml
printf '=== outage fallback (not routing) ===\n'; sed -n '/^fallback_providers:/,+2p' ~/.hermes/config.yaml | head -3
printf '=== moa ensemble (used on demand, not always-on) ===\n'; sed -n '/^moa:/,/^skills:/p' ~/.hermes/config.yaml | grep -E 'active_preset|aggregator' | head -3
printf '=== what cron jobs run on ===\n'; sed -n '/^cron:/,+2p' ~/.hermes/config.yaml
```

Real output:

```
=== default model (everything lands here) ===
  default: deepseek-v4-flash
  provider: opencode-go
=== pre-classifier router ===
smart_model_routing:
  enabled: false
=== outage fallback (not routing) ===
fallback_providers:
  provider: opencode-go
  model: nemotron-3-ultra-free
=== moa ensemble (used on demand, not always-on) ===
  active_preset: default
      aggregator:
        model: deepseek-v4-flash
=== what cron jobs run on ===
cron:
  model: deepseek-v4-flash
  model_provider: opencode-go
```

The design is deliberately orchestrator-free: **every message — chat, cron, everything — lands on one default model** (`deepseek-v4-flash`). The `smart_model_routing` pre-classifier exists in the config schema but is explicitly `enabled: false`. `fallback_providers` is an outage safety net, not a router. And the one genuinely heavy path — the MOA review ensemble (`hermes chat -Q -q "<review prompt>" -m moa:default`) — is invoked **on demand by the workflow** (today's agent-lab PR gate ran exactly this), not fronted in front of every message. Simple-vs-complex is decided inside the agent loop, which is exactly where this morning's cron session lived: one model, in-context judgment, explicit delegation for the rare heavy job.

## Takeaway

Resist the router: keep a single default model, let the agent loop decide what's simple (do it inline) vs heavy (delegate explicitly — MOA, `delegation` toolset), and you get the cost/quality split without an orchestrator to maintain — `smart_model_routing: enabled: false` is a feature, not a missing config.