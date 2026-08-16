# Agent-to-agent communication: what actually works

**Date:** 2026-08-16
**Tags:** `agents`, `protocols`, `messaging`, `automation`

## The problem

I run two agents (mine and a teammate's) that need to exchange messages and state — handoffs, queue items, status pings. The obvious first choice, Telegram, turned out to be a dead end.

## What I tried

1. **Telegram bot-to-bot.** Both agents already live in Telegram, so wiring them together seemed free. Reality: Telegram silently strips bot-to-bot messages — bots can't DM other bots, and the teammate's bot has zero gateway history. Nothing errors; messages just never arrive. Debugging invisible drops is worse than a loud failure.
2. **HTTP endpoint.** A tiny webhook server would work — but that means an open port, a public surface, and credentials on the wire. Rejected: I wanted zero exposure.
3. **Discord.** The common alternative (bot-to-bot works fine there via bots + webhooks) — but it's a whole new platform dependency just to move JSON between two processes I already control.

## What worked

A **private git repository as an asynchronous message queue**. Both agents push to and pull from the same repo; messages are just files.

```bash
# enqueue (my side): timestamped file + commit + push
echo '{"to":"peer","kind":"ping","ts":'"$(date +%s)"'}' > outbox/peer/ping-$(date +%s).json
git add -A && git commit -m "bridge: outbound ping" && git push
```

The pattern:

- **Write:** append timestamped JSON (plus a monotonic id for ordering) to your outbox, commit, push.
- **Read:** `git pull`, process the other side's outbox, delete consumed files, push the ack.
- **Transport:** SSH — already trusted, already configured. No new ports, no HTTP surface, no third party.
- **Cadence:** a 2-minute cron watch on each side; git makes the transport idempotent and resumable — a crashed push just retries on the next tick.

Git-as-queue sounds hacky until you remember it's what the whole open-source world already does with CI: append-only, auditable, conflict-safe, zero infra.

The emerging "proper" answer is the **Agent2Agent (A2A) protocol** — open-sourced by Google, now hosted at the Linux Foundation. Agents advertise themselves via JSON-LD "Agent Cards" and exchange tasks over HTTP (JSON-RPC-style), with support for long-running tasks, streaming, and push notifications. It's designed to pair with MCP: MCP answers "how does an agent use tools?", A2A answers "how do agents talk to each other?". For a two-agent homelab it's heavyweight — you need a reachable endpoint and discovery — but it's the protocol to graduate to when the fleet outgrows a git queue.

## Takeaway

Telegram bot-to-bot traffic is silently dropped — never build agent comms on it; a private git repo makes a zero-exposure async queue with no servers to run, and A2A is the standard to graduate to.
