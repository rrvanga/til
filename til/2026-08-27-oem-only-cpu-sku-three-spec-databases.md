# An OEM-only CPU has no spec page on the vendor's site — triangulate three spec databases

**Date:** 2026-08-27
**Tags:** `hardware-research`, `cpus`, `verification`, `oem`, `web`

## The problem

Researching a mini-PC chip that is an **OEM-only SKU** (no retail box): every product page
says "8 cores, Zen 4, Radeon 780M", but marketing copy is not a spec sheet. AMD's own site
has no product page for the chip at all. How do you verify specifications when the vendor
does not document the part?

## What I tried

- AMD's official product pages (7000-series and AI 200-series URL paths) → 404 / empty.
- `site:amd.com` search → only an AMD forum thread confirming the chip exists and is paired
  with a Radeon 780M iGPU. Existence confirmed; specs not documented.
- Spec aggregators (technical.city, topcpu.net, cputronic, notebookcheck) → several failed
  from this network (rate limits, keyless extraction-backend errors), so the three agreeing
  databases had to be gathered one by one, not in a single pass.

## What worked

Treat "no vendor page" as the signal that **a single source is never enough**. The bar that
cleared the chip: **three independent spec databases agreeing** on every number —

- 4nm, Zen 4 family, 8C/16T, 3.8 → 4.9 GHz, 45W, 16 MB L3
- Radeon 780M (RDNA3, 12 CU) — the iGPU pairing that matters most to mini-PC buyers

The retail listing's own title ("Ryzen 7 H 255 with Radeon 780M") counts only as
*corroboration*, never as proof.

## Takeaway

No vendor page = no single source of truth. For OEM-only chips, require at least three
agreeing spec databases, and treat marketing copy as intent, not evidence.