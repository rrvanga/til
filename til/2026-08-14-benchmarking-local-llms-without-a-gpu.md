# Benchmarking local LLMs without a GPU

**Date:** 2026-08-14
**Tags:** `local-ai`, `benchmarking`, `open-source`

## The problem

I wanted to build an open-source tool around local LLMs — but my current machine can't run the models I'd want to benchmark. "Build a benchmark tool" collides with "I have no hardware to benchmark on."

## What I tried

My first design assumed `bench` meant running `ollama run model` and measuring tokens/sec directly. That's the obvious architecture, and it's exactly the part I couldn't exercise myself.

## What worked

Splitting the tool into three layers that don't all need a GPU:

1. **Estimate (no inference).** A dataset + regression model mapping `{model, quant, vram, ram, gpu}` → `{tokens/sec, quality}`. This runs on a CPU and is the actual value: "know before you download."
2. **Harness (tiny models).** The benchmark *code path* can be validated with a 0.5B quantized model — ~400MB, runs on any CPU. It doesn't prove real-world throughput, but it proves the measurement and output-schema logic are correct.
3. **Crowdsource + seed (no inference).** Real numbers come from community PRs and from normalizing public data — in this case LiteLLM's `model_prices_and_context_window.json`, a ~1,500-model JSON I can fetch, filter, and re-publish without running anything.

The key reframe: the project is an **estimator first, benchmark harness second**. The estimator and the data pipeline need zero local inference, so they're fully buildable today — and the day real hardware arrives, the harness turns me into the first real contributor to my own dataset.

## Takeaway

"Can't run the models" only blocks the *harness*, not the *project* — separate "estimate", "measure", and "source data" into layers, and build the ones that don't need the GPU first.
