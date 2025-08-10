# AIWF v12 — Setup Guide

This document walks you through setting up **AIWF v12** from scratch, including installing dependencies, configuring credentials, setting environment variables, and deploying to n8n.

---

## 1. Requirements

Before you begin, ensure you have:

- **n8n** v1.56+ (self-hosted recommended)
- **Node.js** 18+ (for local development of Code nodes)
- **Docker** (optional, for containerized n8n)
- **Redis** (for caching — Upstash, Redis Cloud, or self-hosted)
- API keys for:
  - **OpenRouter** ([openrouter.ai](https://openrouter.ai/))
  - **Perplexity** ([perplexity.ai](https://perplexity.ai/)) *(optional, for web search augmentation)*

---

## 2. Repository Structure

```plaintext
aiwf/
├── aiwf-main.json                # Main n8n workflow
├── subflows/
│   ├── aiwf-subflow-cache.json    # Caching logic
│   ├── aiwf-subflow-rerank.json   # Local reranking logic
│   ├── aiwf-subflow-validate.json # Schema validation + fixup
│   ├── aiwf-subflow-finalize.json # Final shaping of output
├── credentials-stubs/
│   ├── openrouter.credentials.json
│   ├── perplexity.credentials.json
│   ├── redis.credentials.json
├── docs/
│   ├── architecture.md
│   ├── setup.md                   # This file
├── .env.example                   # Sample environment variables
├── README.md
````

---

## 3. Installation

### Option A — Direct Import into n8n

1. **Export your current workflows** *(for backup)*:

   * In n8n, go to **Workflows → All → Export**.

2. **Import AIWF v12**:

   * Go to **Workflows → Import from file**.
   * Import `aiwf-main.json` and all subflows from the `subflows/` folder.

3. **Update Connections**:

   * Replace all credential stubs with your actual n8n **Credentials**.

---

### Option B — Docker Deployment

If running n8n via Docker:

```bash
git clone https://github.com/yourusername/aiwf-v12.git
cd aiwf-v12
cp .env.example .env
# Edit .env with your values
docker-compose up -d
```

---

## 4. Environment Variables

Copy `.env.example` to `.env`:

```bash
cp .env.example .env
```

Edit `.env` and fill in:

| Variable             | Description                                 |
| -------------------- | ------------------------------------------- |
| `CHEAP_MODEL`        | Low-cost model for routing & simple tasks   |
| `STD_MODEL`          | Standard model for most solves              |
| `PREM_MODEL`         | Premium model for high-risk/complex queries |
| `OPENROUTER_API_KEY` | OpenRouter API key                          |
| `PPLX_API_KEY`       | Perplexity API key *(optional)*             |
| `REDIS_BASE_URL`     | Redis REST endpoint                         |
| `REDIS_AUTH_HEADER`  | Redis REST token                            |
| `CACHE_TTL_SEC`      | Cache TTL in seconds                        |
| `PREMIUM_RATE_CAP`   | Max premium usage fraction (0.0–1.0)        |
| `COST_CAP_DAILY_USD` | Max daily spend for premium requests        |

---

## 5. Credential Setup in n8n

In **n8n → Credentials**:

### OpenRouter

* Type: HTTP Request
* Base URL: `https://openrouter.ai/api/v1`
* Header: `Authorization: Bearer {{ $env.OPENROUTER_API_KEY }}`

### Perplexity *(optional)*

* Type: HTTP Request
* Base URL: `https://api.perplexity.ai`
* Header: `Authorization: Bearer {{ $env.PPLX_API_KEY }}`

### Redis

* Type: HTTP Request
* Base URL: `{{ $env.REDIS_BASE_URL }}`
* Header: `Authorization: {{ $env.REDIS_AUTH_HEADER }}`

---

## 6. Workflow Overview

The AIWF v12 workflow runs in **6 main stages**:

1. **Router** → Picks model tier (cheap, standard, premium) based on query heuristics.
2. **Cache Check** → Looks up content-addressed key in Redis.
3. **Retrieval + Local Rerank** → Uses local cosine/Jaccard scoring to keep top N chunks.
4. **Plan → Solve** → Two-step reasoning: cheap model plans, better model executes.
5. **Validation + Fixup** → Schema validation; if invalid, minimal fixup using cheap model.
6. **Finalize & Cache Store** → Output shaping, clamping, and caching.

---

## 7. Running the Workflow

* Trigger the workflow via **n8n Webhook**, **Cron**, or **Manual Execution**.
* Monitor execution times and token usage in n8n’s **Executions** view.
* Adjust `.env` limits (e.g., `PREMIUM_RATE_CAP`) to control costs.

---

## 8. Testing

Run regression tests:

1. Create a **test data** node with 50–200 sample queries.
2. Compare AIWF v12 output against known-good outputs.
3. Watch for:

   * Schema drift
   * Token usage spikes
   * Cache hit/miss patterns

---

## 9. Updating AIWF

To update:

```bash
git pull origin main
cp .env .env.backup
cp .env.example .env   # Only if new vars were added
# Re-import workflows in n8n
```

---

## 10. Troubleshooting

| Issue              | Likely Cause                 | Fix                               |
| ------------------ | ---------------------------- | --------------------------------- |
| High costs         | `PREMIUM_RATE_CAP` too high  | Reduce to 0.05 or less            |
| Slow responses     | Premium tier calls too often | Tune router heuristics            |
| Cache not working  | Redis credentials wrong      | Verify in n8n Credentials         |
| Broken JSON output | Schema validator disabled    | Re-enable `aiwf-subflow-validate` |

---

## 11. Next Steps

* Set up **nightly regression tests** in n8n Cron.
* Monitor Redis usage and eviction rates.
* Track premium usage ratio; adjust thresholds for cost-performance balance.

---

**Author:** Omar
**Version:** AIWF v12 (Optimized for cost, speed, and robustness)
