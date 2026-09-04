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
| 2026-08-19 | [A CDN blocks the HTML page but forgets the JSON API behind it](til/2026-08-19-cdn-blocks-html-but-not-json-api.md) | curl, cdn, http, api, web-scraping |
| 2026-08-20 | [A validation script that fails closed when there is nothing to scan](til/2026-08-20-fail-closed-when-nothing-to-scan.md) | bash, fail-closed, automation, validation, exit-codes |
| 2026-08-26 | [An auto-refresh cron job commits to whatever branch is checked out — not always main](til/2026-08-26-scheduled-commit-lands-on-checked-out-branch.md) | git, automation, cron |
| 2026-08-27 | [An OEM-only CPU has no spec page on the vendor's site — triangulate three spec databases](til/2026-08-27-oem-only-cpu-sku-three-spec-databases.md) | hardware-research, cpus, verification, oem |
| 2026-08-28 | [A single output capture is not a format change — find the conditional in source](til/2026-08-28-one-capture-not-a-format-change.md) | cli, verification, documentation |
| 2026-08-31 | [A daily data-refresh script commits only when content changed, not when the timestamp changed](til/2026-08-31-data-refresh-commit-only-on-content-change.md) | git, automation, cron, data-engineering, bash |
| 2026-09-01 | [rtcwake -m off never returns — arm with -m no, verify the alarm, then power off yourself](til/2026-09-01-rtcwake-m-off-never-returns.md) | rtcwake, power-management, linux, bash, fail-closed |
| 2026-09-02 | [Hermes config set: "not a recognized config key" warning but the set still lands](til/2026-09-02-hermes-config-set-schema-warning.md) | hermes, config, cli |
| 2026-09-03 | [A pricing refresh added 25 US-gov region clones — diff the snapshots, don't trust model_count](til/2026-09-03-pricing-refresh-us-gov-region-clones.md) | llmcost, data-refresh, json-diff |
| 2026-09-04 | [A midnight-spanning night-window predicate admits daytime when start ≤ end — validate the shape, not just the format](til/2026-09-04-midnight-spanning-window-predicate.md) | bash, time-handling, fail-closed |

## Format

Each entry lives in `til/YYYY-MM-DD-slug.md` and follows [template.md](template.md). If you want to submit one, PR it — keep it concrete: the problem, what I tried, what actually worked, and a takeaway you can act on.

## License

[MIT](LICENSE) — the *code* is MIT; the *content* is yours to read and quote.
