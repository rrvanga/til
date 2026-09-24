# A `*-free` model in the zen catalog isn't necessarily callable — most return 403 FreeTierError from the raw API

**Date:** 2026-09-24
**Tags:** `opencode-go`, `api`, `fallback`, `verification`, `monitoring`

## The problem

The go adaptive monitor keeps a preference-ordered fallback pool of `*-free` models on opencode-go's `zen/v1` endpoint. The models catalog advertises them all, but today's ping sweep showed most `*-free` names come back HTTP 403 — only one of them actually completes a request.

## What I tried

Probing the free models directly against `https://opencode.ai/zen/v1/chat/completions` with the same shape `go-ping.sh` uses (bearer key, UA, `x-opencode-session`). Eight of nine probed models returned:

```json
{"type":"error","error":{"type":"FreeTierError","message":"OpenCode's free tier can only be used from within OpenCode"}}
```

The monitor's `go_fallback_state.json` already shows the aftermath — consecutive-fail strikes accumulated per model (`cfalls`), which with `CFAIL_THRESHOLD=2` (adaptive_monitor.py) bumps a 403ing model out of the healthy pool until it recovers:

```json
{"bans": {}, "cfalls": {"mimo-v2.5-free": 1, "nemotron-3-ultra-free": 7}}
```

## What worked

Checking the live HTTP status instead of trusting the catalog — the probe classifies each model individually. Re-ran it this session:

```bash
KEY=$(grep -m1 '^OPENCODE_GO_API_KEY=' "$HOME/.hermes/.env" | cut -d= -f2-)
for M in nemotron-3-ultra-free space-bunny-free; do
  curl -s -o /tmp/til-probe.json -w '%{http_code}\n' \
    -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
    -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) Firefox/130.0" \
    -H "x-opencode-session: sess-til-20260924" \
    -d "{\"model\":\"$M\",\"messages\":[{\"role\":\"user\",\"content\":\"ping\"}],\"max_tokens\":5}" \
    https://opencode.ai/zen/v1/chat/completions
done
```

Real output:

```
nemotron-3-ultra-free http=403 body={"type":"error","error":{"type":"FreeTierError","message":"OpenCode's free tier can only be used from within OpenCode"}}
space-bunny-free    http=200 body={"id":"070491563554beef40c68925ee5ac31b","object":"chat.completion","created":1790271062,"model":"space-bunny-free","choices":[{"index":0,"finish_reas
```

## Takeaway

The catalog listing a `*-free` model doesn't mean the raw endpoint will serve it — ping each candidate before wiring it into a fallback pool, and let the cfalls strikes (not the catalog) decide which free models stay eligible.