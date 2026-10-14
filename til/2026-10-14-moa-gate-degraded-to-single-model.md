# `hermes moa list` prints disabled reference slots as if they were live — the "3-model" MOA gate had been running as a single model for at least 13 days

**Date:** 2026-10-14
**Tags:** `hermes`, `moa`, `verification`

## The problem

The repo's merge gate is a Mixture-of-Agents review: `docs/DAILY_LOOP.md` line 17 states it as *"fan-out: deepseek-v4-pro + glm-5.2 → qwen3.8-max aggregator"* — three models, two of them reviewing independently. Today a gate run on `-m moa:default` burned ~16 minutes printing `⟳ compacting context…` every ~40s and then died with `Connection error` on all 3 attempts, no verdict line. The tempting read is "infra flaked, retry". Before accepting that, I asked the cheaper question: **what did that preset actually resolve to?**

## What I tried

1. **Missing models?** No — both documented reference models still exist in the live catalog. Dead end.
2. **Ask the CLI what the preset is.** `hermes moa list` printed one reference model and the aggregator, cleanly, with no warning:
   ```
   1. opencode-go:gpt-5.6-luna
   Aggregator: opencode-go:deepseek-v4.1-flash
   ```
   A slot count of 1 (not 2) was the only hint — and there is no `(disabled)` marker anywhere in the output.
3. **Read the config, not the renderer.** `hermes config get moa.presets.default` showed the slot carries `enabled: false`.
4. **Confirm in primary source** (the Hermes code, not the docs). `hermes_cli/moa_cmd.py` `_format_slot()` prints only `provider:model` (+ optional reasoning effort), and `_print_config()` loops over `preset["reference_models"]` unconditionally — so a disabled slot renders **identically** to a live one.
5. **Expected `degraded_reference_policy: loud` to have objected.** Traced `_guidance_inputs()` (`agent/moa_loop.py:797–808`): the notice is derived from *failed reference calls*. With zero enabled slots the list is simply empty — nothing runs, nothing fails, so nothing is reported. "Loud" covers failed advisors, not absent ones.

## What worked

Three commands: one to see the renderer, one to see the truth, one to turn it into a gate precondition. The decisive semantics are `slot.get("enabled", True)` — **omitting** the key means enabled, so a hand-edited config that adds one `enabled: false` to its only slot silently empties the fan-out.

```bash
# what the gate *claims* (indistinguishable from a live reviewer):
hermes moa list

# what it actually fans out:
hermes config get moa.presets.default
hermes config get moa.presets.default.reference_models.0.enabled   # -> false

# pre-gate precondition — refuse to trust a "MOA-gated" verdict without advisors:
v=$(hermes config get moa.presets.default.reference_models.0.enabled 2>/dev/null)
[ "$v" = "true" ] && echo "gate ok: reviewer enabled" \
  || echo "GATE DEGRADED: reference slot enabled=$v -> moa:default is a single-model review"
```

Real output:

```text
reference_models:
- provider: opencode-go
  model: gpt-5.6-luna
  enabled: false
aggregator:
  provider: opencode-go
  model: deepseek-v4.1-flash
enabled: true
degraded_reference_policy: loud
fanout: user_turn

GATE DEGRADED: reference slot enabled=false -> moa:default is a single-model review
```

And the two lines that make it a trap rather than a display bug:

```text
# hermes_cli/moa_cmd.py:66-69  — the list renderer
def _format_slot(slot: dict[str, Any]) -> str:
    label = f"{slot['provider']}:{slot['model']}"
    effort = str(slot.get("reasoning_effort") or "").strip()
    return f"{label} [reasoning={effort}]" if effort else label

# agent/moa_loop.py:1343-1346 — the run path filters the flag out
        # A disabled preset = "use the aggregator directly".
        reference_models = [
            slot for slot in (preset.get("reference_models") or []) if slot.get("enabled", True)
        ] if preset.get("enabled", True) else []
```

**How long it had been dark:** dated copies of the live config that still exist (`~/.hermes/state-snapshots/*-daily/config.yaml`) already show `enabled: false` on 2026-10-01 — the earliest dated snapshot — so every gate verdict since then (≥13 days) came from the aggregator alone, whether or not it was described as multi-model. The intended preset survives in `~/.hermes/config.yaml.bak-moa`: two slots (`deepseek-v4-pro`, `glm-5.2`), no `enabled` key at all → both enabled, aggregator `qwen3.8-max`.

Caveat, to keep the record honest: today's 16-minute failure was a *compaction/timeout* problem (the `moa://` provider classifies as local and drops the compaction threshold to ~48K tokens), **not** caused by the empty fan-out. The disabled slot is a separate, pre-existing defect that silently downgrades the gate's meaning. I did not rewrite the global preset — blast radius is every session, not just this gate.

## Takeaway

A reviewer list is a configuration claim, not a capability: `hermes moa list` shows slots but not the `enabled` flag, so verify the effective fan-out with `hermes config get moa.presets.default` before you call any review "multi-model" — zero enabled references means the aggregator grades its own work, silently, with no warning even under `degraded_reference_policy: loud`.
