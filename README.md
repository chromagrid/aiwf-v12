# AIWF v12 — Advanced Intelligent Workflow Framework

AIWF v12 is a modular, **cost-optimized**, **high-quality** workflow framework for [n8n](https://n8n.io/), designed to deliver structured, validated AI-powered answers while minimizing API spend.  
This version focuses on **robustness**, **result quality**, and **performance efficiency**, with **full caching**, **reranking**, **tiered routing**, and **schema validation**.

---

## ✨ Key Features

- **Cache-first architecture** — instant responses for repeated queries.
- **Tiered routing** — dynamically picks cheap/standard/premium models based on complexity and risk.
- **Local rerank & dedupe** — cut token usage 25–50% without sacrificing relevance.
- **Plan → Solve** pattern — smaller prompts, tighter context, better results.
- **Strict schema validation** — catches malformed outputs and applies cheap fixups.
- **Finalization & clamping** — ensures responses stay within defined limits.
- **Cost controls** — daily and per-tier caps to prevent runaway billing.
- **Metrics & bandit feedback** — lightweight tracking for continuous improvement.
- **Credential isolation** — secrets stored only in n8n Credentials.
- **Environment-configurable** — all key parameters set via `.env`.

---

## 📂 Repository Structure

```

aiwf-v12/
├── workflows/
│   ├── aiwf-main.json                  # Main orchestrator workflow
│   ├── aiwf-subflow-cache.json         # Redis GET/SET subflow
│   ├── aiwf-subflow-rerank.json        # Local rerank & dedupe subflow
│   ├── aiwf-subflow-validate.json      # Schema validation & fixup
│   ├── aiwf-subflow-finalize.json      # Clamp & finalize response
├── credentials-stubs/
│   ├── openrouter.credentials.json
│   ├── perplexity.credentials.json
│   ├── redis.credentials.json
├── docs/
│   ├── architecture.md
├── .env.example
└── README.md

````

---

## ⚙️ Environment Variables

Create a `.env` file from `.env.example` with your values:

```env
# Model configuration
CHEAP_MODEL=llama-3.1-8b-instruct
STD_MODEL=llama-3.1-70b-instruct
PREM_MODEL=gpt-4.1

# API Keys (via n8n Credentials ideally)
OPENROUTER_API_KEY=your_key
PPLX_API_KEY=your_key
REDIS_BASE_URL=https://your-redis-rest-endpoint
REDIS_AUTH_HEADER=Bearer your_redis_token

# Cache settings
CACHE_TTL_SEC=1800
RERANK_TOP=8

# Token budgets
MAX_TOKENS_SHORT=384
MAX_TOKENS_STD=768
MAX_TOKENS_LONG=1024

# Cost controls
PREMIUM_RATE_CAP=0.05
PREMIUM_BLOCK=0
COST_CAP_DAILY_USD=5

# Output limits
MAX_ANSWER_LEN=4000
MAX_CITATIONS=10
````

---

## 🔄 How It Works

1. **Webhook Entry** — Main workflow receives request.
2. **Normalize & CacheKey** — Prepares data and computes a hash key.
3. **Cache GET** — Checks Redis for a hit (returns instantly if found).
4. **Guards** — Enforces input length, rate limits, and allowed paths.
5. **Router** — Chooses the right tier & model based on heuristics.
6. **Catalog Fetch** — Retrieves local or indexed context.
7. **Rerank/Dedupe** — Filters and scores context locally.
8. **Plan** — Cheap model outlines steps & facts needed.
9. **Solve** — Higher-tier model generates answer per schema.
10. **Validate** — Ensures schema compliance; cheap fixup if needed.
11. **Finalize** — Clamps content, sets cache, returns structured JSON.

---

## 🚀 Installation

1. **Clone the repo** or download as ZIP.
2. **Import workflows** into n8n **inactive**:

   * `aiwf-main.json`
   * `aiwf-subflow-cache.json`
   * `aiwf-subflow-rerank.json`
   * `aiwf-subflow-validate.json`
   * `aiwf-subflow-finalize.json`
3. **Create n8n Credentials** for:

   * OpenRouter
   * Redis REST
   * (Optional) Perplexity
4. **Update `.env`** with your configuration.
5. **Wire Execute Workflow nodes** in Main to your subflow IDs.
6. Activate **Main** and test.

---

## 🧪 Testing

* Run with sample queries to confirm:

  * Cache hits are instant
  * Correct model tiers are chosen
  * Schema validation passes
* Use `DEBUG=1` in `.env` to enable verbose logs.

---

## 📊 Observability

* Metrics subflow records:

```json
{
  "ts": 1730000000000,
  "route": "search",
  "tier": "standard",
  "model": "openrouter/llama-3.1-70b-instruct",
  "cache_hit": false,
  "latency_ms": 1100
}
```

* Logs stored in Redis for short-term analysis.

---

## 💰 Cost Optimization

* **Cache-first**: no API calls on repeated queries.
* **Local rerank**: reduces token count for context.
* **Tiered routing**: avoids using expensive models unnecessarily.
* **Daily caps**: stops premium overuse.
* **Adaptive context**: dynamically shortens inputs.

---

## 🔐 Security

* All API keys are stored as **n8n Credentials** or `.env` variables.
* No secrets in workflow JSON.
* Optional allow-lists for paths/locales.

---

## 📌 Recommendations

* Keep `CACHE_TTL_SEC` high for repeated queries.
* Set realistic `COST_CAP_DAILY_USD`.
* Enable Perplexity retrieval only when needed.
* Use `PREMIUM_BLOCK=1` if premium costs are not allowed.

---

## 🛠️ Extending AIWF

* Add compliance gates before Router.
* Plug external retrieval services into Rerank subflow.
* Swap models by changing `.env` only.
* Integrate analytics pipelines for metrics.

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

## 🤝 Contributing

1. Fork repo
2. Create a branch (`feature/my-change`)
3. Commit changes
4. Push and open a PR
