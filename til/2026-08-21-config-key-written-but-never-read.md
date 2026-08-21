# A config key can be written and whitelisted yet never read — grep for readers before calling it dead

**Date:** 2026-08-21
**Tags:** `config`, `hermes`, `code-audit`, `grep`, `dead-code`

## The problem

Issue #3 in the agent-lab claimed `smart_model_routing` was a vestigial config key and should be stripped from docs and the live config. But "nobody seems to use it" is a vibe, not proof — a key can exist in the config-cleaning path for years precisely *because* the write-side code keeps re-creating it. I needed a falsifiable check: does anything actually *read* it at runtime?

## What I tried

First the naive check — is it referenced anywhere in the source at all?

```bash
grep -rn "smart_model_routing" --include="*.py" ~/.hermes/hermes-agent/ | grep -v test
```

That returned hits, so the key was definitely "alive" somewhere. The trap is stopping there: it proved *existence*, not *use*. I had to bucket every hit into write-side vs read-side:

- `hermes_cli/setup.py:3231` — `config.setdefault("smart_model_routing", {})["enabled"] = False` → **write** (setup wizard re-creates it on every new config)
- `hermes_cli/config.py:1998` — a whitelist entry with the comment "written by the setup wizard" → **schema allowance**, not a read
- `tests/hermes_cli/test_setup_blank_slate.py` → **test assertion**, not runtime logic

No `config.get("smart_model_routing")`, no lookup in routing code, nowhere. So the key was write-only: written on setup, allowed by the schema (so no "unknown key" warning), asserted by a blank-slate test — and never consulted by any runtime path.

## What worked

Bucket every reference by direction (write / whitelist / test / read) and demand at least one genuine read site before treating the key as used:

```bash
$ cd ~/.hermes/hermes-agent
$ grep -rn "smart_model_routing" --include="*.py" . | grep -v test
./hermes_cli/config.py:1998:    "smart_model_routing",   # written by the setup wizard (hermes_cli/setup.py)
./hermes_cli/setup.py:3231:    config.setdefault("smart_model_routing", {})["enabled"] = False

$ grep -c "smart_model_routing" ~/.hermes/config.yaml   # 0 — already gone from live config
0
$ hermes config get smart_model_routing
Config key not set: smart_model_routing
$ grep -n -A3 "smart_model_routing" ~/.hermes/config.yaml.bak-smr-20260821-090301  # the removed block
277:smart_model_routing:
278-  enabled: false
```

Zero runtime reads ⇒ safe to strip from docs and live config (`hermes config unset` + backup first). The removal landed as PR #13 (`bff26d8`) and the config half the same morning.

## Takeaway

A config key that is written by a wizard, whitelisted in the schema, and asserted by a test is not "used" — check for a read site; if there is none, it is dead weight with surprisingly good documentation.