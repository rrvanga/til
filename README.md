# TIL — Today I Learned

A collection of concrete, useful things I learn while building and self-hosting — one entry per day, each self-contained enough to be read on its own.

Inspired by [jbranchaud/til](https://github.com/jbranchaud/til). The rule: **one genuinely new thing, written the day I learned it, no filler.**

## Entries

| Date | Title | Tags |
|------|-------|------|
| 2026-08-14 | [Benchmarking local LLMs without a GPU](til/2026-08-14-benchmarking-local-llms-without-a-gpu.md) | local-ai, benchmarking, open-source |
| 2026-08-16 | [Agent-to-agent communication: what actually works](til/2026-08-16-agent-to-agent-communication.md) | agents, protocols, messaging, automation |
| 2026-08-16 | [A silent thermal watchdog on Linux: reading temps from sysfs, not lm-sensors](til/2026-08-16-silent-thermal-watchdog-linux.md) | linux, monitoring, sysfs, automation |
| 2026-08-17 | [Electron's chrome-sandbox must be setuid root:root 4755 — gate on file state, not on sudo](til/2026-08-17-electron-chrome-sandbox-4755.md) | electron, sandbox, systemd, permissions, linux |
| 2026-08-18 | [No orchestrator: one model, in-context routing](til/2026-08-18-no-orchestrator-in-context-routing.md) | agents, llms, routing, config |
| 2026-08-19 | [A CDN blocks the HTML page but forgets the JSON API behind it](til/2026-08-19-cdn-blocks-html-but-not-json-api.md) | curl, cdn, http, api, web-scraping |
| 2026-08-20 | [A validation script that fails closed when there is nothing to scan](til/2026-08-20-fail-closed-when-nothing-to-scan.md) | bash, fail-closed, automation, validation, exit-codes |
| 2026-08-21 | [A config key can be written and whitelisted yet never read — grep for readers before calling it dead](til/2026-08-21-config-key-written-but-never-read.md) | config, dead-code, code-audit |
| 2026-08-24 | [Verify every error string and output example in AI-generated docs against the source](til/2026-08-24-verify-generated-doc-claims-against-source.md) | ai-agents, verification, documentation, grep |
| 2026-08-26 | [An auto-refresh cron job commits to whatever branch is checked out — not always main](til/2026-08-26-scheduled-commit-lands-on-checked-out-branch.md) | git, automation, cron |
| 2026-08-27 | [An OEM-only CPU has no spec page on the vendor's site — triangulate three spec databases](til/2026-08-27-oem-only-cpu-sku-three-spec-databases.md) | hardware-research, cpus, verification, oem |
| 2026-08-28 | [A single output capture is not a format change — find the conditional in source](til/2026-08-28-one-capture-not-a-format-change.md) | cli, verification, documentation |

## Format

Each entry lives in `til/YYYY-MM-DD-slug.md` and follows [template.md](template.md). If you want to submit one, PR it — keep it concrete: the problem, what I tried, what actually worked, and a takeaway you can act on.

## License

[MIT](LICENSE) — the *code* is MIT; the *content* is yours to read and quote.
