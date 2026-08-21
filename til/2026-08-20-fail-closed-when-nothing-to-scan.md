# A validation script that fails closed when there is nothing to scan

**Date:** 2026-08-20
**Tags:** bash, fail-closed, automation, validation, exit-codes

## The problem

A pre-flight `validate-config.sh --check-refs-only` exits 0 on a clean scan — but what if both target files (`.env` and `config.yaml`) are missing? If it printed "clean" and exited 0 there too, it would silently pass an automated gate without actually checking anything. `A && B` both being false gives a false green, not a false alarm.

## What I tried

The first draft just reported `SKIP: no scan targets found` and fell through to the `exit 0` path. Fine for a human reading the screen, wrong for a gate — it turns "I couldn't check anything" into "everything is fine." The fix is a flag-aware early exit: in `--check-refs-only` mode (the mode an automation gate would call), a missing target set must be a failure.

I also confirmed the fail-open pitfall on the healthy path: a target file *present* but with zero dead refs must still exit 0, so the gate doesn't false-positive on good configs.

## What worked

Branch the exit code on whether any file was scanned, and make the refs-only mode fail closed (tested against a temp `HERMES_HOME` so the real config is never touched):

```bash
# Case A: HERMES_HOME with NO config targets  -> fail-closed, exit 1
HERMES_HOME=/tmp/tiltest/empty bash /tmp/tiltest/validate-config.sh --check-refs-only
# == 4. Dead local-endpoint reference check ==
#   SKIP: no scan targets found (/tmp/tiltest/empty/.env / config.yaml missing)
# exit code = 1

# Case B: clean .env present                  -> exit 0
HERMES_HOME=/tmp/tiltest/clean bash /tmp/tiltest/validate-config.sh --check-refs-only
# dead-endpoint check: clean
# exit code = 0

# Case C: dead local ref present              -> fail, exit 1
HERMES_HOME=/tmp/tiltest/dirty bash /tmp/tiltest/validate-config.sh --check-refs-only
#   dead: .env key HERMES_CUSTOM_LOCALHOST_11434_API_KEY
# fix: remove these dead entries
# exit code = 1
```

The relevant guard, so the "no scan targets" branch never looks like a pass:

```bash
SCANNED=0
for FILE in "$ENV_FILE" "$HERMES_HOME/config.yaml"; do
  [ -f "$FILE" ] || continue
  SCANNED=$((SCANNED + 1))
  # ...scan...
done
if [ "$SCANNED" = "0" ]; then
  echo "  SKIP: no scan targets found"
  [ "$CHECK_REFS_ONLY" = "1" ] && exit 1   # never a false green in gate mode
fi
```

## Takeaway

In a gate, "could not check" must not be reported as "clean" — count whether you actually scanned something, and make no-targets a distinct, fail-closed exit code rather than letting it fall into the success path.
