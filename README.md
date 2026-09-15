# Hermes Corpus Reliability Engineering

> **The LLM took ~6 seconds. The pipeline took ~4 minutes.**
>
> A production case study on making a local LLM Agent–Corpus pipeline trustworthy, durable, observable, and fast.

## Overview

This repository documents a real production reliability-engineering effort on a fully local LLM Agent + Corpus ingestion pipeline built around:

- Hermes Agent
- Local Qwen 27B Q6 model
- Ollama
- SQLite durable state
- Hermes native Cron scheduler
- Windows workstation
- RTX 5090 32 GB VRAM
- 192 GB RAM

The project started with what looked like a tiny bug:

```text
他
" he"
---

## Full Paper

📄 **[Read the full technical paper →](paper.md)**

**Key production result:**  
`155.273 s → 0.328 s` post-resource-defer continuation latency, while preserving the final ingestion result.
