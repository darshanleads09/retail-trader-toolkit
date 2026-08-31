---
goal: Build AI-Powered ScalpMaster Pro — Real-Time Paper Trading Simulator for NIFTY/BANKNIFTY Options
version: 4.1
date_created: 2026-08-26
last_updated: 2026-08-31
owner: TraderMan (CMT | CQF | Senior Hedge Fund Manager)
status: 'In progress'
tags: [feature, simulation, paper-trading, local-dev, sqlite, agentic, finance, trading, quantitative, risk-management, data-collection, ml-ready, local-llm, ollama, deepseek]
---

# Introduction

![Status: In progress](https://img.shields.io/badge/status-In%20progress-yellow)

This plan describes the end-to-end implementation of **ScalpMaster Pro v4.1** — a **real-time paper trading simulator** that connects to the live Zerodha Kite Connect v3 API for **read-only market data** (WebSocket ticks, OHLCV, option chains, quotes) and simulates the complete NIFTY/BANKNIFTY options scalping pipeline **without placing a single real order**.

**Version 4.1 replaces all cloud LLM dependencies (Gemini Flash, Gemini Pro, Vertex AI, `google-generativeai` SDK) with a fully local LLM stack: [Ollama](https://ollama.com) running `deepseek-r1:8b`.** There is no Gemini fallback. There is no internet dependency for AI inference. All reasoning happens on-device.

**Why DeepSeek-R1:8b via Ollama:**
- **Best-in-class reasoning at 8B scale** — DeepSeek-R1's chain-of-thought architecture produces structured, multi-step reasoning that is well-suited to evaluating multi-indicator trading signals
- **Explicit `<think>` reasoning blocks** — the model's internal reasoning trace is captured and stored in the `signal_audit` table as `llm_reasoning_trace`, providing a richer ML training signal than a final-answer-only model
- **100% local, zero API cost, zero latency variance** — no network round-trips, no rate limits, no API keys
- **Ollama `"format":"json"` enforcement** — guarantees JSON-only output, eliminating parse failures
- **Separate process memory** — Ollama runs as its own process; the Python agent's 512 MB RAM budget is unaffected

**All cloud infrastructure has been removed.** There is no GCP, no Docker, no Cloud Run, no Firestore, no Pub/Sub, no Secret Manager, no Cloud Scheduler, no Vertex AI, no `google-generativeai`. The entire system runs as two local processes: the Python agent on `localhost:8080` and the Ollama server on `localhost:11434`.

**Purpose of this version:** Accumulate a statistically significant audit trail of simulated trades — every signal, every decision, every simulated entry/exit, every quant metric, every LLM reasoning trace — so that win probability can be calculated, market patterns can be identified, and a predictive ML model can be trained in a future version.

**What the system does:**
- Connects to live KiteTicker WebSocket → receives real-time option price ticks
- Fetches real OHLCV data from Kite `historical_data()` API
- Runs the full 8-indicator signal scoring engine on live data
- Calls DeepSeek-R1:8b (via Ollama) to reason over the signal and produce a structured trade decision
- Captures the full `<think>` reasoning trace for ML audit storage
- Simulates order entry, SL, TP, and trailing SL against live tick prices
- Monitors the simulated position in real-time against the live feed
- Runs the Reflection Agent (also DeepSeek-R1:8b) after each closed simulation
- Writes a **complete, ML-ready audit record** to SQLite for every trade lifecycle event

**What the system does NOT do:**
- Place any real order on Zerodha
- Call any external API for AI inference
- Write to any cloud service
- Require Docker, GCP credentials, or any cloud SDK
- Require an internet connection after initial model pull

---

## 1. Requirements & Constraints

- **REQ-001**: The system must replicate all 8 ScalpMaster Pro indicators: RSI-14, MACD(12,26,9), Stochastic-%K, Momentum-10, EMA-200, SuperTrend(ATR), CCI-20, OBV-ROC — each returning a discrete `bull` or `bear` signal per evaluation cycle.
- **REQ-002**: A STRONG BUY signal requires `bull_score >= 6 AND bull_score > bear_score AND ema_zone == "UP"`. A STRONG SELL signal requires `bear_score >= 6 AND bear_score > bull_score AND ema_zone == "DOWN"`.
- **REQ-003**: Default risk/reward ratio must be 1:2 — Stop Loss at 1% from simulated entry, Take Profit at 2% from simulated entry. When ATR-based SL (REQ-011) is active, the fixed 1% SL is superseded.
- **REQ-004**: All simulated trades must use product type `MIS` semantics. All open simulated positions must be force-closed at 15:15 IST daily by the Simulation Monitor.
- **REQ-005**: The DeepSeek-R1 Decision Agent (via Ollama) must return a structured JSON response: `action` (EXECUTE|SKIP|WAIT), `confidence` (FULL_CONFIDENCE|LOW_CONFIDENCE), `reasoning` (string, max 3 sentences, extracted from post-`<think>` output), `entry_price` (float), `tp_price` (float), `sl_price` (float). The full raw response including `<think>` block must be stored separately in `signal_audit.llm_reasoning_trace`.
- **REQ-006**: The Reflection Agent must run after every closed simulated trade and write an updated `strategy_memory` row to SQLite within 30 seconds of simulated trade closure.
- **REQ-007**: The system must support both NIFTY (lot size 50) and BANKNIFTY (lot size 15) option chains on the NFO segment.
- **REQ-008**: The instrument token list must be refreshed daily at 08:00 IST from `kite.instruments("NFO")`, filtering for weekly expiry CE and PE contracts within ±500 points of spot for NIFTY and ±1000 points for BANKNIFTY. Results stored in SQLite `config` table.
- **REQ-009**: The Kite Connect `access_token` must be renewed daily before 09:00 IST and written to `.env` under key `KITE_ACCESS_TOKEN` using `dotenv.set_key()`.
- **REQ-010**: The agent must enqueue every WebSocket tick from KiteTicker `MODE_FULL` into the in-process `asyncio.Queue` named `tick_queue` (maxsize=500). The Orchestrator consumes from this queue to drive the pipeline.
- **REQ-011**: The Quant Risk Layer must compute ATR-14 on the underlying spot at signal time. Dynamic SL = `entry_price - (1.5 × ATR_14)` for BUY, `entry_price + (1.5 × ATR_14)` for SELL. TP = `entry_price ± (3.0 × ATR_14)`. If ATR-based SL exceeds 2% of entry price, the simulation must be blocked and logged as `BLOCKED_ATR`.
- **REQ-012**: The Quant Risk Layer must compute **Kelly Criterion** optimal simulated position size: `f* = (W × R - (1 - W)) / R`. Apply half-Kelly (`0.5 × f*`). Minimum: 1 lot. Maximum: 5 lots. `W` and `R` derived from last 20 closed simulated trades in SQLite `trades` table.
- **REQ-013**: The Quant Risk Layer must fit a **GARCH(1,1)** model on the last 100 one-minute close returns of the underlying spot. Use the pure NumPy implementation in `src/quant/garch_lite.py` (zero extra packages). On fitting failure, fall back to `np.std(returns[-20:]) * np.sqrt(375)` and log `garch_engine = "ewma_fallback"`.
- **REQ-014**: The Decision Agent must compute and log **Expected Value (EV)**: `EV = (W × TP_distance) - ((1 - W) × SL_distance)`. Simulations with `EV <= 0` must be blocked and logged as `BLOCKED_EV`.
- **REQ-015**: The system must apply an **IV Percentile filter**: block simulation entry if `IV_percentile > 80`. Log as `BLOCKED_IV`.
- **REQ-016**: The Simulation Monitor must track simulated position P&L in real-time by comparing `current_ltp` (from live tick feed) against `tp_price` and `sl_price`. It must NOT call `kite.positions()` or any Kite order API.
- **REQ-017**: Every pipeline evaluation — including SKIP and BLOCKED outcomes — must write a complete audit row to the SQLite `signal_audit` table including the full `llm_reasoning_trace` field. No evaluation may be silently discarded.
- **REQ-018**: The SQLite `trades` table must store all fields required for post-hoc win probability analysis: all 8 indicator signals and values, all quant metrics, LLM reasoning, regime, PCR, DTE, IV, simulated entry/exit prices, simulated P&L, outcome, and timestamps with millisecond precision.
- **REQ-019**: The system must expose a `GET /analytics/summary` endpoint that queries SQLite and returns: total simulations, win rate, average P&L per trade, average EV at entry, win rate by regime, win rate by bull_score, win rate by hour-of-day, most predictive indicator by correlation with outcome.
- **REQ-020**: Total Python process RAM must not exceed **512 MB**. Ollama runs as a separate process and its RAM (~5–6 GB for deepseek-r1:8b) is outside this budget. Monitor Python process via `psutil` in `/health` endpoint. Warn if > 400 MB.
- **REQ-021**: The `OllamaClient` must parse DeepSeek-R1's `<think>...</think>` reasoning block from the raw response and store it separately from the final JSON answer. The `<think>` block is stored in `signal_audit.llm_reasoning_trace`. The JSON answer (after the `</think>` tag) is parsed into the structured response model.
- **REQ-022**: The Ollama server must be verified as running and the `deepseek-r1:8b` model must be confirmed loaded before the agent starts. The `scripts/start.py` launcher must call `GET http://localhost:11434/api/tags` and abort with a clear error message if Ollama is not running or the model is not available.
- **SEC-001**: The Kite `api_key` and `api_secret` must be stored exclusively in `.env` at project root (git-ignored). They must never appear in source code. No LLM API keys are required.
- **SEC-002**: The FastAPI server must bind exclusively to `127.0.0.1` (localhost). Never `0.0.0.0`.
- **SEC-003**: The SQLite database file `data/scalpmaster.db` must have file permissions `600` (owner read/write only). Set via `chmod 600 data/scalpmaster.db` after creation.
- **SEC-004**: The Risk Guard Agent must enforce hard-block rules that cannot be overridden by LLM output under any circumstances.
- **CON-001**: Kite Connect REST API rate limit is 3 requests/second. All REST calls must be throttled using `TokenBucketRateLimiter(capacity=3, refill_rate=3.0)`.
- **CON-002**: KiteTicker WebSocket subscriptions must not exceed 200 instruments per session.
- **CON-003**: The `access_token` expires daily at midnight IST. `scripts/daily_auth.py` must be run before 09:00 IST each trading day.
- **CON-004**: The `tick_queue` must have `maxsize=500`. On overflow, drop the incoming tick and log a WARN to SQLite `system_logs`.
- **CON-005**: DeepSeek-R1:8b via Ollama is the **sole LLM** for both the Decision Agent and the Reflection Agent. There is no fallback to any cloud LLM. If Ollama is unavailable, the pipeline must log `BLOCKED_LLM` to `signal_audit` and skip the trade — it must not attempt any external API call.
- **CON-006**: SQLite `data/scalpmaster.db` must not exceed **100 MB** on disk. A daily cleanup job (APScheduler, 15:30 IST) deletes `system_logs` rows older than 7 days and `signal_audit` rows older than 90 days. `trades` rows are never deleted.
- **CON-007**: GARCH(1,1) fitting via `garch_lite.py` must complete within 200ms. If exceeded, fall back to EWMA and log the event.
- **CON-008**: Kelly Criterion requires minimum 10 completed simulated trades in SQLite `trades`. If fewer, default to 1 lot and log `kelly_fallback = 1`.
- **CON-009**: Total installed venv disk size must not exceed **1.2 GB**. No cloud SDKs, no Docker, no Node.js, no `google-generativeai`.
- **CON-010**: Peak Python process RAM must not exceed **512 MB**. OHLCV lookback hard-capped at 220 bars. `pandas` imported inside function bodies only. `gc.collect()` called after each pipeline cycle.
- **CON-011**: The uvicorn server must run with `--workers 1 --limit-concurrency 4`.
- **CON-012**: Ollama inference timeout must be set to **45 seconds** for the Decision Agent and **90 seconds** for the Reflection Agent. If Ollama does not respond within the timeout, log `BLOCKED_LLM_TIMEOUT` and skip the trade.
- **CON-013**: DeepSeek-R1:8b must be called with `temperature=0.1` and `num_predict=768` for the Decision Agent (structured JSON, low creativity needed) and `temperature=0.3` and `num_predict=1536` for the Reflection Agent (analytical narrative, more tokens needed).
- **CON-014**: The `"format": "json"` parameter must be set in every Ollama API call to enforce JSON-only output mode. This is non-negotiable — without it, DeepSeek-R1 may wrap JSON in markdown code fences.
- **GUD-001**: All agent modules must be independently importable under `src/agents/`. No direct cross-agent imports. Communication via `asyncio.Queue` or SQLite reads only.
- **GUD-002**: All SQLite writes must include `created_at` and `updated_at` as ISO-8601 UTC strings with millisecond precision: `datetime.now(timezone.utc).isoformat(timespec="milliseconds")`.
- **GUD-003**: All LLM prompts must be stored as versioned string constants in `src/prompts/prompts.py`. Prompts must be written for DeepSeek-R1's instruction format and must explicitly instruct the model to output **only valid JSON after its reasoning**.
- **GUD-004**: Every pipeline evaluation — EXECUTE, SKIP, WAIT, BLOCKED — must write to `signal_audit` table per REQ-017, including `llm_reasoning_trace` when an LLM call was made.
- **GUD-005**: All quantitative model outputs must be stored as dedicated columns in the `trades` and `signal_audit` tables (not as JSON blobs) to enable direct SQL analytics queries per REQ-019.
- **GUD-006**: Regime-conditional thresholds: `volatile` regime requires `bull_score >= 7`; `ranging` regime skips all simulations.
- **GUD-007**: All numpy arrays in indicator computation must use `dtype=np.float32`. Exception: GARCH log-return arrays must use `float64`.
- **GUD-008**: SQLite must be opened with WAL mode (`PRAGMA journal_mode=WAL`) and `PRAGMA cache_size=-8000` (8 MB page cache).
- **GUD-009**: The `signal_audit` table is the **primary ML training dataset**. Every row must be self-contained — all features, all labels, and the full `llm_reasoning_trace` — so a future model can be trained with a single `SELECT *` query.
- **GUD-010**: The `<think>` block extracted from DeepSeek-R1 responses must be stored verbatim in `signal_audit.llm_reasoning_trace` with no truncation. This reasoning trace is a first-class ML feature — it encodes the model's chain-of-thought about the trade signal and may be embedded for semantic similarity analysis in future phases.
- **PAT-001**: Sequential agent pipeline: Signal Agent → Decision Agent → Risk Guard Agent → Quant Risk Agent → Simulation Execution Agent → Simulation Monitor Agent → Reflection Agent.
- **PAT-002**: Parallel indicator scoring within Signal Agent using `asyncio.gather()` + `run_in_executor`.
- **PAT-003**: SQLite is the single source of truth for all agent state. No in-memory state persists across pipeline invocations except `tick_queue`.
- **PAT-004**: Quant Risk Agent is a gate between Risk Guard and Simulation Execution.
- **PAT-005**: The Simulation Monitor reads live ticks from `tick_queue` (not from Kite API) to determine TP/SL hits.

---

## 2. Implementation Steps

### Implementation Phase 0 — Project Bootstrap, Environment & Ollama Setup

- GOAL-001: Initialize the project repository, Python virtual environment, `.env` configuration, SQLite schema, Ollama installation, and DeepSeek-R1:8b model pull so all subsequent phases have a working local runtime with zero external dependencies beyond Kite API.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-001 | Create project root `scalpmaster-pro/`. Run `git init`. Create `.gitignore` with entries: `.env`, `data/`, `__pycache__/`, `*.pyc`, `.venv/`, `*.db`, `*.db-wal`, `*.db-shm`. | | |
| TASK-002 | Create Python 3.11 virtual environment: `python3.11 -m venv .venv`. Activate: `source .venv/bin/activate` (Unix) or `.venv\Scripts\activate` (Windows). Verify: `python --version` returns `3.11.x`. | | |
| TASK-003 | Create `.env` at project root with keys: `KITE_API_KEY=`, `KITE_API_SECRET=`, `KITE_ACCESS_TOKEN=PENDING`, `DB_PATH=data/scalpmaster.db`, `TRADING_CAPITAL=500000`, `LOG_LEVEL=INFO`, `HOST=127.0.0.1`, `PORT=8080`, `OLLAMA_BASE_URL=http://localhost:11434`, `DECISION_MODEL=deepseek-r1:8b`, `REFLECTION_MODEL=deepseek-r1:8b`, `DECISION_TEMPERATURE=0.1`, `REFLECTION_TEMPERATURE=0.3`, `DECISION_NUM_PREDICT=768`, `REFLECTION_NUM_PREDICT=1536`, `OLLAMA_DECISION_TIMEOUT=45`, `OLLAMA_REFLECTION_TIMEOUT=90`. Create `.env.example` as a copy with all values replaced by placeholders. Add `.env` to `.gitignore`. Note: **No `GEMINI_API_KEY` or any cloud API key is required.** | | |
| TASK-004 | Create `requirements.txt` with all dependencies: `kiteconnect>=5.0.1`, `pandas>=2.2.0`, `pandas-ta>=0.3.14b`, `numpy>=1.26.0`, `aiosqlite>=0.20.0`, `python-dotenv>=1.0.0`, `apscheduler>=3.10.4`, `fastapi>=0.111.0`, `uvicorn>=0.30.0`, `pydantic>=2.7.0`, `httpx>=0.27.0`, `psutil>=5.9.0`, `plyer>=2.1.0`, `pytest>=8.2.0`, `pytest-asyncio>=0.23.0`. **`google-generativeai` is explicitly excluded.** Run: `pip install -r requirements.txt`. Estimated venv disk size: ~720 MB. | | |
| TASK-005 | **Install Ollama.** On macOS/Linux: `curl -fsSL https://ollama.com/install.sh \| sh`. On Windows: download installer from `https://ollama.com/download`. Verify: `ollama --version`. Start Ollama server: `ollama serve` (runs on `localhost:11434` by default). Add Ollama server start to system startup or run manually before `scripts/start.py`. | | |
| TASK-006 | **Pull DeepSeek-R1:8b model.** Run: `ollama pull deepseek-r1:8b`. This downloads ~4.9 GB. Verify model is available: `ollama list` should show `deepseek-r1:8b`. Run a smoke test: `ollama run deepseek-r1:8b "Reply with valid JSON: {\"status\": \"ok\"}"` — verify JSON output is returned. This is a one-time setup step. | | |
| TASK-007 | Create `scripts/verify_ollama.py` — a pre-flight check script: (1) `GET http://localhost:11434/api/tags` via `httpx`; (2) parse response JSON, check `models` list contains a model with `name` starting with `deepseek-r1:8b`; (3) if not found, print clear error: `"❌ deepseek-r1:8b not found. Run: ollama pull deepseek-r1:8b"` and `sys.exit(1)`; (4) send a test inference request with `{"model":"deepseek-r1:8b","prompt":"Reply with JSON: {\"test\":true}","format":"json","stream":false}`; (5) verify response parses as valid JSON; (6) print `"✅ Ollama + deepseek-r1:8b verified. Inference latency: {ms}ms"`. Called by `scripts/start.py` before launching uvicorn. | | |
| TASK-008 | Initialize project directory structure: `src/agents/`, `src/prompts/`, `src/utils/`, `src/models/`, `src/quant/`, `src/analytics/`, `tests/unit/`, `tests/integration/`, `plan/`, `scripts/`, `data/`, `dashboard/`. Create all `__init__.py` files. | | |
| TASK-009 | Create `src/utils/config.py` with `load_config() -> None` that calls `dotenv.load_dotenv()`. Expose typed accessors: `get_kite_api_key() -> str`, `get_kite_api_secret() -> str`, `get_kite_access_token() -> str`, `get_db_path() -> str`, `get_capital() -> float`, `get_ollama_base_url() -> str`, `get_decision_model() -> str`, `get_reflection_model() -> str`. All raise `ValueError` with descriptive message if key is missing or equals `PENDING`. | | |
| TASK-010 | Create `scripts/init_db.py` — connects to `data/scalpmaster.db` via `sqlite3`, enables WAL mode and 8 MB cache per GUD-008, then creates the full schema (TASK-011 through TASK-016). Run: `python scripts/init_db.py`. Set permissions: `chmod 600 data/scalpmaster.db`. | | |
| TASK-011 | In `scripts/init_db.py`, create `trades` table: `trade_id TEXT PK, symbol TEXT, direction TEXT, entry_time TEXT, entry_price REAL, tp_price REAL, sl_price REAL, quantity INTEGER, status TEXT, exit_price REAL, exit_time TEXT, pnl REAL, pnl_pct REAL, regime TEXT, ema_zone TEXT, bull_score INTEGER, bear_score INTEGER, strength_pct REAL, signal_confidence TEXT, llm_action TEXT, llm_reasoning TEXT, outcome TEXT, ev REAL, qm_atr_14 REAL, qm_atr_sl_price REAL, qm_atr_tp_price REAL, qm_atr_sl_exceeds_limit INTEGER, qm_kelly_fraction REAL, qm_kelly_quantity INTEGER, qm_kelly_fallback INTEGER, qm_garch_vol_forecast REAL, qm_garch_vol_historical_mean REAL, qm_garch_vol_ratio REAL, qm_garch_size_reduction INTEGER, qm_garch_engine TEXT, qm_ev REAL, qm_ev_positive INTEGER, qm_iv_percentile REAL, qm_iv_blocked INTEGER, qm_win_rate REAL, qm_avg_rr REAL, qm_trade_count_used INTEGER, spot_price REAL, option_ltp REAL, pcr REAL, dte INTEGER, iv REAL, created_at TEXT, updated_at TEXT`. | | |
| TASK-012 | In `scripts/init_db.py`, create `signal_audit` table — the **ML training dataset** per GUD-009. Columns: `audit_id TEXT PK, evaluated_at TEXT, symbol TEXT, direction TEXT, pipeline_outcome TEXT, block_reason TEXT, regime TEXT, ema_zone TEXT, bull_score INTEGER, bear_score INTEGER, strength_pct REAL, rsi_signal TEXT, rsi_value REAL, macd_signal TEXT, macd_value REAL, stoch_signal TEXT, stoch_value REAL, momentum_signal TEXT, momentum_value REAL, ema200_signal TEXT, ema200_value REAL, supertrend_signal TEXT, supertrend_value REAL, cci_signal TEXT, cci_value REAL, obv_roc_signal TEXT, obv_roc_value REAL, spot_price REAL, option_ltp REAL, pcr REAL, dte INTEGER, iv REAL, ev REAL, kelly_fraction REAL, garch_vol_forecast REAL, iv_percentile REAL, atr_14 REAL, llm_action TEXT, llm_confidence TEXT, llm_reasoning TEXT, llm_reasoning_trace TEXT, llm_inference_ms INTEGER, trade_id TEXT, created_at TEXT`. The `llm_reasoning_trace` column stores the full raw DeepSeek-R1 `<think>...</think>` block verbatim per GUD-010. | | |
| TASK-013 | In `scripts/init_db.py`, create `simulation_ticks` table: `id INTEGER PK AUTOINCREMENT, trade_id TEXT, symbol TEXT, timestamp TEXT, last_price REAL, volume INTEGER, oi INTEGER, created_at TEXT`. Index: `CREATE INDEX idx_sim_ticks_trade ON simulation_ticks(trade_id)`. | | |
| TASK-014 | In `scripts/init_db.py`, create `strategy_memory` table: `date TEXT PK, notes TEXT, threshold_adjustments TEXT, most_predictive_indicators TEXT, llm_model_accuracy TEXT, win_rate_7d REAL, win_rate_30d REAL, avg_ev_at_win REAL, avg_ev_at_loss REAL, updated_at TEXT`. Create `iv_history` table: `id INTEGER PK AUTOINCREMENT, symbol TEXT, date TEXT, iv REAL, logged_at TEXT, UNIQUE(symbol, date)`. Create `config` table: `key TEXT PK, value TEXT, updated_at TEXT`. Create `system_logs` table: `id INTEGER PK AUTOINCREMENT, level TEXT, message TEXT, context TEXT, created_at TEXT`. | | |
| TASK-015 | In `scripts/init_db.py`, create `daily_summary` table: `date TEXT PK, total_simulations INTEGER, executed INTEGER, skipped INTEGER, blocked INTEGER, wins INTEGER, losses INTEGER, forces INTEGER, win_rate REAL, total_pnl REAL, avg_pnl REAL, best_trade_id TEXT, worst_trade_id TEXT, avg_llm_inference_ms INTEGER, created_at TEXT`. | | |
| TASK-016 | Create `src/utils/db_client.py` with module-level singleton `aiosqlite` connection. Functions: `async def get_db() -> aiosqlite.Connection`, `async def db_execute(sql, params=())`, `async def db_fetchall(sql, params=()) -> list[dict]`, `async def db_fetchone(sql, params=()) -> dict | None`, `async def db_executemany(sql, params_list)`. Connection opened with `row_factory = aiosqlite.Row`. WAL mode set on first connection. | | |
| TASK-017 | Create `src/utils/tick_queue.py` with module-level `tick_queue: asyncio.Queue = asyncio.Queue(maxsize=500)` and `sim_tick_queues: dict[str, asyncio.Queue] = {}`. Functions: `async def enqueue_tick(tick: dict)` — puts on `tick_queue`, fans out to active `sim_tick_queues`; `def register_sim_queue(trade_id: str) -> asyncio.Queue`; `def deregister_sim_queue(trade_id: str)`. | | |
| TASK-018 | Create `scripts/start.py` — unified launcher: (1) `load_dotenv()`, (2) validate all `.env` keys, (3) `python scripts/init_db.py` (idempotent), (4) **call `scripts/verify_ollama.py`** — abort if Ollama or model not ready per REQ-022, (5) start APScheduler with jobs from `scripts/scheduler.py`, (6) start `TickerAgent` in `threading.Thread(daemon=True)`, (7) `uvicorn.run("src.main:app", host="127.0.0.1", port=8080, workers=1, limit_concurrency=4)`. | | |
| TASK-019 | Create `scripts/daily_auth.py` — CLI script: (1) reads `api_key` and `api_secret` from `.env`, (2) prints Kite login URL, (3) accepts `--request-token` arg, (4) calls `kite.generate_session()`, (5) writes `access_token` to `.env` via `dotenv.set_key(".env", "KITE_ACCESS_TOKEN", token)`, (6) prints "✅ Token saved. Valid until midnight IST." | | |
| TASK-020 | Create `scripts/scheduler.py` with `AsyncIOScheduler`. Jobs: (a) `instrument_refresh` — cron `hour=8, minute=0, day_of_week="mon-fri"` IST; (b) `auth_reminder` — cron `hour=9, minute=0` IST, `plyer.notification.notify(title="ScalpMaster", message="Run: python scripts/daily_auth.py")`; (c) `force_close_all` — cron `hour=15, minute=15` IST; (d) `early_close_guard` — cron `hour=15, minute=10` IST; (e) `daily_cleanup` — cron `hour=15, minute=30` IST, runs cleanup SQL per CON-006; (f) `write_daily_summary` — cron `hour=15, minute=35` IST. | | |

### Implementation Phase 1 — Kite Read-Only Integration Layer

- GOAL-002: Implement the complete Kite Connect v3 **read-only** integration: authentication, instrument token management, WebSocket tick streaming to `tick_queue`, and OHLCV historical data fetching. No order API calls exist anywhere in this codebase.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-021 | Create `src/utils/kite_client.py` with `get_kite() -> KiteConnect`. Reads `api_key` and `access_token` from `.env` via `get_config()`. Raises `ValueError` if `access_token == "PENDING"`. Returns configured `KiteConnect` instance. Used **exclusively for read operations**: `kite.quote()`, `kite.instruments()`, `kite.historical_data()`. | | |
| TASK-022 | Create `src/utils/rate_limiter.py` with `TokenBucketRateLimiter(capacity=3, refill_rate=3.0)`. Module-level singleton `kite_limiter`. All Kite REST calls must call `await kite_limiter.acquire()` before execution. | | |
| TASK-023 | Create `src/utils/instrument_manager.py` with `async def refresh_instruments() -> dict[str, int]`: (1) `await kite_limiter.acquire()`, (2) `kite.instruments("NFO")`, (3) filter for weekly expiry CE/PE for NIFTY/BANKNIFTY within ±500/±1000 points of spot, (4) serialize to JSON and write to SQLite `config` table key `instruments`, (5) return `tradingsymbol -> instrument_token` dict. Fallback: if Kite unavailable, load `data/instruments_cache.json`. | | |
| TASK-024 | Create `src/agents/ticker_agent.py` with class `TickerAgent`. Reads instrument tokens from SQLite `config`. Instantiates `KiteTicker(api_key, access_token)`. `on_connect`: subscribes with `ws.set_mode(ws.MODE_FULL, token_list)`. `on_ticks`: calls `enqueue_tick(tick)` for each tick. `on_reconnect`: re-subscribes. `on_noreconnect`: writes CRITICAL to SQLite `system_logs` and calls `os._exit(1)`. **No order-related callbacks.** | | |
| TASK-025 | Create `src/utils/ohlcv_fetcher.py` with `async def fetch_ohlcv(instrument_token: int, interval: str, lookback_bars: int = 220) -> list[dict]`. Calls `kite.historical_data()` wrapped in `kite_limiter.acquire()`. Hard cap at 220 bars per CON-010. Returns list of dicts: `date, open, high, low, close, volume`. | | |
| TASK-026 | Create `src/utils/quote_fetcher.py` with `async def fetch_quote(symbols: list[str]) -> dict`. Calls `kite.quote(symbols)` wrapped in `kite_limiter.acquire()`. Used to fetch spot price, India VIX, and option LTP. **Read-only.** | | |

### Implementation Phase 2 — Signal Scoring Engine

- GOAL-003: Implement all 8 indicator scoring functions as pure, stateless, memory-efficient Python functions. Each function returns both the signal direction and the raw indicator value for storage in `signal_audit`.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-027 | Create `src/models/signal_models.py` with Pydantic models: `OHLCVBar(date: datetime, open: float, high: float, low: float, close: float, volume: int)`, `IndicatorSignal(name: str, value: float, signal: Literal["bull","bear","neutral"])`, `ScoreResult(bull_score: int, bear_score: int, strength_pct: float, signals: dict[str, IndicatorSignal], ema_zone: Literal["UP","DOWN","NEUTRAL"])`. | | |
| TASK-028 | Create `src/utils/bar_converter.py` with `bars_to_numpy(bars: list[OHLCVBar]) -> dict[str, np.ndarray]`. Returns `open`, `high`, `low`, `close`, `volume` as `np.float32` arrays per GUD-007. Single conversion point for all indicator functions. | | |
| TASK-029 | Implement `score_rsi(arrays) -> IndicatorSignal` in `src/agents/signal_agent.py`. Import `pandas_ta` inside function body. RSI-14. `bull` if `rsi < 40`, `bear` if `rsi > 60`. Store raw RSI value. `del` intermediates, `gc.collect()` after. | | |
| TASK-030 | Implement `score_macd(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. MACD(12,26,9). `bull` if `macd > signal`, `bear` if below. Store `macd - signal` as value. | | |
| TASK-031 | Implement `score_stochastic(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. Stochastic %K(14,3). `bull` if `%K < 20`, `bear` if `%K > 80`. | | |
| TASK-032 | Implement `score_momentum(arrays) -> IndicatorSignal`. Pure numpy: `momentum = float(arrays["close"][-1] - arrays["close"][-11])`. `bull` if positive, `bear` if negative. | | |
| TASK-033 | Implement `score_ema200(arrays) -> IndicatorSignal` and `compute_ema_zone(arrays, zone_pct=0.002) -> Literal["UP","DOWN","NEUTRAL"]`. Import `pandas_ta` inside function body. EMA-200. Zone: `UP` if `close > ema * 1.002`, `DOWN` if `close < ema * 0.998`. | | |
| TASK-034 | Implement `score_supertrend(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. SuperTrend(10, 3.0). `bull` if `close > supertrend`, `bear` if below. | | |
| TASK-035 | Implement `score_cci(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. CCI-20. `bull` if `cci < -100`, `bear` if `cci > 100`. | | |
| TASK-036 | Implement `score_obv_roc(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. OBV then 5-bar ROC. `bull` if positive, `bear` if negative. | | |
| TASK-037 | Implement `async def compute_scores(bars: list[OHLCVBar]) -> ScoreResult`. Call `bars_to_numpy()` once. Run all 8 scoring functions concurrently via `asyncio.gather()` + `run_in_executor`. Aggregate `bull_score`, `bear_score`, `strength_pct`. Compute `ema_zone`. `del arrays`, `gc.collect()`. Return `ScoreResult`. | | |

### Implementation Phase 3 — Ollama LLM Client & Decision Agent

- GOAL-004: Implement the Ollama LLM client for DeepSeek-R1:8b, the Decision Agent, and the Risk Guard Agent. The LLM client handles DeepSeek-R1's `<think>` block parsing, JSON extraction, timeout enforcement, and `BLOCKED_LLM` fallback. No cloud LLM code exists anywhere in this codebase.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-038 | Create `src/utils/ollama_client.py` — the sole LLM interface. Implement `async def call_llm(prompt: str, model: str, temperature: float, num_predict: int, timeout: float) -> tuple[str, str, int]` returning `(json_answer, reasoning_trace, inference_ms)`. Logic: (1) record `start_ms = time.monotonic()`; (2) POST to `{OLLAMA_BASE_URL}/api/generate` via `httpx.AsyncClient(timeout=timeout)` with body `{"model": model, "prompt": prompt, "stream": false, "format": "json", "options": {"temperature": temperature, "num_predict": num_predict}}`; (3) on `httpx.TimeoutException` → raise `OllamaTimeoutError(f"Inference exceeded {timeout}s")`; (4) on HTTP error → raise `OllamaConnectionError`; (5) extract `raw_response = response.json()["response"]`; (6) call `_parse_deepseek_response(raw_response)` → returns `(json_str, think_block)`; (7) `inference_ms = int((time.monotonic() - start_ms) * 1000)`; (8) return `(json_str, think_block, inference_ms)`. | | |
| TASK-039 | Implement `_parse_deepseek_response(raw: str) -> tuple[str, str]` in `src/utils/ollama_client.py`. DeepSeek-R1 outputs responses in the format: `<think>\n{chain_of_thought}\n</think>\n{json_answer}`. Logic: (1) use `re.search(r"<think>(.*?)</think>", raw, re.DOTALL)` to extract `think_block`; (2) if match found, `json_str = raw[match.end():].strip()`; (3) if no `<think>` tag found (model skipped reasoning), `think_block = ""`, `json_str = raw.strip()`; (4) strip any residual markdown fences from `json_str`: `json_str = re.sub(r"^```json\s*|\s*```$", "", json_str, flags=re.MULTILINE).strip()`; (5) return `(json_str, think_block)`. | | |
| TASK-040 | Create `src/prompts/prompts.py` with string constants written specifically for DeepSeek-R1's instruction format. `DECISION_AGENT_PROMPT_V1`: opens with `"You are a quantitative options trading analyst. Analyze the following NIFTY/BANKNIFTY signal data and output ONLY a valid JSON object — no explanation outside the JSON.\n\n"`, followed by all signal fields as a structured block, ending with `"\nOutput format (JSON only, no markdown):\n{\"action\": \"EXECUTE|SKIP|WAIT\", \"confidence\": \"FULL_CONFIDENCE|LOW_CONFIDENCE\", \"reasoning\": \"max 3 sentences\", \"entry_price\": float, \"tp_price\": float, \"sl_price\": float}"`. Placeholders: `{symbol}`, `{spot_price}`, `{option_ltp}`, `{bull_score}`, `{bear_score}`, `{strength_pct}`, `{ema_zone}`, `{signals_json}`, `{success_rate}`, `{regime}`, `{dte}`, `{iv}`, `{pcr}`, `{ev}`, `{kelly_fraction}`, `{garch_vol}`, `{iv_percentile}`, `{atr_14}`. | | |
| TASK-041 | Add `REFLECTION_AGENT_PROMPT_V1` to `src/prompts/prompts.py`. Opens with `"You are a systematic trading strategy analyst performing post-trade analysis. Review the following completed simulated trade and output ONLY a valid JSON object.\n\n"`. Placeholders: `{trade_json}`, `{outcome}`, `{regime}`, `{bull_score}`, `{ema_zone}`, `{kelly_fraction}`, `{garch_vol}`, `{ev}`, `{iv_percentile}`, `{win_rate_7d}`, `{win_rate_30d}`. Output format: `{"entry_reasoning_correct": bool, "most_predictive_indicators": ["list"], "threshold_adjustment": {"indicator": delta}, "memory_note": "string", "llm_model_accuracy": "string"}`. | | |
| TASK-042 | Create `src/models/decision_models.py` with `DecisionInput(symbol, spot_price, option_ltp, score_result: ScoreResult, success_rate, regime, dte, iv, pcr, quant_metrics: QuantMetrics)` and `DecisionOutput(action: Literal["EXECUTE","SKIP","WAIT"], confidence: Literal["FULL_CONFIDENCE","LOW_CONFIDENCE"], reasoning: str, entry_price: float, tp_price: float, sl_price: float)`. `QuantMetrics` defined in TASK-055. | | |
| TASK-043 | Create `src/agents/decision_agent.py` with `async def run_decision_agent(input: DecisionInput) -> tuple[DecisionOutput, str, int]` returning `(decision, reasoning_trace, inference_ms)`. Logic: (1) format `DECISION_AGENT_PROMPT_V1` with all input fields; (2) call `call_llm(prompt, model=get_decision_model(), temperature=0.1, num_predict=768, timeout=45.0)`; (3) on `OllamaTimeoutError` or `OllamaConnectionError` → log to `system_logs`, raise `LLMUnavailableError`; (4) parse `json_str` into `DecisionOutput` via `pydantic.model_validate_json()`; (5) on parse failure: retry once with `DECISION_AGENT_PROMPT_V1` (same prompt, model may self-correct on second attempt); (6) on second failure: raise `LLMParseError`; (7) return `(decision, reasoning_trace, inference_ms)`. The `reasoning_trace` (full `<think>` block) is passed up to the orchestrator for storage in `signal_audit`. | | |
| TASK-044 | Create `src/agents/risk_guard_agent.py` with `async def check_risk(current_time: datetime) -> tuple[bool, str]`. Hard-block rules: (a) query SQLite `SELECT COALESCE(SUM(pnl),0) FROM trades WHERE date(entry_time)=date('now')` for `daily_pnl`; if `daily_pnl <= -(capital * 0.02)` → block `"Daily loss limit 2% breached"`; (b) `current_time >= 15:15 IST` → block `"Past MIS cutoff"`; (c) `current_time < 09:15 IST` → block `"Before market open"`; (d) fetch India VIX via `fetch_quote(["NSE:INDIA VIX"])`, if `vix > 25.0` → block `"India VIX > 25"`. Return `(True, "APPROVED")` only if all pass. **No Kite order API calls.** | | |
| TASK-045 | Create `src/utils/market_utils.py` with: (a) `async def compute_pcr(kite, nifty_spot) -> float`; (b) `async def detect_regime(bars) -> tuple[str, float]` returning `(regime_string, atr_14)`; (c) `async def get_success_rate() -> float` — queries SQLite last 20 closed trades; (d) `async def get_avg_rr() -> float` — queries SQLite last 20 closed trades. | | |

### Implementation Phase 4 — Quantitative Risk & Alpha Layer

- GOAL-005: Implement the institutional-grade Quant Risk Agent using pure NumPy GARCH, SQLite-backed Kelly, ATR-14, EV gating, and IV percentile filtering. All computations are fully local — no external libraries beyond numpy.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-046 | Create `src/quant/garch_lite.py` with `fit_garch_lite(returns: np.ndarray) -> tuple[float, float, float, float]` returning `(sigma_forecast, omega, alpha, beta)`. Pure NumPy GARCH(1,1): (1) `var_0 = np.var(returns)`; (2) params `omega=1e-6, alpha=0.1, beta=0.85`; (3) 50-iteration coordinate descent: `h[t] = omega + alpha * r[t-1]**2 + beta * h[t-1]`; (4) log-likelihood `LL = -0.5 * sum(log(h) + r**2/h)`; (5) return `(sqrt(omega + alpha*r[-1]**2 + beta*h[-1]), omega, alpha, beta)`. Uses `float64` per GUD-007 exception. Memory: ~1.6 KB. | | |
| TASK-047 | Create `src/models/quant_models.py` with `QuantMetrics(atr_14: float, atr_sl_price: float, atr_tp_price: float, atr_sl_exceeds_limit: bool, kelly_fraction: float, kelly_quantity: int, kelly_fallback: bool, garch_vol_forecast: float, garch_vol_historical_mean: float, garch_vol_ratio: float, garch_size_reduction: bool, garch_engine: Literal["garch_lite","ewma_fallback"], ev: float, ev_positive: bool, iv_percentile: float, iv_blocked: bool, win_rate: float, avg_rr: float, trade_count_used: int)`. | | |
| TASK-048 | Create `src/quant/atr_calculator.py` with `compute_atr_sl_tp(arrays, entry_price, direction, multiplier_sl=1.5, multiplier_tp=3.0) -> tuple[float, float, float]`. Pure NumPy ATR-14: `TR = np.maximum(H-L, np.abs(H-prev_C), np.abs(L-prev_C))`, ATR via EWM with `span=14`. Validate `abs(entry - sl) / entry <= 0.02`. | | |
| TASK-049 | Create `src/quant/kelly_calculator.py` with `async def compute_kelly(capital: float, max_lots: int = 5) -> tuple[float, int, bool]`. Query SQLite: `SELECT outcome, pnl FROM trades WHERE status != 'OPEN' ORDER BY entry_time DESC LIMIT 20`. If `count < 10` → return `(0.0, 1, True)`. Compute `W`, `R`, `f*`, `f_half`. Clamp to `[0.01, 0.20]`. | | |
| TASK-050 | Create `src/quant/garch_forecaster.py` with `compute_garch_forecast(arrays) -> tuple[float, float, float, str]`. Compute log returns (float64). Call `fit_garch_lite()` in `ThreadPoolExecutor` with 0.2s timeout per CON-007. On timeout: `sigma = np.std(returns[-20:]) * np.sqrt(375)`, `engine = "ewma_fallback"`. Return `(sigma_forecast, sigma_historical_mean, vol_ratio, engine)`. | | |
| TASK-051 | Create `src/quant/ev_calculator.py` with `compute_ev(win_rate: float, tp_distance: float, sl_distance: float) -> float`. `EV = (win_rate * tp_distance) - ((1 - win_rate) * sl_distance)`. Pure Python. | | |
| TASK-052 | Create `src/quant/iv_filter.py` with: (a) `async def compute_iv_percentile(symbol: str, current_iv: float) -> float` — queries SQLite `SELECT iv FROM iv_history WHERE symbol=? ORDER BY date DESC LIMIT 30`, computes `np.searchsorted(np.sort(iv_array), current_iv) / len(iv_array) * 100`; (b) `async def log_iv(symbol: str, iv: float)` — `INSERT OR REPLACE INTO iv_history`. | | |
| TASK-053 | Create `src/agents/quant_risk_agent.py` with `async def run_quant_risk_agent(bars, entry_price, direction, current_iv, symbol, capital) -> tuple[bool, str, QuantMetrics]`. Orchestrates: ATR → Kelly → GARCH (with size reduction if `vol_ratio > 2.0`) → EV → IV percentile. Block conditions: `atr_sl_exceeds_limit` → `"BLOCKED_ATR"`; `ev <= 0` → `"BLOCKED_EV"`; `iv_percentile > 80` → `"BLOCKED_IV"`. Return `(True, "APPROVED", quant_metrics)` if all pass. | | |
| TASK-054 | Create `src/quant/iv_history_seeder.py` — one-time script to backfill 30 days of IV history into SQLite `iv_history`. Run once: `python src/quant/iv_history_seeder.py`. | | |

### Implementation Phase 5 — Simulation Execution & Monitor Agents

- GOAL-006: Implement the Simulation Execution Agent (creates simulated trade records in SQLite with synthetic order IDs) and the Simulation Monitor Agent (tracks live tick prices against simulated TP/SL levels, writes outcome to SQLite, stores every tick during the open position).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-055 | Create `src/models/trade_models.py` with `TradeRecord` Pydantic model matching all columns of the SQLite `trades` table. Include `quant_metrics: QuantMetrics | None` as a transient field — flattened to `qm_*` columns on SQLite write. | | |
| TASK-056 | Create `src/agents/simulation_execution_agent.py` with `async def execute_simulation(decision: DecisionOutput, symbol: str, quantity: int, direction: Literal["BUY","SELL"], score_result: ScoreResult, regime: str, quant_metrics: QuantMetrics, spot_price: float, option_ltp: float, pcr: float, dte: int, iv: float) -> TradeRecord`. Actions: (1) generate `trade_id = f"SIM_{symbol}_{datetime.now(utc).strftime('%Y%m%d%H%M%S%f')}"`, (2) generate synthetic `entry_order_id = f"SIM_ENTRY_{uuid4().hex[:8]}"`, (3) set `entry_price = option_ltp`, (4) construct `TradeRecord` with `status = "OPEN"`, (5) INSERT all fields including all `qm_*` columns into SQLite `trades`, (6) call `register_sim_queue(trade_id)`, (7) return `TradeRecord`. **No Kite order API calls.** | | |
| TASK-057 | Create `src/agents/simulation_monitor_agent.py` with `async def monitor_simulation(trade: TradeRecord) -> TradeRecord`. Subscribes to `sim_tick_queues[trade.trade_id]`. On each tick: (1) INSERT tick into `simulation_ticks`; (2) check TP hit → `outcome = "TP"`; (3) check SL hit → `outcome = "SL"`; (4) trailing SL: if BUY and `ltp >= entry * 1.01` → move SL to `entry`; if `ltp >= entry * 1.015` → move SL to `entry * 1.005`; (5) force-close at 15:15 IST → `outcome = "FORCE"`; (6) on close: compute `pnl`, `pnl_pct`, UPDATE SQLite `trades`, call `deregister_sim_queue(trade_id)`, return closed `TradeRecord`. | | |
| TASK-058 | Create `src/utils/force_close.py` with `async def force_close_all_simulations() -> None`. Queries SQLite for all `OPEN` trades. For each: fetches current LTP via `fetch_quote()`, sets `outcome = "FORCE"`, computes `pnl`, UPDATEs SQLite. Calls `deregister_sim_queue(trade_id)`. | | |

### Implementation Phase 6 — Reflection Agent & Audit Writer

- GOAL-007: Implement the Reflection Agent (DeepSeek-R1:8b post-trade analysis → SQLite `strategy_memory`) and the Audit Writer (writes complete ML-ready row to `signal_audit` for every pipeline evaluation including the full `<think>` reasoning trace).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-059 | Create `src/agents/audit_writer.py` with `async def write_signal_audit(audit_id: str, evaluated_at: str, symbol: str, direction: str, pipeline_outcome: str, block_reason: str | None, score_result: ScoreResult, market_context: dict, quant_metrics: QuantMetrics | None, decision: DecisionOutput | None, reasoning_trace: str | None, inference_ms: int | None, trade_id: str | None) -> None`. Constructs a flat dict from all inputs and INSERTs into SQLite `signal_audit` table. The `llm_reasoning_trace` column receives `reasoning_trace` verbatim per GUD-010. Called at the end of every pipeline cycle regardless of outcome. | | |
| TASK-060 | Create `src/agents/reflection_agent.py` with `async def run_reflection(trade: TradeRecord) -> None`. (1) Compute `win_rate_7d` and `win_rate_30d` from SQLite. (2) Format `REFLECTION_AGENT_PROMPT_V1` with full trade JSON. (3) Call `call_llm(prompt, model=get_reflection_model(), temperature=0.3, num_predict=1536, timeout=90.0)`. (4) On `OllamaTimeoutError`: log WARN to `system_logs` with message `"Reflection LLM timeout — skipping memory update for {trade_id}"` and return without writing to `strategy_memory`. (5) Parse JSON response. (6) `INSERT OR REPLACE INTO strategy_memory` with merged fields and `updated_at`. | | |

### Implementation Phase 7 — Orchestrator & FastAPI Server

- GOAL-008: Wire all agents into the sequential pipeline, implement all FastAPI endpoints including analytics, and configure APScheduler for daily lifecycle management.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-061 | Create `src/orchestrator.py` with `async def run_pipeline(tick_data: dict) -> dict` and `async def consume_tick_queue() -> None`. Full pipeline per PAT-001: (1) `fetch_ohlcv()`, (2) `compute_scores()`, (3) regime check per GUD-006 → if blocked, `write_signal_audit(outcome="BLOCKED_REGIME",
