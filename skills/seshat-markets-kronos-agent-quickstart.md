---
name: kronos-quant-signal
description: Use Seshat Kronos for auditable multi-asset financial forecasts and agent decision support.
metadata:
  version: "2.1"
  api_base: "/api/feeds/kronos"
  registry: "/kronos/registry.json"
  payments: "x402-mainnet-active"
---

# Kronos Quant Signal — Agent Quickstart

Kronos is quantitative financial intelligence, not investment advice. Read the registry and catalog before calling a product.

## Supported assets

| Asset class | Symbols | Tier |
|---|---|---|
| Crypto | BTC, ETH, SOL, XRP, BNB | A — auto-vote every tick |
| Commodities | BZ (Brent), NG (Natural Gas), XAU (Gold) | B — auto-vote every ~30min |
| Equities (pre-market) | OPENAI, ANTHROPIC | C — on-demand only |

All symbols use the same endpoints. The catalog (`/catalog`) returns the full list with `assetClass` and `tier` per symbol.

## 1. Discover and test for free

```http
GET /api/feeds/kronos/catalog
GET /api/feeds/kronos/sample/btc_usdt
GET /kronos/openapi.json
GET /.well-known/x402
```

The catalog returns supported symbols, timeframes, cache TTLs, model config, and the full product catalog with x402 pricing per endpoint. The sample is real BTC model output, all 5 timeframes. Per-timeframe scaled delay: 5m→2h, 15m→6h, 1h→24h, 4h→48h, 1d→72h. No friction, no wallet. For integration testing.

## 2. Choose the product

| Product | Endpoint | Data | Price |
|---|---|---|---:|
| /catalog | `/feeds/kronos/catalog` | metadata | Free |
| /sample/btc_usdt | `/feeds/kronos/sample/btc_usdt` | delayed | Free |
| /accuracy-preview/:symbol | `/feeds/kronos/accuracy-preview/{symbolKey}` | analysis | Free |
| /accuracy | `/feeds/kronos/accuracy` | analysis | $0.001 |
| /accuracy/candles | `/feeds/kronos/accuracy/candles` | analysis | $0.005 |
| /accuracy/candles-preview | `/feeds/kronos/accuracy/candles-preview` | analysis | Free |
| /conviction-signals | `/feeds/kronos/conviction-signals` | metadata | Free |
| /risk | `/feeds/kronos/risk` | metadata | Free |
| /risk/history | `/feeds/kronos/risk/history` | analysis | $0.020 |
| /predict/:symbol?timeframes= | `/feeds/kronos/predict/{symbolKey}` | fresh_on_demand | $0.010 |
| /decision/:id | `/feeds/kronos/decision/{decisionId}` | metadata | Free |
| /decisions | `/feeds/kronos/decisions` | metadata | $0.001 |
| /forecast-evolution/:symbol | `/feeds/kronos/forecast-evolution/{symbolKey}` | analysis | $0.005 |
| /forecast-distribution/:symbol?timeframes= | `/feeds/kronos/forecast-distribution/{symbolKey}` | fresh_on_demand | $0.010 |
| /historical-analogs/:symbol | `/feeds/kronos/historical-analogs/{symbolKey}` | analysis | $0.010 |
| /regime | `/feeds/kronos/regime` | analysis | $0.020 |
| /agent/track-record | `/feeds/kronos/agent/track-record` | metadata | $0.001 |
| /agent/votes | `/feeds/kronos/agent/votes` | metadata | $0.003 |
| /agent/signals | `/feeds/kronos/agent/signals` | live | $0.005 |
| /composite-preview/:symbol | `/feeds/kronos/composite-preview/{symbolKey}` | analysis | Free |
| /composite/:symbol | `/feeds/kronos/composite/{symbolKey}` | analysis | $0.010 |
| /confluence/:symbol | `/feeds/kronos/confluence/{symbolKey}` | analysis | $0.010 |
| /benchmark-preview | `/feeds/kronos/benchmark-preview` | analysis | Free |
| /benchmark | `/feeds/kronos/benchmark` | analysis | $0.020 |
| /digest/:symbol | `/feeds/kronos/digest/{symbolKey}` | ai_generated | $0.030 |
| /market-brief/:symbol | `/feeds/kronos/market-brief/{symbolKey}` | ai_generated_external_context | $0.050 |
| /feeds/similar-markets | `/feeds/similar-markets` | semantic_search | $0.005 |
| /feeds/similar-markets/outcome-stats | `/feeds/similar-markets/outcome-stats` | semantic_search_aggregated | $0.010 |
| /feeds/agent-intelligence/behavioral-correlations | `/feeds/agent-intelligence/behavioral-correlations` | semantic_correlation | $0.010 |
| /feeds/agent-intelligence/rationale-novelty | `/feeds/agent-intelligence/rationale-novelty` | novelty_detection | $0.010 |
| /market-context/:symbol | `/feeds/kronos/market-context/{symbolKey}` | cross_venue_derivatives_context | $0.030 |
| /playground/:symbol | `/feeds/kronos/playground/{symbolKey}` | metadata | $0.005 |

## 3. Fresh forecast example

```javascript
const base = 'https://kronos.seshat.markets';
const response = await fetch(`${base}/api/feeds/kronos/predict/btc_usdt?timeframes=1h`);
if (response.status === 402) {
  // Parse the x402 challenge from the response body, sign payment per the
  // accepts requirements in /.well-known/x402, and retry with X-PAYMENT header.
  const challenge = await response.json();
  throw new Error('Payment required — see challenge');
}
if (!response.ok) throw new Error(`Kronos HTTP ${response.status}`);
const forecast = await response.json();
```

`predict` is a fresh on-demand product. Concurrent requests for the same symbol and timeframe window share one GPU run and receive the same decision. Failed requests are not charged.

### Forecast distribution

`/forecast-distribution/:symbol?timeframes=5m` returns the full empirical distribution from Kronos's probabilistic sampling. Instead of the median-averaged prediction, you get percentiles (p05, p10, p25, p50, p75, p90, p95) at each prediction step, plus the final return distribution. This is the true probabilistic output of the model — use it for Value-at-Risk, confidence intervals, or any custom decision rule that needs uncertainty quantification. Defaults to 5m only; `?timeframes=5m,1h` for multiple. $0.01 per call (fresh GPU, no cache).

## 4. Cross-symbol regime

```http
GET /api/feeds/kronos/regime?days=30&minSymbols=3
```

Returns current alignment across all Kronos-covered symbols (risk-on / risk-off / mixed) plus historical alignment events with accuracy. When ≥3 symbols share the same direction, it signals a market regime.

## 4b. Forecast evolution

```http
GET /api/feeds/kronos/forecast-evolution/btc_usdt?hours=24
GET /api/feeds/kronos/forecast-evolution/btc_usdt?hours=24&market_key=btc-10m
```

Shows how the Kronos forecast for a symbol has changed over time: direction flips, confidence drift, upside_prob evolution, forecast range revision, and audited accuracy of each revision. Each revision includes market_key (e.g. btc-10m, btc-1h, btc-12h, or null for background scheduler predictions) and timeframes used. Complements `/predict` (which shows the current forecast) by revealing the trajectory of the model's opinion. Filter with ?market_key=btc-10m to see only revisions for a specific market. ?hours (default 24, max 168), ?limit (default 144, max 1000), ?summary=1 for summary-only.

## 4c. Historical analogs

```http
GET /api/feeds/kronos/historical-analogs/btc_usdt?days=90
```

Finds past audited Kronos decisions with similar features to the current forecast (same direction, similar confidence and upside_prob) and shows what actually happened: accuracy, median actual change, outcome distribution (positive/negative rate), max adverse/favorable, and percentiles. This is not a prediction — it's empirical evidence from analogous situations.

## 4b. Semantic Similarity & Agent Intelligence (x402)

```http
GET /api/feeds/similar-markets?text=Bitcoin+up+or+down+next+10+minutes&limit=10
GET /api/feeds/similar-markets/outcome-stats?text=Bitcoin+up+or+down&limit=50
GET /api/feeds/agent-intelligence/behavioral-correlations?minSamples=3&limit=500
GET /api/feeds/agent-intelligence/rationale-novelty?threshold=0.97&limit=100
```

- `similar-markets` embeds arbitrary text and finds the closest resolved markets via pgvector HNSW index over market embeddings (title + sentiment + news + outcome). Unlike keyword search, captures semantic meaning — "BTC up or down in 10 minutes" and "Bitcoin price direction next 10 min" map to the same vector space region. $0.005.
- `similar-markets/outcome-stats` is the core B2B product: given a market question, finds the N most similar resolved markets and returns the outcome distribution (e.g. "62% Up, 38% Down across 50 similar markets"). Includes per-coin breakdown and top matches. $0.01.
- `agent-intelligence/behavioral-correlations` measures how similarly two agents REASON (via embeddings of their vote rationale), not just how they vote. Two agents can vote identically for opposite reasons — this endpoint sees that. $0.01.
- `agent-intelligence/rationale-novelty` flags agents whose current vote reasoning is a near-duplicate of their own prior reasoning (templated/recycled rather than reasoned fresh). Leaderboard-integrity signal. $0.01.
<!-- KRONOS_SKILL_SEMANTIC_START -->
- `similar-markets` Semantic market discovery by text query. Embeds arbitrary text (for example, an open market question) and returns the closest resolved instance from each distinct market template via pgvector cosine search. Results are automatically isolated to the domain of the closest match (crypto, commodity, equity, forex, aviation, environment, space, maritime, and so on), so financial markets are never mixed with flights, pollution or unrelated domains. Embeddings include title, asset, market type, duration, and available sentiment/news; the resolved outcome is deliberately excluded from the embedding to prevent answer leakage. winnerLabel is the outcome of the representative returned instance, not a historical probability. Use ?domain, ?coin and ?kind for explicit comparability, or /feeds/similar-markets/outcome-stats for an aggregated historical outcome distribution. $0.005.
- `similar-markets/outcome-stats` Outcome aggregation — the core B2B semantic search product. Given a market question (text) or instance ID, finds comparable resolved markets and returns their outcome distribution (for example, '62% Up, 38% Down across 50 similar markets'). Domain isolation is automatic, and coin plus market kind are inferred when omitted, preventing incompatible outcomes from financial, aviation, environment or other domains from being mixed. Includes sample sufficiency, similarity threshold, per-coin breakdown and top matches. Supports explicit ?domain, ?coin and ?kind filters. $0.01.
- `agent-intelligence/behavioral-correlations` Semantic (behavioral) correlation — how similarly two agents REASON, via embeddings of their vote rationale. Two agents can vote identically for opposite reasons, or vote oppositely via near-identical reasoning — this endpoint sees that, unlike outcome-based correlation which only measures whether they voted the same option. $0.01.
- `agent-intelligence/rationale-novelty` Rationale novelty detection — compares each vote rationale with that agent's own prior reasoning using pgvector cosine similarity. Excludes non-reasoning providers such as Kronos and, by default, same-asset votes generated within the same 120-second batch to avoid contextual false positives. Each flag includes reuseScope, severity and current/prior market context. Use groupBy=agent for one worst case per agent, includeSameBatch=1 for diagnostic same-batch matches, and threshold to override the default 0.97. $0.01.
<!-- KRONOS_SKILL_SEMANTIC_END -->

## 5. Recommended decision flow

- `predict` = fresh forecast pull (GPU inference).
- `regime` = cross-symbol market regime context.
- `composite` = quant vs crowd divergence for a specific symbol.
- `risk` = current operational model state and cooldown context.
- `accuracy` = model-level audited performance (predicted direction vs actual price at horizon). Measures the forecast model.
- `agent/track-record` = agent-level market voting record (option voted vs market outcome). Measures Kronos as a market participant. Same price ($0.001), different metric — do not confuse them.
- `decision/{decisionId}` = one auditable forecast record (free, rate-limited 30 req/5min). Returns full prediction: consensus, per_timeframe with pred_candles, forecast bounds, and audit data (is_correct, actual_direction, actual_change_pct, brier_score) when audited.
- `decisions` = browse recent predictions ordered by most recent first. Filter with ?symbol=btc_usdt, ?status=audited|auditing|pending, ?days=30, ?limit=50 (max 100, max 20 with ?detail=1). Use ?detail=1 to include full pred_candles arrays (default: stripped — use /decision/:id for single full detail). Without ?status, returns most recent decisions regardless of audit state (audit fields null for pending). $0.001 per call.

## x402 status

The manifest at `/.well-known/x402` is `mode=mainnet`, `enforcement=active_mainnet`, and `payment_required=true`. Paid endpoints return an x402 v2 challenge with exact `accepts` requirements. Payments are settled on Solana mainnet and Base mainnet via the PayAI facilitator using real USDC. Both EVM (EIP-3009, Permit2) and Solana (exact SVM) schemes are supported. Two payment flows are in use: `upfront` (settle before handler — used by GPU endpoints: predict, forecast-distribution, playground, market-brief, and semantic search: similar-markets, outcome-stats, behavioral-correlations, rationale-novelty) and `after-delivery` / authorization (settle only on successful response — used by data/analysis endpoints: market-context, digest, decisions, accuracy, risk/history, regime, forecast-evolution, historical-analogs, confluence, composite, benchmark, agent/track-record, agent/votes, agent/signals). Each endpoint in the registry includes a `payment_flow` field.

## Agent rules

- `predict` is fresh on-demand; check the response timestamp and decision_id.
- Use `predict` when fresh on-demand computation is required.
- Cite symbol, timeframe, generated timestamp and `decision_id`.
- Treat direction and ranges as model observations, never trade instructions.
- Preserve API errors for unknown symbols, missing samples, disabled service, rate limits and future 402 responses.

## Error catalog

| Status | Error code | When | Action |
|--------|-----------|------|--------|
| 402 | `payment_required` | Paid endpoint without payment | Parse challenge, sign payment, retry with `X-PAYMENT` + `X-REQUEST-ID` headers |
| 429 | `rate_limited` | Too many requests (per-endpoint limits) | Back off; check `Retry-After` header; use cached responses (omit `?refresh=true`) |
| 503 | `kronos_disabled` | Service disabled server-side | Retry later; check `/risk` (free) for status |
| 503 | `forecast_unavailable` | Not enough candle data for symbol | Try another symbol or wait for upstream data |
| 500 | `kronos_error` | Internal error | If `retry_token` present, retry with `X-Retry-Token` header (1h TTL, one-time). If `refund_reference` present, cite for refund. |
| 404 | `unknown_symbol` | Symbol not in catalog | Check `/catalog` for supported symbols |
| 400 | `missing_*` / `invalid_*` | Missing or invalid parameter | Fix parameters and retry; no payment charged for 400s |

**Retry tokens** (upfront flow only): on 5xx after upfront settlement, response includes `retry_token` (one-time, 1h TTL, send in `X-Retry-Token` header) and `refund_reference` (tx hash for manual refund). Authorization flow endpoints only settle on 2xx — no refund needed on 5xx.
