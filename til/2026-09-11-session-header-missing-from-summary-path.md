# The x-opencode-session fix was in the running Hermes code, yet the max-iterations summary path still sent the request without it

**Date:** 2026-09-11
**Tags:** `opencode-go`, `headers`, `verification`, `error-handling`

## The problem

The 09-08 TIL diagnosed OpenCode Go's `400 MissingSessionID` (requests must carry `x-opencode-session`). The fix was supposedly in place — commit `139396995a` "feat(opencode): send x-opencode-session on every OpenCode request" is in the installed tree, and its docstring claims the header "cannot drift per code path". Yet the very next working day (2026-09-09 10:06) the daily-engineering-loop cron job died with the *identical* 400 on its end-of-run summary. Grep said the feature existed — so why did it fail again?

## What I tried

1. Ruled out the stale-process theory: `git merge-base --is-ancestor 139396995a a6e10e693f` → the commit IS an ancestor of the checked-out HEAD; `gateway_state.json` stamps `code_sha = a6e10e693f…`, `code_version = 0.21.0`; the gateway process started 09-09 08:11, *after* the 09-02 commit. The running code genuinely contains the feature.
2. Grepped every request path that merges the header: only TWO call sites.

## What worked

Tracing the failing request from the log line to the code that builds it:

```bash
$ grep -a "MissingSessionID" ~/.hermes/logs/errors.log | tail -1
2026-09-09 10:06:01,386 WARNING [cron_26e42ff314c6_20260909_090029] agent.chat_completion_helpers: Failed to get summary response: Error code: 400 - {'type': 'error', 'error': {'type': 'MissingSessionID', 'message': 'Error from provider (Console Go): Request is missing x-opencode-session and cannot be routed efficiently. Please see https://opencode.ai/docs/go/#where-can-i-use-it'}}

$ grep -n "Failed to get summary response" ~/.hermes/hermes-agent/agent/chat_completion_helpers.py
2366:        logger.warning("Failed to get summary response: %s", e)

$ grep -n "merge_opencode_session_headers" ~/.hermes/hermes-agent/agent/chat_completion_helpers.py ~/.hermes/hermes-agent/agent/auxiliary_client.py
agent/chat_completion_helpers.py:1534:    from agent.opencode_affinity import merge_opencode_session_headers
agent/chat_completion_helpers.py:1537:    return merge_opencode_session_headers(
agent/auxiliary_client.py:5971:    from agent.opencode_affinity import merge_opencode_session_headers
agent/auxiliary_client.py:5972:    return merge_opencode_session_headers(kwargs, provider, base_url, _runtime_main_value("session_id") or None)
```

The two merge sites cover (a) main-turn requests (`build_api_kwargs`, line 1537) and (b) auxiliary calls (line 5972). The max-iterations summary is a **third** path: `_iteration_summary_chat_kwargs()` (line 2230) builds its own `summary_kwargs = {"model", "messages", …}` and never merges the header — so `_chat_summary_attempt` (line 2312) sends the summary request headerless and the provider 400s. The `codex_responses` summary path (line 2294, via `agent._build_api_kwargs`) DOES get the header, which proves the summary path was meant to mirror it.

This also explains the recurrence pattern: the failure only fires when a run hits max_iterations (=50) and summarizes — main turns were fine. `errors.log` shows the same 400 since 09-07 (20:06, 22:46, 09-08 09:48, 09-09 10:06).

## Takeaway

"Present in the tree" — even present in the **running process** — is not "present on the failing path": grep the exact error's log line to the code that emits it, then read how *that* request is built; coverage claims in docstrings are not tests, and multiple request builders in one module drift easily. (The one-line fix: call `merge_opencode_session_headers` in `_iteration_summary_chat_kwargs`.)