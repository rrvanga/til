# archify receipts commit absolute paths — scrub committed JSON evidence to repo-relative before pushing

**Date:** 2026-09-29
**Tags:** `archify`, `pii`, `verification`

## The problem

The archify prototype receipts (`deliver-receipt.json`, `visual-check.json`) committed to the public agent-lab repo on PR #27 contained absolute host paths — `/home/<user>/dev/agent-lab/...` in `input`/`output`/`artifact.path` and `/usr/bin/google-chrome-stable` in `chrome.executable`. A PII sweep flagged them: committing machine-specific paths leaks the username and layout of the build machine into a public repo, and the chrome binary path bakes in distro-specific state that isn't part of the evidence.

## What I tried

- Checked whether archify has a relative-path or `--repo-root` option for receipts — it doesn't. The tool hardcodes `path.resolve()` when writing the receipt's path fields (verified at `archify.mjs:880` for `input` and `:893` for `output`), so the absolute paths are generated every run with no flag to suppress them.
- Decided not to wrap the call with `realpath`-style tricks (they'd fight the tool) — instead, treat the committed JSON as a deliverable and sanitize it before commit, like any other generated artifact.

## What worked

Rewrite just the path-valued fields to repo-relative paths, keep every evidence field (sha256, bytes, validation) byte-identical, then verify the result parses and contains no absolute paths before committing:

```bash
cd ~/dev/agent-lab
git show 79f57d0 -- docs/experiments/archify/   # before → after
#  - "input": "/home/<user>/dev/agent-lab/docs/experiments/archify/agent-setup.architecture.json"
#  + "input": "docs/experiments/archify/agent-setup.architecture.json"
#  + "chrome.executable": "google-chrome-stable"  (binary name only, no /usr/bin/ prefix)

node -e '                                       # parse check + field dump
const fs = require("fs");
const r = "docs/experiments/archify/agent-setup.architecture.deliver-receipt.json";
const v = "docs/experiments/archify/agent-setup.architecture.visual-check.json";
for (const f of [r, v]) JSON.parse(fs.readFileSync(f, "utf8"));
const d = JSON.parse(fs.readFileSync(r, "utf8"));
const c = JSON.parse(fs.readFileSync(v, "utf8"));
console.log("both receipts parse OK");
console.log("deliver.input  =", d.input);
console.log("visual.path    =", c.artifact.path);
console.log("visual.chrome  =", c.chrome.executable);
'
# both receipts parse OK
# deliver.input  = docs/experiments/archify/agent-setup.architecture.json
# visual.path    = docs/experiments/archify/agent-setup.architecture.html
# visual.chrome  = google-chrome-stable

grep -rnE '"(input|output|path|executable)"[^/]*"/(home|usr|tmp)' docs/experiments/archify/ --include='*.json'
echo "absolute_path_grep_exit=$? (1 = none found)"   # → 1, no absolute path values remain
```

Committed as `79f57d0` — receipts re-validated, sha256/bytes/9-9 validation untouched, PII grep clean.

## Takeaway

Generated evidence files are code too: if the generator hardcodes `path.resolve()` output (no relative-path flag), sanitize path-valued fields to repo-relative before committing to a public repo, and gate the commit on a grep for `/(home|usr|tmp)` — otherwise every run leaks the build machine's username and layout.