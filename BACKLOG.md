# TIL Backlog

Topics reserved for future entries. The daily cron (Mon–Fri) consumes the top unchecked item whose date is due — if a topic needs its own date, it stays here until then.

| Date | Topic | Notes |
|------|-------|-------|
| ✓ 2026-08-17 | Electron chrome-sandbox needs `root:root` mode `4755` — and the skip-if-already-correct gate | Systemd user unit failed mid-update on a no-passwordless-sudo host; gate on file state instead of sudo; `systemctl --user reset-failed` |
| ✓ 2026-08-18 | No orchestrator: one model, in-context routing | Every message lands on the default model; simple-vs-complex routing is decided inside the agent loop, not by a pre-classifier; delegation for heavy work |
| ✓ 2026-08-19 | CDN blocks the HTML but forgets the JSON API | Retail site: HTML page 403s behind Akamai, but the `/api/v2/json/search` endpoint works with a browser UA + `Accept: application/json` — the API behind a CDN-protected page |
| ✓ 2026-08-20 | Fail-closed validation when nothing to scan | validate-config.sh `--check-refs-only` with no scan targets must exit 1 (false green) not 0 |
