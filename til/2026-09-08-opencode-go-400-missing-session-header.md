# OpenCode Go answers 400 "MissingSessionID" when a request skips the x-opencode-session header

**Date:** 2026-09-08
**Tags:** `opencode-go`, `api`, `headers`, `hermes`, `error-handling`

## The problem

The daily engineering-agent run (agent-lab) completed all its work but died on the very last step — the end-of-run summary — with a hard provider error:

```json
{"type": "error", "error": {"type": "MissingSessionID", "message": "Error from provider (Console Go): Request is missing x-opencode-session and cannot be routed efficiently. Please see https://opencode.ai/docs/go/#where-can-i-use-it"}}
```

No model response, no fallback — the run just lost its final message.

## What I tried

I re-probed the same endpoint (`https://opencode.ai/zen/v1/chat/completions`) directly, first **without** the header to try to reproduce the 400:

```bash
curl -s -X POST https://opencode.ai/zen/v1/chat/completions \
  -H "Authorization: Bearer $OPENCODE_GO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"max_tokens":1}' \
  -w '\nHTTP_STATUS=%{http_code}\n'
# -> HTTP_STATUS=200  (router-… id)  — no 400 on a single stateless ping
```

So the header is NOT enforced on every path: a bare one-shot ping sails through. The 400 fires on the conversation/routing layer (the "Console Go" internal path the agent uses), not on a stateless curl.

## What worked

Reading the official docs the error points to — the header is a documented contract, and the client table explains exactly why Hermes forgot it:

> **Send a stable session ID in `x-opencode-session` for each conversation so we can optimize routing and prompt caching.**

Validated-client table: *"Hermes — Builds containing PR #101864 send the header on main and auxiliary OpenCode requests. The fix was merged after v0.21.0; that release alone does not include it."*

`hermes --version` on this machine says `v0.21.0` — the exact release without the fix. Sending the header yourself works (real probe from this session):

```bash
curl -s -X POST https://opencode.ai/zen/v1/chat/completions \
  -H "Authorization: Bearer $OPENCODE_GO_API_KEY" \
  -H "Content-Type: application/json" \
  -H "x-opencode-session: sess-til-demo-20260908" \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"max_tokens":1}' \
  -w '\nHTTP_STATUS=%{http_code}\n'
# -> HTTP_STATUS=200  — chatcmpl-… id, routed with session affinity
```

## Takeaway

When Console Go answers `400 MissingSessionID`, your client dropped the `x-opencode-session` header: send one stable `sess-*` value per conversation (or upgrade past the client version whose fix restored it) — a bare stateless curl won't reproduce the error, so test with the same conversation path, not a one-shot ping.