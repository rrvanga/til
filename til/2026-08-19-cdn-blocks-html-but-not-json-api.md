# A CDN blocks the HTML page but forgets the JSON API behind it

**Date:** 2026-08-19
**Tags:** `curl`, `cdn`, `http`, `api`, `web-scraping`

## The problem

Scraping a retail storefront with a bare `curl` returns an opaque `403 Access Denied` from the CDN (Akamai), so the page looks untouchable. But the store's own product-search JSON endpoint on the same origin is guarded by looser rules — the page is locked down, the API is not.

## What I tried

A bare `curl https://www.homedepot.com/` returns Akamai's `Access Denied` (`Reference #18…errors.edgesuite.net` — EdgeSuite is Akamai). Reproducing against the JSON API with no headers also fails. The blind spot: a default `curl` sends `User-Agent: curl/8.0` and **no `Accept` header**, and CDN bot rules key on exactly that.

## What worked

Send a browser `User-Agent` **and** `Accept: application/json` — the JSON endpoint answers. Demonstrated with a local mock that mimics the CDN rules, then confirmed the real-world 403:

```bash
# mock CDN: HTML page + JSON API, both blocking non-browser UAs
curl -s -o /dev/null -w "curl/8.0  HTML page :  HTTP %{http_code}\n" \
  "http://127.0.0.1:8931/"          # -> 403
curl -s -o /dev/null -w "curl/8.0  JSON API  :  HTTP %{http_code}\n" \
  "http://127.0.0.1:8931/api/v2/json/search?q=hammer"   # -> 403

# browser UA + Accept: application/json -> API answers
curl -s -w "\n           JSON API  :  HTTP %{http_code}\n" \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/126.0" \
  -H "Accept: application/json" \
  "http://127.0.0.1:8931/api/v2/json/search?q=hammer"
# {"results": [{"name": "hammer", "price": 9.99}]}
#            JSON API  :  HTTP 200

# same idea, live: bare curl to the real storefront HTML -> Akamai 403
curl -s -o /dev/null -w "curl/8.0  homedepot HTML :  HTTP %{http_code}\n" \
  -A "curl/8.0" "https://www.homedepot.com/"   # -> 403
```

## Takeaway

A CDN's bot block lives on HTML/document routes, not always on the same origin's JSON API — retry the endpoint with a browser `User-Agent` plus `Accept: application/json` before giving up, and you'll often get structured data the HTML page hides.
