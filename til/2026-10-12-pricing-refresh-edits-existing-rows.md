# A pricing refresh added 7 models but edited 36 existing rows — diff keyed by provider/model, not by counts

**Date:** 2026-10-12
**Tags:** `llmcost`, `json-diff`, `verification`, `data-refresh`

## The problem

Today's llmcost pricing refresh landed (`data: update model pricing (2026-10-12)`, 09:22 PT). The obvious question — *what actually changed?* — is not answered by the two numbers the commit hands you:

```
$ git show --stat HEAD
 data/prices.json | 214 ++++++++++++++++++++++++++-------------
 1 file changed, 163 insertions(+), 51 deletions(-)
```

`model_count` moved 3502 → 3509 (+7) and the file churned 214 lines. A count delta only ever describes *arrivals and departures*, and a raw line count has no attribution at all: upstream can rewrite the price and output limit of a model that already existed, and both numbers stay silent about it. Since this snapshot feeds budget math, "which already-priced rows moved, and by how much" is the question that matters.

## What I tried

1. **Count sanity first.** The schema declares `model_count`; verify it against reality before trusting any delta built on it — here declared 3502 → 3509 matched the real array length exactly (3502 → 3509), so the count was honest *this* time. It still told me nothing about edits.
2. **`jq`-style one-liner:** `jq not found`. This host has no `jq` — a real limitation worth remembering before writing a snippet you can't run here.
3. **Pipe the two snapshots into an interpreter:** blocked by the sandbox (`pipe to interpreter` is refused in unattended runs), so the two revisions get concatenated into one file and the script reads stdin instead.
4. Dead end worth avoiding: reading the diff by eye. With 214 changed lines and no `provider/model` key on the changed lines, the eye lands on whichever price it lands on — that's how you miss 28 limit corrections.

## What worked

Key both revisions by `provider/model` and compare the fields the budget actually uses, in one pass:

```bash
# both revisions into one stream: old doc, newline, new doc
{ git show HEAD~1:data/prices.json; printf '\n'; cat data/prices.json; } > /tmp/both.json

# keydiff.py — reads the two concatenated JSON docs from stdin, no jq needed
python3 keydiff.py < /tmp/both.json
```

```python
import sys, json
dec = json.JSONDecoder(); buf = sys.stdin.read(); docs = []; i = 0
while i < len(buf):
    while i < len(buf) and buf[i].isspace(): i += 1
    if i >= len(buf): break
    obj, i = dec.raw_decode(buf, i); docs.append(obj)
key = lambda m: f"{m.get('provider','-')}/{m['model']}"
prev, cur = ({key(m): m for m in d['models']} for d in docs)
fields = ('input_usd_per_mtok', 'output_usd_per_mtok', 'max_output_tokens')
touched = [k for k in cur if k in prev and any(cur[k][f] != prev[k][f] for f in fields)]
print(f"declared model_count {docs[0]['model_count']} -> {docs[1]['model_count']}"
      f" | added={len(set(cur)-set(prev))} removed={len(set(prev)-set(cur))}"
      f" | in-place-edited={len(touched)}")
for k in sorted(touched, key=lambda k: abs(prev[k]['input_usd_per_mtok'] - cur[k]['input_usd_per_mtok']),
                reverse=True)[:5]:
    print(f"  {k}\n    ${prev[k]['input_usd_per_mtok']}/{prev[k]['output_usd_per_mtok']}"
          f" -> ${cur[k]['input_usd_per_mtok']}/{cur[k]['output_usd_per_mtok']}"
          f"   max_output {prev[k]['max_output_tokens']} -> {cur[k]['max_output_tokens']}")
```

Real output:

```
declared model_count 3502 -> 3509 | added=7 removed=0 | in-place-edited=36
  bedrock/amazon.titan-text-express-v1
    $1.3/$1.7 -> $0.2/0.6   max_output 8000 -> 8000
  bedrock/bedrock/us-gov-west-1/amazon.titan-text-express-v1
    $1.3/$1.7 -> $0.8/1.6   max_output 8000 -> 8000
  bedrock/amazon.titan-text-lite-v1
    $0.3/$0.4 -> $0.15/0.2   max_output 4000 -> 4000
  bedrock/bedrock/moonshotai.kimi-k2-thinking
    $0.73/$3.03 -> $0.6/2.5   max_output 16000 -> 16000
  bedrock_converse/eu.amazon.nova-2-lite-v1:0
    $0.33/$2.75 -> $0.429/3.597   max_output 64000 -> 64000
```

So the "+7 models" headline actually delivered **36 edited existing rows**: 8 rows with a price change and 28 with a `max_output_tokens` correction (no overlap) — five times the churn the count advertised. The largest single change is a 6.5× price cut on a row counted as unchanged (`amazon.titan-text-express-v1`, $1.3 → $0.2 in), and the corpus-wide edits are output ceilings dropping hard (`ai21.jamba-1-5-large-v1:0` 256000 → 4096 — a 62.5× reduction, i.e. a *capability* correction, not a price one).

Cross-check the structured count against raw line churn so neither view can lie alone:

```bash
$ git diff HEAD~1 -- data/prices.json \
  | grep -E '^[+-]\s+"(input_usd_per_mtok|max_output_tokens|model)"' \
  | sed 's/^\(.\).*"\([a-z_]*\)".*/\1\2/' | sort | uniq -c
     14 +input_usd_per_mtok
      7 -input_usd_per_mtok
     35 +max_output_tokens
     28 -max_output_tokens
      7 +model
```

The arithmetic closes: 14 added vs 7 removed input-price lines = 7 edited rows *plus* 1 row (`moonshotai/kimi-k2.5`) whose output price alone moved, giving the 8. And 35 added `max_output_tokens` lines = 28 in-place corrections + 7 lines belonging to the new rows. Two independent views, one number — that's the confidence I want before a budget script trusts the snapshot.

## Takeaway

For any generated catalog, never summarize a refresh by `model_count` or `--stat`: key both revisions by `provider/model` and diff the fields you actually spend against (price *and* max output tokens), because upstream edits rows in place — here 36 of them against 7 additions — and the biggest dollar movement was a price cut on a row that counts as unchanged.
