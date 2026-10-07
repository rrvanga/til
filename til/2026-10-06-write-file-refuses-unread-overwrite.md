# A temp file you only patched can't be overwritten blind — write_file demands a full read, even in /tmp

**Date:** 2026-10-06
**Tags:** `hermes`, `tooling`, `verification`

## The problem

The ops daily-loop writes its MOA review prompt to `/tmp/moa_review_prompt.txt` and regenerates it before each gate run. On 2026-10-06 the write_file call was **refused** ("this task has not seen its full current content (never read, only patched…)"), yet the run summary reported everything green except one verifier footnote. `stat` proved the consequence: the file's mtime was still `2026-10-01 09:05:13` — the overwrite never landed, so the gate ran against a 5-day-old prompt.

## What I tried

- `stat -c '%y' /tmp/moa_review_prompt.txt` → `2026-10-01 09:05:13.372931019 -0700` (size 4694): hard proof the blocked overwrite left the file stale.
- Reproduced the exact history in a scratch file: `write_file` (create) → `patch` (edit) → `write_file` full overwrite with **no `read_file` in between**.

## What worked

The refusal is by design and it's a real tool output:

```text
[write_file] Refusing to overwrite .../til-wf-demo.txt: ... exists but this task has not
seen its full current content (never read, only patched, or only a redacted/partial view).
The file was NOT modified. ... Reload the current contents with read_file ... then call
write_file again. For small edits, prefer patch.

$ stat -c '%y' /tmp/moa_review_prompt.txt
2026-10-01 09:05:13.372931019 -0700
```

Patch history (or an external disk change) does **not** count as having seen the file's current content, so a blind overwrite is blocked file-untouched — even in /tmp. The fix that worked: `read_file` the file in full (every offset/limit page), merge the change, then `write_file` — that succeeded immediately. For small edits use `patch`; for scratch artifacts, write to a fresh path (step- or hash-named) so the stale-copy contract never comes into play.

## Takeaway

In this runtime you may only overwrite a file whose current full content your session has genuinely seen — read it first (or write to a new path), and don't trust a self-reported "✅" over a tool refusal: check the artifact's mtime after the fact.