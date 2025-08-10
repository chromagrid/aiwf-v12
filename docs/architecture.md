# AIWF v12 — Architecture

## Goals
- Keep features. Raise answer quality. Cut token and API spend.
- Deterministic outputs via schema. Minimal retries. Cache-first.
- Modular subflows to reduce node count and keep maintenance cheap.

---

## High-level flow
1) **HTTP Webhook** (`POST /aiwf/v12`)  
2) **Normalize** input → `path`, `q`, `locale`, `policyVersion`, `catalogIds`, `allowWeb`  
3) **CacheKey** (content-addressed SHA256)  
4) **Cache GET** → short-circuit on hit  
5) **Guards**: input size, (your) rate limit slot  
6) **Router**: pick `tier` + model + budgets (cheap|standard|premium, premium-capped)  
7) **Catalog Fetch** (local/Redis or your store)  
8) **Rerank & Dedupe** (local TF-cosine + Jaccard; top N)  
9) **Adaptive Context Budget** (short/std/long cap)  
10) **Plan** (cheap model; steps + needed facts)  
11) **Solve** (std/prem model; schema-first JSON)  
12) **Validator subflow** (fast check → cheap fixup if invalid)  
13) **Finalize subflow** (clamp + pack response)  
14) **Cache SET** (SETEX)  
15) **Metrics subflow** (micro-events)  
16) **Bandit subflow** (cost-aware win counters)  
17) **Respond** (JSON)

---

## Components
- **Main workflow**: end-to-end orchestrator.  
- **Subflows**:
  - `aiwf-subflow-cache.json` — GET/SET/DEL wrapper for Redis REST.
  - `aiwf-subflow-rerank.json` — local rerank + semantic dedupe.
  - `aiwf-subflow-validate.json` — schema validation + cheap fixup.
  - `aiwf-subflow-finalize.json` — clamp + cache write + final shape (optional if you prefer inline Finalize).
  - `metrics` + `bandit` (lightweight counters).
- **Credentials** (stubs only): OpenRouter, Perplexity (optional), Redis REST.

---

## Data contracts (JSON shapes)

### Request (Webhook POST body)
```json
{
  "path": "search|qa|…",
  "q": "user query",
  "locale": "en",
  "policyVersion": "v1",
  "catalogIds": ["..."],
  "allowWeb": false
}
````

### Router output

```json
{
  "tier": "cheap|standard|premium",
  "model": "openrouter/…",
  "max_tokens": 384|768|1024,
  "temperature": 0.2|0.3,
  "reason": { "len": 1234, "risk": true }
}
```

### Rerank output

```json
{
  "context": [{ "id": "X", "text": "…", "source": "…", "url": "…", "_score": 0.73 }],
  "kept": 8,
  "dropped_duplicate": 3,
  "dropped_lowscore": 10
}
```

### Plan output

```json
{ "steps": ["…","…"], "needed": ["…","…"] }
```

### Solve output (strict schema)

```json
{ "answer": "string", "citations": ["url1","url2"] }
```

### Final response

```json
{
  "answer": "string",
  "citations": ["url"],
  "model": "openrouter/…",
  "tier": "cheap|standard|premium",
  "cache_hit": false
}
```

---

## Cache strategy

* **Key**: SHA256 of `{path,q(normalized lower/trim),locale,catalog(sorted),policy}`
* **GET** before any LLM call. **SETEX** after Finalize.
* **TTLs**: 15–60 min public; 24h doc-heavy (set via `CACHE_TTL_SEC`).
* **Content normalization**: lower/trim, collapse whitespace, stable sort of IDs.

---

## Routing (tiered, cost-aware)

* **Heuristics**:

  * Length thresholds (e.g., `len>900 → standard`, `risk && len>1600 → premium`).
  * Risk keywords: `refund|legal|medical|pricing|policy|deadline`.
  * Premium hard cap via `PREMIUM_RATE_CAP` or `PREMIUM_BLOCK=1`.
  * Daily cost guard reads `rate:<path>:daily_cost_usd` and downgrades when ≥ `COST_CAP_DAILY_USD`.
* **Models** (env switchable):

  * Cheap: `llama-3.1-8b-instruct` or `qwen2.5-7b-instruct`
  * Standard: `llama-3.1-70b-instruct`, `qwen2.5-72b-instruct`, `deepseek-v3`
  * Premium: `gpt-4.1` or `claude-3.5-sonnet` (if enabled)
* **Determinism**: `temperature 0.2–0.3`, `top_p 0.7` (if supported), strict `max_tokens` per route.

---

## Rerank & dedupe (local, zero API)

* **Scoring**: TF-cosine over tokens ≥3 chars.
* **Duplicate drop**:

  * Same `id` OR Jaccard similarity > 0.90.
  * Keep **top N** (`RERANK_TOP`, default 8).
* **Outcome**: 25–50% fewer context tokens with equal/better relevance.

---

## Plan → Solve

* **Plan (cheap)**: produce `steps` + `needed` concisely.
* **Solve (std/prem)**: feed **only** the kept context and needed facts.
* **Benefit**: tighter prompts, fewer wasted tokens, clearer structure.

---

## Validation & fixup

* **Validator subflow**:

  * Try to parse LLM output as `{answer, citations?}`.
  * If invalid: one **cheap** fixup call with strict “output JSON only” instruction.
  * No blind retries; only on structure failure.

---

## Finalize & clamp

* **Clamp**:

  * `answer` length → `MAX_ANSWER_LEN` (default 4000 chars).
  * `citations` count → `MAX_CITATIONS` (default 10).
* **Response**: `{answer,citations,model,tier,cache_hit:false}`.
* **Cache SET**: write using the computed key and `CACHE_TTL_SEC`.

---

## Credentials (n8n)

* **OpenRouter**: HTTP Header Auth → `Authorization: Bearer {{$env.OPENROUTER_API_KEY}}`
* **Perplexity** (optional): HTTP Header Auth → `Authorization: Bearer {{$env.PPLX_API_KEY}}`
* **Redis REST**: HTTP Header Auth → `Authorization: {{$env.REDIS_AUTH_HEADER}}`
* **Never** embed secrets in JSON nodes.

---

## Environment variables (.env)

* **Models**: `CHEAP_MODEL`, `STD_MODEL`, `PREM_MODEL`
* **Caps**: `PREMIUM_RATE_CAP`, `PREMIUM_BLOCK`, `COST_CAP_DAILY_USD`
* **Cache/Rerank**: `CACHE_TTL_SEC`, `RERANK_TOP`
* **Token budgets**: `MAX_TOKENS_SHORT`, `MAX_TOKENS_STD`, `MAX_TOKENS_LONG`
* **Clamp**: `MAX_ANSWER_LEN`, `MAX_CITATIONS`
* **HTTP**: `HTTP_TIMEOUT_MS`, `HTTP_MAX_RETRIES`
* **Redis**: `REDIS_BASE_URL`, `REDIS_AUTH_HEADER`, namespaces (`REDIS_RESP_NS`, `REDIS_RATE_NS`, `REDIS_BANDIT_NS`, `REDIS_LOG_NS`)
* **CORS**: `CORS_ORIGIN`, `CORS_MAX_AGE`
* **Debug**: `DEBUG`

---

## Observability (micro-events)

* Emit one JSON per request:

```json
{
  "ts": 1730000000000,
  "route": "path",
  "tier": "standard",
  "model": "openrouter/…",
  "kept": 8,
  "cache_hit": false,
  "latency_ms": 1234,
  "retries": 0
}
```

* Store via `LPUSH ${REDIS_LOG_NS}` + `LTRIM` to 1k records.

---

## Bandit nudging

* Hash per tier: `${REDIS_BANDIT_NS}${tier}` with fields `{win,loss}`.
* Router may bias toward the cheapest tier that meets SLA.
* No heavy math; slow drift only.

---

## Error handling

* **Guard errors**: 4xx with `{error:'EMPTY_QUERY'|'TOO_LARGE'}`.
* **Vendor**: one retry max on 408/429/5xx; exponential backoff (vendor or node).
* **Schema failures**: go to **Validator fixup** once; else return shape with minimal message.

---

## Cost controls

* **Cache-first** → serve zero-cost on hit.
* **Adaptive context** + **Rerank** → cut input tokens 25–50%.
* **Prompt compression**: minimal system prompts; no meta talk.
* **Daily cap** via `rate:<path>:daily_cost_usd` check → downgrade tier.
* **Premium cap** via `PREMIUM_RATE_CAP` or hard `PREMIUM_BLOCK=1`.

---

## Security

* No secrets in workflows. Use n8n Credentials + env only.
* Clamp outputs; never echo request secrets.
* Optional allow-list on `path`/`locale`.

---

## Testing & regression

* Create 50–200 canned inputs per route.
* Validate:

  * Response is valid JSON, has `answer`, optional `citations[]`.
  * Contains required keywords for golden tests.
* Run nightly via n8n Cron at off-peak.
* Use **cheap** tier for tests; no external APIs if not needed.

---

## Import & wiring checklist

* [ ] Import **Main** + **Subflows** inactive.
* [ ] Create Credentials (OpenRouter, Redis; Perplexity if used).
* [ ] Set Execute Workflow nodes in Main:

  * Validator → `aiwf-subflow-validate.json` ID
  * Metrics → metrics subflow ID
  * Bandit → bandit subflow ID
* [ ] Fill `.env` values on your n8n host.
* [ ] Activate **Main** and run smoke tests.

---

## Troubleshooting

* **Cache hit never happens**: confirm `REDIS_BASE_URL`, namespaces, and that `CacheKey` normalization matches both GET/SET.
* **Premium overuse**: set `PREMIUM_RATE_CAP` lower or `PREMIUM_BLOCK=1`.
* **Bloated outputs**: reduce `MAX_TOKENS_*`, `MAX_ANSWER_LEN`, and `RERANK_TOP`.
* **Invalid JSON responses**: ensure Validator subflow ID is wired; check cheap fixup call succeeds (OpenRouter creds).

---

## Performance budgets (defaults)

* `RERANK_TOP = 8`
* `MAX_TOKENS_SHORT = 384`, `STD = 768`, `LONG = 1024`
* `temperature = 0.2–0.3`
* Cache TTL = 1800s (30 min)

---

## Extensibility

* Add a `Policy` gate before Router for compliance.
* Plug a **Perplexity retrieval** branch only when `allowWeb=true` and catalog misses; pass results through Rerank path.
* Swap models via env without touching nodes.
