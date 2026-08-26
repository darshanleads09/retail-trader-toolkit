---
goal: Build AI-Powered ScalpMaster Pro — Autonomous NIFTY/BANKNIFTY Options Scalping Agent
version: 1.0
date_created: 2026-08-26
last_updated: 2026-08-26
owner: TraderMan
status: 'Planned'
tags: [feature, architecture, infrastructure, agentic, finance, trading]
---

# Introduction

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

This plan describes the end-to-end implementation of **ScalpMaster Pro** — an autonomous, event-driven AI agent that replicates the 8-indicator scoring logic of the TradingView *AI-Powered ScalpMaster Pro* indicator and executes NIFTY/BANKNIFTY options scalp trades on Zerodha via Kite Connect v3. The system is built on Google Cloud (ADK, Gemini 3.5 Flash/Pro, Pub/Sub, Cloud Run, Firestore, Cloud Scheduler, Secret Manager) and submitted to the **All Things Agentic Hackathon** under the **Taskmaster** track. The agent operates fully autonomously during NSE market hours (09:15–15:15 IST), requires zero human intervention per trade, and persists a self-improving strategy memory across sessions.

---

## 1. Requirements & Constraints

- **REQ-001**: The system must replicate all 8 ScalpMaster Pro indicators: RSI-14, MACD(12,26,9), Stochastic-%K, Momentum-10, EMA-200, SuperTrend(ATR), CCI-20, OBV-ROC — each returning a discrete `bull` or `bear` signal per evaluation cycle.
- **REQ-002**: A STRONG BUY signal requires `bull_score >= 6 AND bull_score > bear_score AND ema_zone == "UP"`. A STRONG SELL signal requires `bear_score >= 6 AND bear_score > bull_score AND ema_zone == "DOWN"`.
- **REQ-003**: Default risk/reward ratio must be 1:2 — Stop Loss at 1% from entry, Take Profit at 2% from entry.
- **REQ-004**: All trades must use Zerodha product type `MIS` (intraday margin). All open MIS positions must be force-closed by 15:15 IST daily.
- **REQ-005**: The Gemini 3.5 Decision Agent must return a structured JSON response containing fields: `action` (EXECUTE|SKIP|WAIT), `confidence` (FULL_CONFIDENCE|LOW_CONFIDENCE), `reasoning` (string, max 3 sentences), `entry_price` (float), `tp_price` (float), `sl_price` (float).
- **REQ-006**: The Reflection Agent must run after every closed trade and write an updated `strategy_memory/{YYYY-MM-DD}` document to Firestore within 30 seconds of trade closure.
- **REQ-007**: The system must support both NIFTY (lot size 50) and BANKNIFTY (lot size 15) option chains on the NFO segment.
- **REQ-008**: The instrument token list must be refreshed daily at 08:00 IST from `kite.instruments("NFO")`, filtering for weekly expiry CE and PE contracts within ±500 points of spot for NIFTY and ±1000 points for BANKNIFTY.
- **REQ-009**: The Kite Connect `access_token` must be renewed daily before 09:00 IST and stored in Google Secret Manager under secret ID `kite-access-token`.
- **REQ-010**: The agent must publish a Pub/Sub message to topic `scalpmaster-ticks` for every WebSocket tick received from KiteTicker in `MODE_FULL`.
- **SEC-001**: The Kite `api_key` and `api_secret` must be stored exclusively in Google Secret Manager. They must never appear in source code, environment variables, or Firestore documents.
- **SEC-002**: All Cloud Run service endpoints must require authentication (IAM-based). No public unauthenticated access is permitted.
- **SEC-003**: Firestore security rules must restrict read/write to the Cloud Run service account identity only.
- **SEC-004**: The Risk Guard Agent must enforce hard-block rules that cannot be overridden by Gemini output under any circumstances.
- **CON-001**: Kite Connect REST API rate limit is 3 requests/second. All REST calls must be throttled using a token-bucket rate limiter with capacity=3, refill_rate=3/sec.
- **CON-002**: KiteTicker WebSocket supports a maximum of 3000 instrument subscriptions per connection. Subscriptions must not exceed 200 instruments per session to stay within budget.
- **CON-003**: The `access_token` expires daily at midnight IST. The re-auth Cloud Scheduler job must complete before 09:00 IST.
- **CON-004**: Cloud Run minimum instances must be set to 0 outside market hours (15:30–08:45 IST) to eliminate idle billing.
- **CON-005**: Gemini Pro must only be invoked for the Reflection Agent post-trade analysis. All real-time Decision Agent calls must use Gemini 3.5 Flash to minimize latency and cost.
- **CON-006**: The total GCP spend must not exceed the $150 hackathon credit allocation. Budget alerts must be configured at $75 (50%) and $120 (80%) thresholds.
- **GUD-001**: All agent modules must be independently deployable Python packages under `src/agents/`. No agent module may import from another agent module directly — communication occurs exclusively via Pub/Sub messages or Firestore reads.
- **GUD-002**: All Firestore writes must use server-side timestamps (`firestore.SERVER_TIMESTAMP`) for `created_at` and `updated_at` fields.
- **GUD-003**: All Gemini prompts must be stored as versioned string constants in `src/prompts/prompts.py`, not inline in agent code.
- **GUD-004**: Every trade execution attempt must be logged to Firestore collection `execution_log/{order_id}` regardless of success or failure, including the full Kite API response or exception message.
- **PAT-001**: Follow the ADK Sequential Agent pattern for the main trade pipeline: Signal Agent → Decision Agent → Risk Guard Agent → Execution Agent → Monitor Agent → Reflection Agent.
- **PAT-002**: Use the ADK Parallel Agent pattern within the Signal Agent to compute all 8 indicator scores concurrently using `asyncio.gather()`.
- **PAT-003**: Use Firestore as the single source of truth for all agent state. No in-memory state may persist across Cloud Run request boundaries.

---

## 2. Implementation Steps

### Implementation Phase 1 — Project Scaffold & Infrastructure

- GOAL-001: Establish the complete GCP infrastructure, project directory structure, authentication pipeline, and local development environment so all subsequent phases can execute without infrastructure blockers.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-001 | Create GCP project named `scalpmaster-pro`. Enable APIs: `run.googleapis.com`, `pubsub.googleapis.com`, `firestore.googleapis.com`, `secretmanager.googleapis.com`, `cloudscheduler.googleapis.com`, `aiplatform.googleapis.com`. Execute: `gcloud services enable run.googleapis.com pubsub.googleapis.com firestore.googleapis.com secretmanager.googleapis.com cloudscheduler.googleapis.com aiplatform.googleapis.com --project=scalpmaster-pro` | | |
| TASK-002 | Create Pub/Sub topic `scalpmaster-ticks` and subscription `scalpmaster-ticks-sub` with ack deadline 60s. Execute: `gcloud pubsub topics create scalpmaster-ticks` and `gcloud pubsub subscriptions create scalpmaster-ticks-sub --topic=scalpmaster-ticks --ack-deadline=60` | | |
| TASK-003 | Create Firestore database in Native mode, region `asia-south1`. Execute: `gcloud firestore databases create --region=asia-south1 --project=scalpmaster-pro` | | |
| TASK-004 | Store Kite credentials in Secret Manager. Execute: `echo -n "<api_key>" \| gcloud secrets create kite-api-key --data-file=-` and `echo -n "<api_secret>" \| gcloud secrets create kite-api-secret --data-file=-`. Create placeholder secret `kite-access-token` with value `PENDING`. | | |
| TASK-005 | Create service account `scalpmaster-sa@scalpmaster-pro.iam.gserviceaccount.com`. Grant roles: `roles/pubsub.publisher`, `roles/pubsub.subscriber`, `roles/datastore.user`, `roles/secretmanager.secretAccessor`, `roles/aiplatform.user`, `roles/run.invoker`. | | |
| TASK-006 | Initialize project directory structure: create directories `src/agents/`, `src/prompts/`, `src/utils/`, `src/models/`, `tests/unit/`, `tests/integration/`, `plan/`, `scripts/`. Create files: `src/__init__.py`, `src/agents/__init__.py`, `src/prompts/__init__.py`, `src/utils/__init__.py`, `src/models/__init__.py`. | | |
| TASK-007 | Create `pyproject.toml` at project root with dependencies: `kiteconnect>=5.0.1`, `google-cloud-pubsub>=2.21.0`, `google-cloud-firestore>=2.16.0`, `google-cloud-secret-manager>=2.20.0`, `google-cloud-aiplatform>=1.60.0`, `google-adk>=1.0.0`, `pandas>=2.2.0`, `pandas-ta>=0.3.14b`, `numpy>=1.26.0`, `fastapi>=0.111.0`, `uvicorn>=0.30.0`, `pydantic>=2.7.0`, `pytest>=8.2.0`, `pytest-asyncio>=0.23.0`. Python version: `>=3.11`. | | |
| TASK-008 | Create `src/utils/secrets.py` with function `get_secret(secret_id: str) -> str` that calls `secretmanager.SecretManagerServiceClient().access_secret_version(name=f"projects/scalpmaster-pro/secrets/{secret_id}/versions/latest")` and returns the decoded payload. | | |
| TASK-009 | Create `src/utils/rate_limiter.py` with class `TokenBucketRateLimiter(capacity: int, refill_rate: float)` implementing `async def acquire() -> None` that blocks until a token is available. Instantiate a module-level singleton `kite_limiter = TokenBucketRateLimiter(capacity=3, refill_rate=3.0)` for use by all Kite REST callers. | | |
| TASK-010 | Create `src/utils/firestore_client.py` with function `get_db() -> firestore.AsyncClient` returning a module-level singleton Firestore async client initialized with project `scalpmaster-pro`. | | |
| TASK-011 | Set GCP budget alert: `gcloud billing budgets create --billing-account=<BILLING_ACCOUNT_ID> --display-name="ScalpMaster Budget" --budget-amount=150 --threshold-rule=percent=0.5 --threshold-rule=percent=0.8`. | | |

### Implementation Phase 2 — Kite Connect Integration Layer

- GOAL-002: Implement the complete Kite Connect v3 integration including daily authentication renewal, instrument token management, WebSocket tick streaming to Pub/Sub, and REST order execution — all rate-limited and secret-managed per REQ-009, CON-001, SEC-001.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-012 | Create `src/utils/kite_client.py` with function `get_kite() -> KiteConnect` that reads `api_key` from Secret Manager via `get_secret("kite-api-key")`, instantiates `KiteConnect(api_key=api_key)`, reads `access_token` from `get_secret("kite-access-token")`, calls `kite.set_access_token(access_token)`, and returns the instance. | | |
| TASK-013 | Create `scripts/daily_auth.py` — a standalone script that: (1) reads `api_key` and `api_secret` from Secret Manager, (2) prints the Kite login URL, (3) accepts `request_token` as a CLI argument `--request-token`, (4) calls `kite.generate_session(request_token, api_secret=api_secret)`, (5) writes the resulting `access_token` to Secret Manager secret `kite-access-token` using `client.add_secret_version(parent=..., payload=...)`. This script is invoked manually once per day or via a semi-automated Cloud Scheduler HTTP trigger to a dedicated `/auth/callback` endpoint. | | |
| TASK-014 | Create `src/utils/instrument_manager.py` with function `async def refresh_instruments() -> dict[str, int]` that: (1) calls `kite.instruments("NFO")` wrapped in `kite_limiter.acquire()`, (2) filters rows where `instrument_type` in `["CE","PE"]` and `name` in `["NIFTY","BANKNIFTY"]` and `expiry` equals the nearest weekly expiry date, (3) further filters to strikes within ±500 of NIFTY spot and ±1000 of BANKNIFTY spot using `kite.quote(["NSE:NIFTY 50","NSE:NIFTY BANK"])`, (4) returns a dict mapping `tradingsymbol -> instrument_token`, (5) writes the dict to Firestore document `config/instruments` with field `tokens: dict` and `refreshed_at: SERVER_TIMESTAMP`. | | |
| TASK-015 | Create `src/agents/ticker_agent.py` with class `TickerAgent` that: (1) reads instrument tokens from Firestore `config/instruments`, (2) instantiates `KiteTicker(api_key, access_token)`, (3) in `on_connect` callback subscribes to all tokens with `ws.set_mode(ws.MODE_FULL, token_list)`, (4) in `on_ticks` callback serializes each tick to JSON and publishes to Pub/Sub topic `scalpmaster-ticks` using `google.cloud.pubsub_v1.PublisherClient`, (5) implements `on_reconnect` to re-subscribe after reconnection, (6) implements `on_noreconnect` to log a CRITICAL alert to Firestore `system_logs` collection and exit with code 1. | | |
| TASK-016 | Create `src/utils/ohlcv_fetcher.py` with function `async def fetch_ohlcv(instrument_token: int, interval: str, lookback_bars: int) -> list[dict]` that calls `kite.historical_data(instrument_token, from_date, to_date, interval)` wrapped in `kite_limiter.acquire()`, where `from_date` is computed as `datetime.now() - timedelta(minutes=lookback_bars * interval_minutes)`. Returns list of dicts with keys: `date`, `open`, `high`, `low`, `close`, `volume`. Supported intervals: `"minute"`, `"5minute"`, `"15minute"`. | | |
| TASK-017 | Create `src/utils/order_manager.py` with functions: (a) `async def place_market_order(kite, tradingsymbol, transaction_type, quantity) -> str` returning `order_id`; (b) `async def place_slm_order(kite, tradingsymbol, transaction_type, quantity, trigger_price) -> str` returning `order_id`; (c) `async def modify_slm_order(kite, order_id, trigger_price) -> str`; (d) `async def cancel_order(kite, order_id) -> str`. All functions must call `kite_limiter.acquire()` before each Kite REST call and write the full API response to Firestore `execution_log/{order_id}` per GUD-004. | | |

### Implementation Phase 3 — Signal Scoring Engine

- GOAL-003: Implement all 8 indicator scoring functions as pure, stateless, testable Python functions in `src/agents/signal_agent.py`, each accepting a `list[dict]` OHLCV payload and returning `Literal["bull", "bear", "neutral"]`. Implement the parallel aggregation pipeline per PAT-002.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-018 | Create `src/models/signal_models.py` with Pydantic models: `OHLCVBar(date: datetime, open: float, high: float, low: float, close: float, volume: int)`, `IndicatorSignal(name: str, value: float, signal: Literal["bull","bear","neutral"])`, `ScoreResult(bull_score: int, bear_score: int, strength_pct: float, signals: dict[str, IndicatorSignal], ema_zone: Literal["UP","DOWN","NEUTRAL"])`. | | |
| TASK-019 | Implement `score_rsi(bars: list[OHLCVBar]) -> IndicatorSignal` in `src/agents/signal_agent.py`: compute RSI-14 using `pandas_ta.rsi(close_series, length=14)`. Return `bull` if `rsi < 40`, `bear` if `rsi > 60`, else `neutral`. Store computed RSI value in `IndicatorSignal.value`. | | |
| TASK-020 | Implement `score_macd(bars: list[OHLCVBar]) -> IndicatorSignal`: compute MACD(12,26,9) using `pandas_ta.macd(close_series, fast=12, slow=26, signal=9)`. Return `bull` if `macd_line > signal_line`, `bear` if `macd_line < signal_line`, else `neutral`. Store `macd_line - signal_line` as value. | | |
| TASK-021 | Implement `score_stochastic(bars: list[OHLCVBar]) -> IndicatorSignal`: compute Stochastic %K using `pandas_ta.stoch(high_series, low_series, close_series, k=14, d=3)`. Return `bull` if `%K < 20`, `bear` if `%K > 80`, else `neutral`. | | |
| TASK-022 | Implement `score_momentum(bars: list[OHLCVBar]) -> IndicatorSignal`: compute `momentum = close[-1] - close[-11]` (10-bar momentum). Return `bull` if `momentum > 0`, `bear` if `momentum < 0`, else `neutral`. | | |
| TASK-023 | Implement `score_ema200(bars: list[OHLCVBar]) -> IndicatorSignal` and `compute_ema_zone(bars: list[OHLCVBar], zone_pct: float = 0.002) -> Literal["UP","DOWN","NEUTRAL"]`: compute EMA-200 using `pandas_ta.ema(close_series, length=200)`. For `score_ema200`: return `bull` if `close[-1] > ema200`, `bear` if `close[-1] < ema200`. For `compute_ema_zone`: define `zone_upper = ema200 * (1 + zone_pct)`, `zone_lower = ema200 * (1 - zone_pct)`. Return `UP` if `close[-1] > zone_upper`, `DOWN` if `close[-1] < zone_lower`, else `NEUTRAL`. | | |
| TASK-024 | Implement `score_supertrend(bars: list[OHLCVBar]) -> IndicatorSignal`: compute SuperTrend using `pandas_ta.supertrend(high_series, low_series, close_series, length=10, multiplier=3.0)`. Return `bull` if `close[-1] > supertrend_value`, `bear` if `close[-1] < supertrend_value`. | | |
| TASK-025 | Implement `score_cci(bars: list[OHLCVBar]) -> IndicatorSignal`: compute CCI-20 using `pandas_ta.cci(high_series, low_series, close_series, length=20)`. Return `bull` if `cci < -100`, `bear` if `cci > 100`, else `neutral`. | | |
| TASK-026 | Implement `score_obv_roc(bars: list[OHLCVBar]) -> IndicatorSignal`: compute OBV using `pandas_ta.obv(close_series, volume_series)`, then compute 5-bar ROC of OBV as `(obv[-1] - obv[-6]) / obv[-6] * 100`. Return `bull` if `obv_roc > 0`, `bear` if `obv_roc < 0`, else `neutral`. | | |
| TASK-027 | Implement `async def compute_scores(bars: list[OHLCVBar]) -> ScoreResult` using `asyncio.gather()` to run all 8 scoring functions concurrently via `asyncio.get_event_loop().run_in_executor(None, score_fn, bars)`. Aggregate results: `bull_score = count(signal=="bull")`, `bear_score = count(signal=="bear")`, `strength_pct = (max(bull_score, bear_score) / 8) * 100`. Compute `ema_zone` via `compute_ema_zone`. Return populated `ScoreResult`. | | |

### Implementation Phase 4 — Gemini Decision & Risk Guard Agents

- GOAL-004: Implement the Gemini 3.5 Flash Decision Agent that reasons over `ScoreResult` and market context to produce a structured trade decision, and the hard-rule Risk Guard Agent that enforces non-overridable safety constraints per SEC-004.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-028 | Create `src/prompts/prompts.py` with string constant `DECISION_AGENT_PROMPT_V1` containing the full decision prompt template with placeholders: `{symbol}`, `{spot_price}`, `{option_ltp}`, `{bull_score}`, `{bear_score}`, `{strength_pct}`, `{ema_zone}`, `{signals_json}`, `{success_rate}`, `{regime}`, `{dte}`, `{iv}`, `{pcr}`. Prompt must instruct Gemini to return valid JSON only with fields: `action`, `confidence`, `reasoning`, `entry_price`, `tp_price`, `sl_price`. | | |
| TASK-029 | Create `src/models/decision_models.py` with Pydantic models: `DecisionInput(symbol: str, spot_price: float, option_ltp: float, score_result: ScoreResult, success_rate: float, regime: str, dte: int, iv: float, pcr: float)`, `DecisionOutput(action: Literal["EXECUTE","SKIP","WAIT"], confidence: Literal["FULL_CONFIDENCE","LOW_CONFIDENCE"], reasoning: str, entry_price: float, tp_price: float, sl_price: float)`. | | |
| TASK-030 | Create `src/agents/decision_agent.py` with `async def run_decision_agent(input: DecisionInput) -> DecisionOutput`: (1) format `DECISION_AGENT_PROMPT_V1` with input fields, (2) call Vertex AI Gemini 3.5 Flash via `aiplatform.GenerativeModel("gemini-2.5-flash").generate_content_async(prompt)`, (3) parse JSON response into `DecisionOutput` using `pydantic.model_validate_json()`, (4) on JSON parse failure retry once then return `DecisionOutput(action="SKIP", confidence="LOW_CONFIDENCE", reasoning="Parse failure", ...)`. | | |
| TASK-031 | Create `src/agents/risk_guard_agent.py` with `async def check_risk(kite, daily_pnl: float, capital: float, position_size_value: float, current_time: datetime) -> tuple[bool, str]`: implement hard-block rules — (a) `daily_pnl <= -(capital * 0.02)` → block "Daily loss limit 2% breached"; (b) `position_size_value > (capital * 0.05)` → block "Position exceeds 5% capital"; (c) `current_time.time() >= time(15, 15)` → block "Past MIS cutoff 15:15 IST"; (d) `current_time.time() < time(9, 15)` → block "Before market open 09:15 IST"; (e) fetch India VIX quote via `kite.quote(["NSE:INDIA VIX"])`, if `vix > 25.0` → block "India VIX spike above 25". Return `(True, "APPROVED")` only if all checks pass. | | |
| TASK-032 | Create `src/utils/market_utils.py` with functions: (a) `compute_pcr(instruments_df, nifty_spot) -> float` — aggregates OI for all PE and CE strikes within ±500 of spot, returns `total_put_oi / total_call_oi`; (b) `detect_regime(bars: list[OHLCVBar]) -> str` — returns `"trending_up"` if `close[-1] > ema20[-1] > ema50[-1]`, `"trending_down"` if inverse, `"ranging"` if ATR-14 < 0.5% of close, else `"volatile"`; (c) `get_success_rate(db: AsyncClient) -> float` — reads last 5 trades from Firestore `trades` collection ordered by `entry_time desc`, returns `count(outcome=="TP") / 5 * 100`. | | |

### Implementation Phase 5 — Execution, Monitor & Reflection Agents

- GOAL-005: Implement the Execution Agent that places entry + SL orders via Kite, the Monitor Agent that manages trailing SL and detects TP/SL hits, and the Reflection Agent that performs post-trade analysis using Gemini Pro and updates Firestore strategy memory.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-033 | Create `src/models/trade_models.py` with Pydantic models: `TradeRecord(trade_id: str, symbol: str, direction: Literal["BUY","SELL"], entry_time: datetime, entry_price: float, tp_price: float, sl_price: float, quantity: int, entry_order_id: str, sl_order_id: str, status: Literal["OPEN","CLOSED_TP","CLOSED_SL","CLOSED_AGENT","CLOSED_FORCE"], exit_price: float \| None, exit_time: datetime \| None, pnl: float \| None, regime: str, bull_score: int, bear_score: int, signal_confidence: str, gemini_reasoning: str, outcome: Literal["TP","SL","AGENT","FORCE"] \| None)`. | | |
| TASK-034 | Create `src/agents/execution_agent.py` with `async def execute_trade(kite, decision: DecisionOutput, symbol: str, quantity: int, direction: Literal["BUY","SELL"], score_result: ScoreResult, regime: str) -> TradeRecord`: (1) call `place_market_order()` for entry, (2) call `place_slm_order()` for SL at `decision.sl_price`, (3) construct `TradeRecord` with status `OPEN`, (4) write to Firestore `trades/{trade_id}`, (5) return `TradeRecord`. `trade_id` = `f"{symbol}_{datetime.now().strftime('%Y%m%d%H%M%S%f')}"`. | | |
| TASK-035 | Create `src/agents/monitor_agent.py` with `async def monitor_position(kite, trade: TradeRecord) -> TradeRecord`: poll every 5 seconds using `asyncio.sleep(5)`. On each poll: (1) fetch `kite.positions()` wrapped in `kite_limiter.acquire()`, (2) find matching position by `tradingsymbol`, (3) check if `current_ltp >= trade.tp_price` → cancel SL order, place market exit, update Firestore status `CLOSED_TP`; (4) check if SL order status is `COMPLETE` → update Firestore status `CLOSED_SL`; (5) implement trailing SL: if `direction==BUY` and `current_ltp >= entry_price * 1.01`, call `modify_slm_order()` to move SL to `entry_price` (breakeven); if `current_ltp >= entry_price * 1.015`, move SL to `entry_price * 1.005`; (6) if `current_time >= 15:15 IST`, force-close via market order and set status `CLOSED_FORCE`. | | |
| TASK-036 | Create `src/prompts/prompts.py` constant `REFLECTION_AGENT_PROMPT_V1` with placeholders: `{trade_json}`, `{entry_reasoning}`, `{outcome}`, `{regime}`, `{bull_score}`, `{ema_zone}`. Prompt instructs Gemini Pro to return JSON with fields: `entry_reasoning_correct: bool`, `most_predictive_indicators: list[str]`, `threshold_adjustment: dict[str, float]`, `memory_note: str`. | | |
| TASK-037 | Create `src/agents/reflection_agent.py` with `async def run_reflection(trade: TradeRecord) -> None`: (1) format `REFLECTION_AGENT_PROMPT_V1` with trade data, (2) call Vertex AI Gemini Pro via `aiplatform.GenerativeModel("gemini-2.5-pro").generate_content_async(prompt)`, (3) parse JSON response, (4) read existing `strategy_memory/{today_date}` from Firestore, (5) merge `threshold_adjustment` and append `memory_note` to `notes` array, (6) write updated document back to Firestore with `updated_at: SERVER_TIMESTAMP`. | | |

### Implementation Phase 6 — Orchestrator, Cloud Run & Scheduler

- GOAL-006: Wire all agents into the ADK Sequential Orchestrator, deploy to Cloud Run, configure Cloud Scheduler jobs for daily lifecycle management, and implement the FastAPI health/status endpoints.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-038 | Create `src/main.py` with FastAPI app. Define endpoints: `GET /health` returns `{"status":"ok","timestamp":...}`; `POST /trade/evaluate` accepts `EvaluateRequest(tick_data: dict)` and runs the full agent pipeline (Signal → Decision → Risk Guard → Execution → Monitor → Reflection); `GET /trades/today` returns all trades from Firestore for today's date; `GET /strategy/memory` returns latest `strategy_memory` document. | | |
| TASK-039 | Create `src/orchestrator.py` with `async def run_pipeline(tick_data: dict) -> dict`: implements PAT-001 sequential pipeline — (1) fetch OHLCV via `fetch_ohlcv()`, (2) `compute_scores()`, (3) early-exit if `score_result.bull_score < 6 AND score_result.bear_score < 6`, (4) build `DecisionInput`, (5) `run_decision_agent()`, (6) early-exit if `decision.action != "EXECUTE"`, (7) `check_risk()`, (8) early-exit if risk blocked, (9) `execute_trade()`, (10) `asyncio.create_task(monitor_position(...))` — non-blocking, (11) register `run_reflection()` as callback on monitor completion. Return pipeline result dict. | | |
| TASK-040 | Create `Dockerfile` at project root: base image `python:3.11-slim`, copy `src/` and `pyproject.toml`, run `pip install .`, expose port 8080, set `CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]`. | | |
| TASK-041 | Create `cloudbuild.yaml` with steps: (1) `docker build -t asia-south1-docker.pkg.dev/scalpmaster-pro/scalpmaster/app:latest .`, (2) `docker push ...`, (3) `gcloud run deploy scalpmaster-app --image=... --region=asia-south1 --service-account=scalpmaster-sa@... --min-instances=0 --max-instances=3 --memory=2Gi --cpu=2 --no-allow-unauthenticated`. | | |
| TASK-042 | Create Cloud Scheduler jobs via `gcloud scheduler jobs create http`: (a) `instrument-refresh` — cron `0 8 * * 1-5` IST, POST to `/internal/refresh-instruments`; (b) `market-open` — cron `14 9 * * 1-5` IST (09:14 IST), POST to `/internal/start`; (c) `market-close` — cron `15 15 * * 1-5` IST (15:15 IST), POST to `/internal/stop`; (d) `daily-cleanup` — cron `30 15 * * 1-5` IST, POST to `/internal/cleanup`. All jobs use OIDC token auth with `scalpmaster-sa` service account. | | |
| TASK-043 | Create `src/agents/ticker_agent.py` Cloud Run entrypoint `POST /internal/start` that starts `KiteTicker.connect(threaded=True)` and `POST /internal/stop` that calls `KiteTicker.close()` and triggers force-close of all open MIS positions via `kite.positions()` loop. | | |

### Implementation Phase 7 — Dashboard & Demo Preparation

- GOAL-007: Build a real-time web dashboard using the existing fullstack skills, generate architecture diagram, record demo video, and complete Devpost submission before 2026-09-01 05:30 IST.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-044 | Create `dashboard/` directory with a React + Vite frontend. Implement components: `SignalTable` (displays all 8 indicator statuses as ✅/❌, bull/bear scores, strength %), `TradeLog` (last 5 trades with outcome, P&L, entry/exit prices), `GeminiReasoningPanel` (displays `gemini_reasoning` field from latest trade), `RegimeIndicator` (current market regime badge), `DailyPnL` (running P&L for the day). Connect to Cloud Run `/trades/today` and `/strategy/memory` endpoints via polling every 10 seconds. | | |
| TASK-045 | Deploy dashboard to Firebase Hosting: `firebase init hosting` in `dashboard/`, configure `firebase.json` with `public: "dist"`, run `npm run build && firebase deploy`. | | |
| TASK-046 | Generate architecture diagram using `diagrams` Python library (`pip install diagrams`). Create `scripts/generate_diagram.py` that renders the full agent pipeline (KiteTicker → Pub/Sub → Cloud Run Orchestrator → 6 agents → Firestore → Dashboard) as `plan/architecture.png`. | | |
| TASK-047 | Record 4-minute demo video following this exact script: (0:00–0:45) screen-record manual TradingView ScalpMaster Pro workflow showing the friction; (0:45–2:30) live terminal showing Cloud Run logs with agent pipeline executing, Firestore console updating in real-time, Gemini reasoning JSON visible; (2:30–3:30) Cloud Run metrics dashboard proving GCP deployment, Vertex AI call logs; (3:30–4:00) Firestore `strategy_memory` document showing reflection output. Upload to YouTube as unlisted. | | |
| TASK-048 | Complete Devpost submission at `allthingsagentichackathon.devpost.com`: fill project name "ScalpMaster Pro — Autonomous NIFTY/BANKNIFTY Options Agent", description (500 words covering friction story, architecture, Gemini reasoning, self-improving memory), embed YouTube demo URL, add GitHub repo URL, select track "Taskmaster", tag technologies: `Google ADK`, `Gemini 3.5 Flash`, `Vertex AI`, `Cloud Run`, `Pub/Sub`, `Firestore`, `Kite Connect v3`. Submit before 2026-09-01 05:30 IST. | | |

---

## 3. Alternatives

- **ALT-001**: Use Alpaca or Interactive Brokers API instead of Kite Connect v3 — rejected because the project is specifically designed for NIFTY/BANKNIFTY on NSE/NFO, and Kite Connect is the most reliable, lowest-latency broker API for Indian markets with native WebSocket tick streaming.
- **ALT-002**: Use a rule-based engine (pure `if/else` on `bull_score >= 6`) instead of Gemini 3.5 for the Decision Agent — rejected because it eliminates the agentic reasoning layer that differentiates this project from a simple trading bot and is required to score on the Innovation criterion (40%).
- **ALT-003**: Use Redis instead of Firestore for agent state — rejected because Firestore is serverless, scales to zero, has native GCP IAM integration, and eliminates the need for a dedicated always-on cache cluster (violates CON-004 and CON-006).
- **ALT-004**: Use LangChain instead of Google ADK — rejected because the hackathon explicitly recommends ADK and judges will evaluate use of Google-native tooling. ADK also provides native Vertex AI integration with lower boilerplate.
- **ALT-005**: Use `ta-lib` instead of `pandas-ta` for indicator computation — rejected because `ta-lib` requires a C library system dependency that complicates Docker builds. `pandas-ta` is pure Python and pip-installable without system dependencies.
- **ALT-006**: Deploy on GKE instead of Cloud Run — rejected because Cloud Run scales to zero (CON-004), requires no cluster management, and is sufficient for the agent's request-driven workload pattern.

---

## 4. Dependencies

- **DEP-001**: `kiteconnect>=5.0.1` — Zerodha's official Python client for Kite Connect v3 REST API and KiteTicker WebSocket. Source: `pip install kiteconnect`. Required by TASK-012, TASK-015, TASK-016, TASK-017.
- **DEP-002**: `pandas-ta>=0.3.14b` — Pure Python technical analysis library providing RSI, MACD, Stochastic, EMA, SuperTrend, CCI, OBV implementations. Source: `pip install pandas-ta`. Required by TASK-019 through TASK-026.
- **DEP-003**: `google-cloud-aiplatform>=1.60.0` — Vertex AI Python SDK for Gemini 3.5 Flash and Gemini Pro model invocation. Source: `pip install google-cloud-aiplatform`. Required by TASK-030, TASK-037.
- **DEP-004**: `google-adk>=1.0.0` — Google Agent Development Kit for agent orchestration patterns. Source: `pip install google-adk`. Required by TASK-039.
- **DEP-005**: `google-cloud-pubsub>=2.21.0` — GCP Pub/Sub client for tick event publishing and subscription. Required by TASK-015.
- **DEP-006**: `google-cloud-firestore>=2.16.0` — Firestore async client for all agent state persistence. Required by TASK-010, TASK-034, TASK-035, TASK-037.
- **DEP-007**: `google-cloud-secret-manager>=2.20.0` — Secret Manager client for Kite credential retrieval. Required by TASK-008, TASK-013.
- **DEP-008**: `fastapi>=0.111.0` + `uvicorn>=0.30.0` — ASGI web framework and server for Cloud Run HTTP endpoints. Required by TASK-038.
- **DEP-009**: `pydantic>=2.7.0` — Data validation and serialization for all agent input/output models. Required by TASK-018, TASK-029, TASK-033.
- **DEP-010**: Zerodha Kite Connect API subscription — requires an active Kite Connect developer account at `kite.trade`. Monthly fee applies (₹2000/month as of knowledge cutoff). `api_key` and `api_secret` must be obtained before TASK-012.
- **DEP-011**: GCP Project `scalpmaster-pro` with billing account linked and $150 hackathon credit applied via `forms.gle/5PtXmw1dSbDnpYke9`. Must be completed before TASK-001.
- **DEP-012**: `diagrams>=0.23.4` — Python library for generating architecture diagrams as PNG. Required by TASK-046. Source: `pip install diagrams`.

---

## 5. Files

- **FILE-001**: `src/main.py` — FastAPI application entrypoint. Defines all HTTP endpoints. Created in TASK-038.
- **FILE-002**: `src/orchestrator.py` — Sequential agent pipeline coordinator. Implements `run_pipeline()`. Created in TASK-039.
- **FILE-003**: `src/agents/signal_agent.py` — All 8 indicator scoring functions and `compute_scores()` async aggregator. Created in TASK-019 through TASK-027.
- **FILE-004**: `src/agents/decision_agent.py` — Gemini 3.5 Flash Decision Agent. Created in TASK-030.
- **FILE-005**: `src/agents/risk_guard_agent.py` — Hard-rule Risk Guard Agent. Created in TASK-031.
- **FILE-006**: `src/agents/execution_agent.py` — Kite order placement and TradeRecord creation. Created in TASK-034.
- **FILE-007**: `src/agents/monitor_agent.py` — Position monitoring, trailing SL, TP/SL detection. Created in TASK-035.
- **FILE-008**: `src/agents/reflection_agent.py` — Post-trade Gemini Pro analysis and strategy memory update. Created in TASK-037.
- **FILE-009**: `src/agents/ticker_agent.py` — KiteTicker WebSocket manager and Pub/Sub publisher. Created in TASK-015, TASK-043.
- **FILE-010**: `src/prompts/prompts.py` — All versioned Gemini prompt string constants. Created in TASK-028, TASK-036.
- **FILE-011**: `src/models/signal_models.py` — Pydantic models for OHLCV, IndicatorSignal, ScoreResult. Created in TASK-018.
- **FILE-012**: `src/models/decision_models.py` — Pydantic models for DecisionInput, DecisionOutput. Created in TASK-029.
- **FILE-013**: `src/models/trade_models.py` — Pydantic model for TradeRecord. Created in TASK-033.
- **FILE-014**: `src/utils/kite_client.py` — KiteConnect singleton factory. Created in TASK-012.
- **FILE-015**: `src/utils/secrets.py` — Secret Manager accessor function. Created in TASK-008.
- **FILE-016**: `src/utils/rate_limiter.py` — Token bucket rate limiter for Kite REST API. Created in TASK-009.
- **FILE-017**: `src/utils/firestore_client.py` — Firestore async client singleton. Created in TASK-010.
- **FILE-018**: `src/utils/ohlcv_fetcher.py` — Kite historical data fetcher. Created in TASK-016.
- **FILE-019**: `src/utils/order_manager.py` — Kite order placement/modification/cancellation wrappers. Created in TASK-017.
- **FILE-020**: `src/utils/instrument_manager.py` — Daily NFO instrument token refresh. Created in TASK-014.
- **FILE-021**: `src/utils/market_utils.py` — PCR computation, regime detection, success rate retrieval. Created in TASK-032.
- **FILE-022**: `scripts/daily_auth.py` — Manual/semi-automated Kite OAuth token renewal script. Created in TASK-013.
- **FILE-023**: `scripts/generate_diagram.py` — Architecture diagram generator. Created in TASK-046.
- **FILE-024**: `Dockerfile` — Container build definition for Cloud Run deployment. Created in TASK-040.
- **FILE-025**: `cloudbuild.yaml` — Cloud Build CI/CD pipeline definition. Created in TASK-041.
- **FILE-026**: `pyproject.toml` — Python project metadata and dependency specification. Created in TASK-007.
- **FILE-027**: `dashboard/src/` — React + Vite frontend source directory. Created in TASK-044.
- **FILE-028**: `plan/feature-scalpmaster-agent-1.md` — This implementation plan file.
- **FILE-029**: `plan/architecture.png` — Generated architecture diagram. Created in TASK-046.
- **FILE-030**: `tests/unit/test_signal_agent.py` — Unit tests for all 8 indicator scoring functions. Created in TEST-001 through TEST-008.
- **FILE-031**: `tests/integration/test_pipeline.py` — Integration tests for full agent pipeline. Created in TEST-009 through TEST-012.

---

## 6. Testing

- **TEST-001**: Unit test `score_rsi()` in `tests/unit/test_signal_agent.py`: provide synthetic OHLCV bars with known RSI values (RSI=35 → assert `bull`, RSI=65 → assert `bear`, RSI=50 → assert `neutral`). Use `pytest` with `@pytest.mark.parametrize`.
- **TEST-002**: Unit test `score_macd()`: provide bars where MACD line is above signal line → assert `bull`; below → assert `bear`.
- **TEST-003**: Unit test `score_stochastic()`: provide bars with %K=15 → assert `bull`; %K=85 → assert `bear`.
- **TEST-004**: Unit test `score_momentum()`: provide bars where `close[-1] > close[-11]` → assert `bull`; inverse → assert `bear`.
- **TEST-005**: Unit test `score_ema200()` and `compute_ema_zone()`: provide 250 bars with known EMA-200 value. Assert `UP` when price is 0.5% above EMA, `DOWN` when 0.5% below, `NEUTRAL` when within 0.2%.
- **TEST-006**: Unit test `score_supertrend()`: provide bars with known SuperTrend value above and below close price.
- **TEST-007**: Unit test `score_cci()`: provide bars with CCI=-120 → assert `bull`; CCI=+120 → assert `bear`.
- **TEST-008**: Unit test `score_obv_roc()`: provide bars with rising OBV → assert `bull`; falling OBV → assert `bear`.
- **TEST-009**: Unit test `compute_scores()`: provide bars that produce `bull_score=7, bear_score=1` → assert `strength_pct=87.5`, `ema_zone="UP"`.
- **TEST-010**: Unit test `check_risk()` in `tests/unit/test_risk_guard.py`: assert block when `daily_pnl = -(capital * 0.025)`; assert block when `current_time = time(15, 20)`; assert `APPROVED` when all conditions are within limits.
- **TEST-011**: Integration test `run_pipeline()` in `tests/integration/test_pipeline.py`: mock `fetch_ohlcv()` to return synthetic bars with `bull_score=7`, mock `run_decision_agent()` to return `action="EXECUTE"`, mock `check_risk()` to return `(True, "APPROVED")`, mock `execute_trade()` to return a `TradeRecord`. Assert pipeline returns a dict with key `trade_id`.
- **TEST-012**: Integration test for early-exit path: mock `compute_scores()` to return `bull_score=4, bear_score=4`. Assert `run_pipeline()` returns `{"action": "SKIP", "reason": "insufficient_score"}` without calling `run_decision_agent()`.
- **TEST-013**: Integration test for Risk Guard block path: mock scores to produce `bull_score=7`, mock Decision Agent to return `EXECUTE`, mock `check_risk()` to return `(False, "Daily loss limit 2% breached")`. Assert `run_pipeline()` returns `{"action": "BLOCKED", "reason": "Daily loss limit 2% breached"}` without calling `execute_trade()`.

---

## 7. Risks & Assumptions

- **RISK-001**: Kite Connect `access_token` renewal failure — if `scripts/daily_auth.py` fails before 09:00 IST, the entire agent is inoperable for the day. Mitigation: implement a `/health` endpoint that checks token validity at startup and alerts via Firestore `system_logs` if token is stale.
- **RISK-002**: KiteTicker WebSocket disconnection during market hours — `on_noreconnect` callback triggers a CRITICAL log but the agent goes blind. Mitigation: implement a Cloud Scheduler heartbeat every 5 minutes that checks `system_logs` for recent tick activity and restarts the Cloud Run instance if no ticks received in 10 minutes.
- **RISK-003**: Gemini 3.5 Flash API latency exceeding 2 seconds — for scalping, decision latency is critical. Mitigation: set `timeout=3.0` on all Gemini API calls; on timeout, fall back to pure rule-based decision (`bull_score >= 6 → EXECUTE`) and log the fallback event.
- **RISK-004**: GCP credit exhaustion before hackathon deadline — Gemini Pro calls in Reflection Agent are expensive. Mitigation: cap Reflection Agent to run maximum once per 5 minutes using a Firestore timestamp check; use Gemini Flash for reflection if daily Gemini Pro spend exceeds $5.
- **RISK-005**: Kite MIS auto-square-off at 15:20 IST overriding agent's 15:15 IST force-close — if the agent's force-close fails, Zerodha will square off at market price with a penalty. Mitigation: implement a redundant force-close trigger at 15:10 IST as a second Cloud Scheduler job.
- **RISK-006**: `pandas-ta` SuperTrend computation diverging from TradingView's SuperTrend due to different ATR smoothing methods — `pandas-ta` uses RMA (Wilder's smoothing) by default, matching TradingView. Validate by comparing outputs on 50 historical bars before live deployment.
- **ASSUMPTION-001**: The Kite Connect v3 API interface (authentication flow, `instruments()`, `historical_data()`, `place_order()`, `positions()`, KiteTicker WebSocket) has not changed materially since the knowledge cutoff of January 2025. Verify against `kite.trade/docs/connect/v3/` before TASK-012.
- **ASSUMPTION-002**: NIFTY lot size is 50 and BANKNIFTY lot size is 15 as of the implementation date. NSE periodically revises lot sizes; verify current lot sizes at `nseindia.com` before TASK-034.
- **ASSUMPTION-003**: The hackathon judges will have access to the Cloud Run deployment URL and Firestore console screenshots as proof of GCP usage. A live demo environment will remain active during the judging period (estimated Sep 1–15, 2026).
- **ASSUMPTION-004**: The `pandas-ta` library correctly implements all 8 indicators (RSI, MACD, Stochastic, EMA, SuperTrend, CCI, OBV) with default parameters matching TradingView's Pine Script implementations. Discrepancies in signal generation vs. the original TradingView indicator are acceptable as long as the logic is directionally consistent.

---

## 8. Related Specifications / Further Reading

- [Kite Connect v3 API Documentation](https://kite.trade/docs/connect/v3/)
- [Google Agent Development Kit (ADK) Documentation](https://google.github.io/adk-docs)
- [Vertex AI Gemini API Reference](https://cloud.google.com/vertex-ai/generative-ai/docs/reference/python/latest)
- [Google Cloud Pub/Sub Python Client](https://cloud.google.com/python/docs/reference/pubsub/latest)
- [Cloud Run Deployment Documentation](https://cloud.google.com/run/docs/deploying)
- [Firestore Async Python Client](https://cloud.google.com/python/docs/reference/firestore/latest)
- [pandas-ta Documentation](https://github.com/twopirllc/pandas-ta)
- [All Things Agentic Hackathon — Taskmaster Track](https://allthingsagentichackathon.devpost.com/)
- [GEAR (Gemini Enterprise Agent Ready) Program](https://developers.google.com/program/gear)
