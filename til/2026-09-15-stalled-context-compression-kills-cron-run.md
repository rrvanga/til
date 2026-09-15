# Hermes terminates the turn when context compression stalls — a long cron session dies with "Context compression timed out"

**Date:** 2026-09-15
**Tags:** `hermes`, `cron`, `context-compression`, `timeouts`

## The problem

The daily engineering-loop cron job failed for the third time this month with a bare terminal error and zero delivered output:

```
RuntimeError: Context compression timed out without reducing this conversation. No messages were dropped. Start a fresh session with /new, or check auxiliary.compression before retrying /compress.
```

A long autonomous session (dozens of tool calls, big diffs and review transcripts) eventually crosses the compression threshold — and when the compressor can't finish, the whole run dies with no partial results.

## What I tried

- Checked whether it was a one-off: grepping the cron output store showed the same error in three runs — `26e42ff314c6/2026-09-11_13-37-11.md`, `26e42ff314c6/2026-09-14_19-30-39.md`, and another job `3b8f77fe13ca/2026-09-14_18-50-40.md`. Recurring, not a flake.
- Checked `~/.hermes/config.yaml`: compression is enabled (`threshold: 0.5`, `threshold_tokens: 48000`, `target_ratio: 0.2`, `protect_last_n: 20`), but there is **no `auxiliary:` section at all** — the error message's advice ("check auxiliary.compression") points at a config block that isn't there. No `context_timeout_seconds` tuning either.
- Read the Hermes source to find who raises this. Key facts:
  - Compression prunes old tool results first (no LLM call), then generates a structured summary with the **auxiliary compression model** (`agent/conversation_compression.py`, `agent/AGENTS.md`). That summary step is an LLM call.
  - The summary call runs under a progress-aware host timeout: `DEFAULT_CONTEXT_TIMEOUT_SECONDS = 120.0` idle, `DEFAULT_CONTEXT_TOTAL_CEILING_SECONDS = 600.0` total, tunable via `compression.context_timeout_seconds` / `compression.context_total_ceiling_seconds`.
  - If there's no progress from the summary model and the request is *still oversized*, `agent/conversation_loop.py` (#98722) deliberately ends the turn with the exact RuntimeError above: re-sending the unchanged request would only bounce off the provider's overflow error and re-enter compression. Terminal by design, not an accident.
  - Repeated stalls escalate a cooldown ladder 60s → 300s → 900s (`_TIMEOUT_COOLDOWN_LADDER`), so retrying makes it worse.

## What worked

Diagnosing it as a design-level stop, not a transient, using the actual outputs:

```bash
# 1) prove the failure is recurring across cron runs (real output)
$ grep -rl 'Context compression timed out' ~/.hermes/cron/output/ | sed 's|.*/||'
2026-09-14_18-50-40.md
2026-09-11_13-37-11.md
2026-09-14_19-30-39.md

# 2) what the summary model is allowed to wait before the run dies (real output)
$ grep -n 'DEFAULT_CONTEXT_TIMEOUT_SECONDS\|DEFAULT_CONTEXT_TOTAL_CEILING_SECONDS' \
    ~/.hermes/hermes-agent/agent/conversation_compression.py
581:DEFAULT_CONTEXT_TIMEOUT_SECONDS = 120.0
582:DEFAULT_CONTEXT_TOTAL_CEILING_SECONDS = 600.0

# 3) the config surface that is NOT being used today (real output — empty)
$ grep -n -i 'auxiliary' ~/.hermes/config.yaml
```

The fix options that fall out of this: pin a responsive summary model under `auxiliary.compression` (provider/model) so the compress call can't stall, raise `compression.context_timeout_seconds` if the provider is just slow, or — best for long autonomous jobs — keep the session short enough that compression never fires mid-mission (fresh session per phase, delegate heavy tool output instead of accumulating it in one conversation).

## Takeaway

If a Hermes cron run dies with "Context compression timed out", it's not a flake — the aux summary LLM stalled past its 120s budget while the conversation was still oversized, and the loop is coded to stop rather than resend; pin `auxiliary.compression`, raise the timeout, or bound the session's growth.