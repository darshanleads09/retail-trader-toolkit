---
goal: Build AI-Powered ScalpMaster Pro — Real-Time Paper Trading Simulator for NIFTY/BANKNIFTY Options
version: 4.0
date_created: 2026-08-26
last_updated: 2026-08-31
owner: TraderMan (CMT | CQF | Senior Hedge Fund Manager)
status: 'In progress'
tags: [feature, simulation, paper-trading, local-dev, sqlite, agentic, finance, trading, quantitative, risk-management, data-collection, ml-ready]
---

# Introduction

![Status: In progress](https://img.shields.io/badge/status-In%20progress-yellow)

This plan describes the end-to-end implementation of **ScalpMaster Pro v4.0** — a **real-time paper trading simulator** that connects to the live Zerodha Kite Connect v3 API for **read-only market data** (WebSocket ticks, OHLCV, option chains, quotes) and simulates the complete NIFTY/BANKNIFTY options scalping pipeline **without placing a single real order**.

**All cloud infrastructure has been removed.** There is no GCP, no Docker, no Cloud Run, no Firestore, no Pub/Sub, no Secret Manager, no Cloud Scheduler. The entire system runs as a single Python process on `localhost:8080` with **SQLite as the sole persistence layer** at `data/scalpmaster.db`.

**Purpose of this version:** Accumulate a statistically significant audit trail of simulated trades — every signal, every decision, every simulated entry/exit, every quant metric — so that win probability can be calculated, market patterns can be identified, and a predictive ML model can be trained in a future version.

**What the system does:**
- Connects to live KiteTicker WebSocket → receives real-time option price ticks
- Fetches real OHLCV data from Kite `historical_data()` API
- Runs the full 8-indicator signal scoring engine on live data
- Calls Gemini Flash to reason over the signal and produce a trade decision
- Simulates order entry, SL, TP, and trailing SL against live tick prices
- Monitors the simulated position in real-time against the live feed
- Runs the Reflection Agent after each closed simulation
- Writes a **complete, ML-ready audit record** to SQLite for every trade lifecycle event

**What the system does NOT do:**
- Place any real order on Zerodha
- Write to any cloud service
- Require Docker, GCP credentials, or any cloud SDK

The agent incorporates: **Kelly Criterion** position sizing (simulated), **GARCH(1,1)** intraday volatility forecasting, **ATR-14 dynamic stop-loss** calibration, **Expected Value (EV) gating**, **IV Percentile filtering**, and **regime-conditional signal thresholds**.

---

## 1. Requirements & Constraints

- **REQ-001**: The system must replicate all 8 ScalpMaster Pro indicators: RSI-14, MACD(12,26,9), Stochastic-%K, Momentum-10, EMA-200, SuperTrend(ATR), CCI-20, OBV-ROC — each returning a discrete `bull` or `bear` signal per evaluation cycle.
- **REQ-002**: A STRONG BUY signal requires `bull_score >= 6 AND bull_score > bear_score AND ema_zone == "UP"`. A STRONG SELL signal requires `bear_score >= 6 AND bear_score > bull_score AND ema_zone == "DOWN"`.
- **REQ-003**: Default risk/reward ratio must be 1:2 — Stop Loss at 1% from simulated entry, Take Profit at 2% from simulated entry. When ATR-based SL (REQ-011) is active, the fixed 1% SL is superseded.
- **REQ-004**: All simulated trades must use product type `MIS` semantics. All open simulated positions must be force-closed at 15:15 IST daily by the Simulation Monitor.
- **REQ-005**: The Gemini Decision Agent must return a structured JSON response: `action` (EXECUTE|SKIP|WAIT), `confidence` (FULL_CONFIDENCE|LOW_CONFIDENCE), `reasoning` (string, max 3 sentences), `entry_price` (float), `tp_price` (float), `sl_price` (float).
- **REQ-006**: The Reflection Agent must run after every closed simulated trade and write an updated `strategy_memory` row to SQLite within 30 seconds of simulated trade closure.
- **REQ-007**: The system must support both NIFTY (lot size 50) and BANKNIFTY (lot size 15) option chains on the NFO segment.
- **REQ-008**: The instrument token list must be refreshed daily at 08:00 IST from `kite.instruments("NFO")`, filtering for weekly expiry CE and PE contracts within ±500 points of spot for NIFTY and ±1000 points for BANKNIFTY. Results stored in SQLite `config` table.
- **REQ-009**: The Kite Connect `access_token` must be renewed daily before 09:00 IST and written to `.env` under key `KITE_ACCESS_TOKEN` using `dotenv.set_key()`.
- **REQ-010**: The agent must enqueue every WebSocket tick from KiteTicker `MODE_FULL` into the in-process `asyncio.Queue` named `tick_queue` (maxsize=500). The Orchestrator consumes from this queue to drive the pipeline.
- **REQ-011**: The Quant Risk Layer must compute ATR-14 on the underlying spot at signal time. Dynamic SL = `entry_price - (1.5 × ATR_14)` for BUY, `entry_price + (1.5 × ATR_14)` for SELL. TP = `entry_price ± (3.0 × ATR_14)`. If ATR-based SL exceeds 2% of entry price, the simulation must be blocked and logged as `BLOCKED_ATR`.
- **REQ-012**: The Quant Risk Layer must compute **Kelly Criterion** optimal simulated position size: `f* = (W × R - (1 - W)) / R`. Apply half-Kelly (`0.5 × f*`). Minimum: 1 lot. Maximum: 5 lots. `W` and `R` derived from last 20 closed simulated trades in SQLite `trades` table.
- **REQ-013**: The Quant Risk Layer must fit a **GARCH(1,1)** model on the last 100 one-minute close returns of the underlying spot. Use the pure NumPy implementation in `src/quant/garch_lite.py` (zero extra packages). On fitting failure, fall back to `np.std(returns[-20:]) * np.sqrt(375)` and log `garch_engine = "ewma_fallback"` in the trade audit record.
- **REQ-014**: The Decision Agent must compute and log **Expected Value (EV)**: `EV = (W × TP_distance) - ((1 - W) × SL_distance)`. Simulations with `EV <= 0` must be blocked and logged as `BLOCKED_EV`.
- **REQ-015**: The system must apply an **IV Percentile filter**: block simulation entry if `IV_percentile > 80`. Log as `BLOCKED_IV`.
- **REQ-016**: The Simulation Monitor must track simulated position P&L in real-time by comparing `current_ltp` (from live tick feed) against `tp_price` and `sl_price`. It must NOT call `kite.positions()` or any Kite order API — it reads only from the live tick queue and the SQLite `trades` table.
- **REQ-017**: Every pipeline evaluation — including SKIP and BLOCKED outcomes — must write a complete audit row to the SQLite `signal_audit` table. This is the primary data source for future ML model training. No evaluation may be silently discarded.
- **REQ-018**: The SQLite `trades` table must store all fields required for post-hoc win probability analysis: all 8 indicator signals, all quant metrics, Gemini reasoning, regime, PCR, DTE, IV, simulated entry/exit prices, simulated P&L, outcome, and timestamps with millisecond precision.
- **REQ-019**: The system must expose a `GET /analytics/summary` endpoint that queries SQLite and returns: total simulations, win rate, average P&L per trade, average EV at entry, win rate by regime, win rate by bull_score, win rate by hour-of-day, most predictive indicator (by correlation with outcome).
- **REQ-020**: Total Python process RAM must not exceed **512 MB**. Monitor via `psutil` in `/health` endpoint. Warn in logs if > 400 MB.
- **SEC-001**: The Kite `api_key` and `api_secret` must be stored exclusively in `.env` at project root (git-ignored). They must never appear in source code.
- **SEC-002**: The FastAPI server must bind exclusively to `127.0.0.1` (localhost). Never `0.0.0.0`.
- **SEC-003**: The SQLite database file `data/scalpmaster.db` must have file permissions `600` (owner read/write only). Set via `chmod 600 data/scalpmaster.db` after creation.
- **SEC-004**: The Risk Guard Agent must enforce hard-block rules that cannot be overridden by Gemini output under any circumstances.
- **CON-001**: Kite Connect REST API rate limit is 3 requests/second. All REST calls must be throttled using `TokenBucketRateLimiter(capacity=3, refill_rate=3.0)`.
- **CON-002**: KiteTicker WebSocket subscriptions must not exceed 200 instruments per session to limit tick queue memory pressure.
- **CON-003**: The `access_token` expires daily at midnight IST. `scripts/daily_auth.py` must be run before 09:00 IST each trading day.
- **CON-004**: The `tick_queue` must have `maxsize=500`. On overflow, drop the incoming tick and log a WARN to SQLite `system_logs`. This prevents unbounded RAM growth during high-frequency bursts.
- **CON-005**: Gemini Flash must be used for the Decision Agent. Gemini Pro must be used for the Reflection Agent. Both called via `google-generativeai` REST SDK using `GEMINI_API_KEY` from `.env`.
- **CON-006**: SQLite `data/scalpmaster.db` must not exceed **100 MB** on disk. A daily cleanup job (APScheduler, 15:30 IST) deletes `system_logs` rows older than 7 days and `signal_audit` rows older than 90 days. `trades` rows are never deleted — they are the permanent ML training dataset.
- **CON-007**: GARCH(1,1) fitting via `garch_lite.py` must complete within 200ms (50-iteration coordinate descent on 100 returns). If it exceeds 200ms, fall back to EWMA and log the event.
- **CON-008**: Kelly Criterion requires minimum 10 completed simulated trades in SQLite `trades`. If fewer, default to 1 lot and log `kelly_fallback = 1`.
- **CON-009**: Total installed venv disk size must not exceed **1.2 GB**. No cloud SDKs, no Docker, no Node.js.
- **CON-010**: Peak RAM must not exceed **512 MB**. OHLCV lookback hard-capped at 220 bars. `pandas` imported inside function bodies only. `gc.collect()` called after each pipeline cycle.
- **CON-011**: The uvicorn server must run with `--workers 1 --limit-concurrency 4`.
- **GUD-001**: All agent modules must be independently importable under `src/agents/`. No direct cross-agent imports. Communication via `asyncio.Queue` or SQLite reads only.
- **GUD-002**: All SQLite writes must include `created_at` and `updated_at` as ISO-8601 UTC strings with millisecond precision: `datetime.now(timezone.utc).isoformat(timespec="milliseconds")`.
- **GUD-003**: All Gemini prompts must be stored as versioned string constants in `src/prompts/prompts.py`.
- **GUD-004**: Every pipeline evaluation — EXECUTE, SKIP, WAIT, BLOCKED — must write to `signal_audit` table per REQ-017.
- **GUD-005**: All quantitative model outputs must be stored as dedicated columns in the `trades` and `signal_audit` tables (not as JSON blobs) to enable direct SQL analytics queries per REQ-019.
- **GUD-006**: Regime-conditional thresholds: `volatile` regime requires `bull_score >= 7`; `ranging` regime skips all simulations.
- **GUD-007**: All numpy arrays in indicator computation must use `dtype=np.float32`. Exception: GARCH log-return arrays must use `float64`.
- **GUD-008**: SQLite must be opened with WAL mode (`PRAGMA journal_mode=WAL`) and `PRAGMA cache_size=-8000` (8 MB page cache).
- **GUD-009**: The `signal_audit` table is the **primary ML training dataset**. Every row must be self-contained — it must include all features (indicator values, quant metrics, market context) and the label (`outcome`: TP/SL/FORCE/BLOCKED/SKIP) so that a future model can be trained with a single `SELECT *` query.
- **PAT-001**: Sequential agent pipeline: Signal Agent → Decision Agent → Risk Guard Agent → Quant Risk Agent → Simulation Execution Agent → Simulation Monitor Agent → Reflection Agent.
- **PAT-002**: Parallel indicator scoring within Signal Agent using `asyncio.gather()` + `run_in_executor`.
- **PAT-003**: SQLite is the single source of truth for all agent state. No in-memory state persists across pipeline invocations except `tick_queue`.
- **PAT-004**: Quant Risk Agent is a gate between Risk Guard and Simulation Execution. It either enriches the decision with `quantity`, `sl_price`, `tp_price` or returns `BLOCKED` with reason.
- **PAT-005**: The Simulation Monitor reads live ticks from `tick_queue` (not from Kite API) to determine TP/SL hits. It writes outcome updates to SQLite `trades` table.

---

## 2. Implementation Steps

### Implementation Phase 0 — Project Bootstrap & Environment

- GOAL-001: Initialize the project repository, Python virtual environment, `.env` configuration, SQLite schema, and APScheduler process so all subsequent phases have a working local runtime with zero external dependencies beyond Kite API and Gemini API.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-001 | Create project root `scalpmaster-pro/`. Run `git init`. Create `.gitignore` with entries: `.env`, `data/`, `__pycache__/`, `*.pyc`, `.venv/`, `*.db`, `*.db-wal`, `*.db-shm`. | | |
| TASK-002 | Create Python 3.11 virtual environment: `python3.11 -m venv .venv`. Activate: `source .venv/bin/activate`. Verify: `python --version` returns `3.11.x`. | | |
| TASK-003 | Create `.env` at project root with keys: `KITE_API_KEY=`, `KITE_API_SECRET=`, `KITE_ACCESS_TOKEN=PENDING`, `GEMINI_API_KEY=`, `DB_PATH=data/scalpmaster.db`, `TRADING_CAPITAL=500000`, `LOG_LEVEL=INFO`, `HOST=127.0.0.1`, `PORT=8080`. Create `.env.example` as a copy with all values replaced by placeholders. Add `.env` to `.gitignore`. | | |
| TASK-004 | Create `requirements.txt` with all dependencies: `kiteconnect>=5.0.1`, `google-generativeai>=0.7.0`, `pandas>=2.2.0`, `pandas-ta>=0.3.14b`, `numpy>=1.26.0`, `aiosqlite>=0.20.0`, `python-dotenv>=1.0.0`, `apscheduler>=3.10.4`, `fastapi>=0.111.0`, `uvicorn>=0.30.0`, `pydantic>=2.7.0`, `httpx>=0.27.0`, `psutil>=5.9.0`, `pytest>=8.2.0`, `pytest-asyncio>=0.23.0`. Run: `pip install -r requirements.txt`. Estimated venv disk size: ~750 MB. | | |
| TASK-005 | Initialize project directory structure: `src/agents/`, `src/prompts/`, `src/utils/`, `src/models/`, `src/quant/`, `src/analytics/`, `tests/unit/`, `tests/integration/`, `plan/`, `scripts/`, `data/`, `dashboard/`. Create all `__init__.py` files. | | |
| TASK-006 | Create `src/utils/config.py` with `load_config() -> dict` that calls `dotenv.load_dotenv()` and returns typed accessors: `get_kite_api_key() -> str`, `get_kite_api_secret() -> str`, `get_kite_access_token() -> str`, `get_gemini_api_key() -> str`, `get_db_path() -> str`, `get_capital() -> float`. All raise `ValueError` with descriptive message if key is missing or equals `PENDING`. | | |
| TASK-007 | Create `scripts/init_db.py` — connects to `data/scalpmaster.db` via `sqlite3`, enables WAL mode and 8 MB cache per GUD-008, then creates the full schema (see TASK-008 through TASK-013). Run: `python scripts/init_db.py`. Set permissions: `chmod 600 data/scalpmaster.db`. | | |
| TASK-008 | In `scripts/init_db.py`, create `trades` table with columns: `trade_id TEXT PK, symbol TEXT, direction TEXT, entry_time TEXT, entry_price REAL, tp_price REAL, sl_price REAL, quantity INTEGER, status TEXT, exit_price REAL, exit_time TEXT, pnl REAL, pnl_pct REAL, regime TEXT, ema_zone TEXT, bull_score INTEGER, bear_score INTEGER, strength_pct REAL, signal_confidence TEXT, gemini_action TEXT, gemini_reasoning TEXT, outcome TEXT, ev REAL, qm_atr_14 REAL, qm_atr_sl_price REAL, qm_atr_tp_price REAL, qm_atr_sl_exceeds_limit INTEGER, qm_kelly_fraction REAL, qm_kelly_quantity INTEGER, qm_kelly_fallback INTEGER, qm_garch_vol_forecast REAL, qm_garch_vol_historical_mean REAL, qm_garch_vol_ratio REAL, qm_garch_size_reduction INTEGER, qm_garch_engine TEXT, qm_ev REAL, qm_ev_positive INTEGER, qm_iv_percentile REAL, qm_iv_blocked INTEGER, qm_win_rate REAL, qm_avg_rr REAL, qm_trade_count_used INTEGER, spot_price REAL, option_ltp REAL, pcr REAL, dte INTEGER, iv REAL, created_at TEXT, updated_at TEXT`. | | |
| TASK-009 | In `scripts/init_db.py`, create `signal_audit` table — the **ML training dataset** per GUD-009. Columns: `audit_id TEXT PK, evaluated_at TEXT, symbol TEXT, direction TEXT, pipeline_outcome TEXT, block_reason TEXT, regime TEXT, ema_zone TEXT, bull_score INTEGER, bear_score INTEGER, strength_pct REAL, rsi_signal TEXT, rsi_value REAL, macd_signal TEXT, macd_value REAL, stoch_signal TEXT, stoch_value REAL, momentum_signal TEXT, momentum_value REAL, ema200_signal TEXT, ema200_value REAL, supertrend_signal TEXT, supertrend_value REAL, cci_signal TEXT, cci_value REAL, obv_roc_signal TEXT, obv_roc_value REAL, spot_price REAL, option_ltp REAL, pcr REAL, dte INTEGER, iv REAL, ev REAL, kelly_fraction REAL, garch_vol_forecast REAL, iv_percentile REAL, atr_14 REAL, gemini_action TEXT, gemini_confidence TEXT, gemini_reasoning TEXT, trade_id TEXT, created_at TEXT`. | | |
| TASK-010 | In `scripts/init_db.py`, create `simulation_ticks` table — stores every live tick received for a symbol while a simulated position is OPEN, enabling post-hoc replay and analysis. Columns: `id INTEGER PK AUTOINCREMENT, trade_id TEXT, symbol TEXT, timestamp TEXT, last_price REAL, volume INTEGER, oi INTEGER, created_at TEXT`. Create index: `CREATE INDEX idx_sim_ticks_trade ON simulation_ticks(trade_id)`. | | |
| TASK-011 | In `scripts/init_db.py`, create `strategy_memory` table: `date TEXT PK, notes TEXT, threshold_adjustments TEXT, most_predictive_indicators TEXT, quant_model_accuracy TEXT, win_rate_7d REAL, win_rate_30d REAL, avg_ev_at_win REAL, avg_ev_at_loss REAL, updated_at TEXT`. | | |
| TASK-012 | In `scripts/init_db.py`, create `iv_history` table: `id INTEGER PK AUTOINCREMENT, symbol TEXT, date TEXT, iv REAL, logged_at TEXT, UNIQUE(symbol, date)`. Create `config` table: `key TEXT PK, value TEXT, updated_at TEXT`. Create `system_logs` table: `id INTEGER PK AUTOINCREMENT, level TEXT, message TEXT, context TEXT, created_at TEXT`. | | |
| TASK-013 | In `scripts/init_db.py`, create `daily_summary` table — written by APScheduler at 15:30 IST each day: `date TEXT PK, total_simulations INTEGER, executed INTEGER, skipped INTEGER, blocked INTEGER, wins INTEGER, losses INTEGER, forces INTEGER, win_rate REAL, total_pnl REAL, avg_pnl REAL, best_trade_id TEXT, worst_trade_id TEXT, created_at TEXT`. | | |
| TASK-014 | Create `src/utils/db_client.py` with module-level singleton `aiosqlite` connection. Functions: `async def get_db() -> aiosqlite.Connection`, `async def db_execute(sql, params=())`, `async def db_fetchall(sql, params=()) -> list[dict]`, `async def db_fetchone(sql, params=()) -> dict | None`, `async def db_executemany(sql, params_list)`. Connection opened with `row_factory = aiosqlite.Row`. WAL mode set on first connection. | | |
| TASK-015 | Create `src/utils/tick_queue.py` with module-level `tick_queue: asyncio.Queue = asyncio.Queue(maxsize=500)` and `sim_tick_queues: dict[str, asyncio.Queue] = {}` — a per-trade-id queue that the Simulation Monitor subscribes to for live price updates. Functions: `async def enqueue_tick(tick: dict)` — puts on `tick_queue`, also fans out to any active `sim_tick_queues`; `def register_sim_queue(trade_id: str) -> asyncio.Queue` — creates and registers a per-trade queue; `def deregister_sim_queue(trade_id: str)` — removes the queue after trade closes. | | |
| TASK-016 | Create `scripts/start.py` — unified launcher: (1) `load_dotenv()`, (2) validate all `.env` keys, (3) `python scripts/init_db.py` (idempotent), (4) start APScheduler with jobs from `scripts/scheduler.py`, (5) start `TickerAgent` in `threading.Thread(daemon=True)`, (6) `uvicorn.run("src.main:app", host="127.0.0.1", port=8080, workers=1, limit_concurrency=4)`. | | |
| TASK-017 | Create `scripts/daily_auth.py` — CLI script: (1) reads `api_key` and `api_secret` from `.env`, (2) prints Kite login URL, (3) accepts `--request-token` arg, (4) calls `kite.generate_session()`, (5) writes `access_token` to `.env` via `dotenv.set_key(".env", "KITE_ACCESS_TOKEN", token)`, (6) prints "✅ Token saved. Valid until midnight IST." | | |
| TASK-018 | Create `scripts/scheduler.py` with `AsyncIOScheduler`. Jobs: (a) `instrument_refresh` — cron `hour=8, minute=0, day_of_week="mon-fri"` IST; (b) `market_open_notify` — cron `hour=9, minute=0` IST, sends desktop notification via `plyer` reminding user to run `daily_auth.py`; (c) `force_close_all` — cron `hour=15, minute=15` IST, calls `force_close_all_simulations()`; (d) `early_close_guard` — cron `hour=15, minute=10` IST, same call as safety net; (e) `daily_cleanup` — cron `hour=15, minute=30` IST, runs cleanup SQL per CON-006; (f) `write_daily_summary` — cron `hour=15, minute=35` IST, writes `daily_summary` row. Add `plyer>=2.1.0` to `requirements.txt`. | | |

### Implementation Phase 1 — Kite Read-Only Integration Layer

- GOAL-002: Implement the complete Kite Connect v3 **read-only** integration: authentication, instrument token management, WebSocket tick streaming to `tick_queue`, and OHLCV historical data fetching. No order API calls are made anywhere in this phase or any subsequent phase.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-019 | Create `src/utils/kite_client.py` with `get_kite() -> KiteConnect`. Reads `api_key` and `access_token` from `.env` via `get_config()`. Raises `ValueError` if `access_token == "PENDING"`. Returns configured `KiteConnect` instance. This client is used **exclusively for read operations**: `kite.quote()`, `kite.instruments()`, `kite.historical_data()`. | | |
| TASK-020 | Create `src/utils/rate_limiter.py` with `TokenBucketRateLimiter(capacity=3, refill_rate=3.0)`. Module-level singleton `kite_limiter`. All Kite REST calls must call `await kite_limiter.acquire()` before execution. | | |
| TASK-021 | Create `src/utils/instrument_manager.py` with `async def refresh_instruments() -> dict[str, int]`: (1) `await kite_limiter.acquire()`, (2) `kite.instruments("NFO")`, (3) filter for weekly expiry CE/PE for NIFTY/BANKNIFTY within ±500/±1000 points of spot, (4) serialize to JSON and write to SQLite `config` table key `instruments`, (5) return `tradingsymbol -> instrument_token` dict. Fallback: if Kite unavailable, load `data/instruments_cache.json`. | | |
| TASK-022 | Create `src/agents/ticker_agent.py` with class `TickerAgent`. Reads instrument tokens from SQLite `config`. Instantiates `KiteTicker(api_key, access_token)`. In `on_connect`: subscribes with `ws.set_mode(ws.MODE_FULL, token_list)`. In `on_ticks`: calls `enqueue_tick(tick)` for each tick. In `on_reconnect`: re-subscribes. In `on_noreconnect`: writes CRITICAL to SQLite `system_logs` and calls `os._exit(1)`. **No order-related callbacks implemented.** | | |
| TASK-023 | Create `src/utils/ohlcv_fetcher.py` with `async def fetch_ohlcv(instrument_token: int, interval: str, lookback_bars: int = 220) -> list[dict]`. Calls `kite.historical_data()` wrapped in `kite_limiter.acquire()`. Hard cap at 220 bars per CON-010. Returns list of dicts: `date, open, high, low, close, volume`. | | |
| TASK-024 | Create `src/utils/quote_fetcher.py` with `async def fetch_quote(symbols: list[str]) -> dict`. Calls `kite.quote(symbols)` wrapped in `kite_limiter.acquire()`. Used to fetch spot price, India VIX, and option LTP. This is the **only** Kite API call that touches live prices outside the WebSocket. | | |

### Implementation Phase 2 — Signal Scoring Engine

- GOAL-003: Implement all 8 indicator scoring functions as pure, stateless, memory-efficient Python functions. Each function returns both the signal direction and the raw indicator value for storage in `signal_audit`.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-025 | Create `src/models/signal_models.py` with Pydantic models: `OHLCVBar(date: datetime, open: float, high: float, low: float, close: float, volume: int)`, `IndicatorSignal(name: str, value: float, signal: Literal["bull","bear","neutral"])`, `ScoreResult(bull_score: int, bear_score: int, strength_pct: float, signals: dict[str, IndicatorSignal], ema_zone: Literal["UP","DOWN","NEUTRAL"])`. | | |
| TASK-026 | Create `src/utils/bar_converter.py` with `bars_to_numpy(bars: list[OHLCVBar]) -> dict[str, np.ndarray]`. Returns `open`, `high`, `low`, `close`, `volume` as `np.float32` arrays per GUD-007. Single conversion point for all indicator functions. | | |
| TASK-027 | Implement `score_rsi(arrays) -> IndicatorSignal` in `src/agents/signal_agent.py`. Import `pandas_ta` inside function body. RSI-14. `bull` if `rsi < 40`, `bear` if `rsi > 60`. Store raw RSI value in `IndicatorSignal.value`. `del` intermediates, `gc.collect()` after. | | |
| TASK-028 | Implement `score_macd(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. MACD(12,26,9). `bull` if `macd > signal`, `bear` if below. Store `macd - signal` as value. | | |
| TASK-029 | Implement `score_stochastic(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. Stochastic %K(14,3). `bull` if `%K < 20`, `bear` if `%K > 80`. | | |
| TASK-030 | Implement `score_momentum(arrays) -> IndicatorSignal`. Pure numpy: `momentum = float(arrays["close"][-1] - arrays["close"][-11])`. `bull` if positive, `bear` if negative. | | |
| TASK-031 | Implement `score_ema200(arrays) -> IndicatorSignal` and `compute_ema_zone(arrays, zone_pct=0.002) -> Literal["UP","DOWN","NEUTRAL"]`. Import `pandas_ta` inside function body. EMA-200. Zone: `UP` if `close > ema * 1.002`, `DOWN` if `close < ema * 0.998`. | | |
| TASK-032 | Implement `score_supertrend(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. SuperTrend(10, 3.0). `bull` if `close > supertrend`, `bear` if below. | | |
| TASK-033 | Implement `score_cci(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. CCI-20. `bull` if `cci < -100`, `bear` if `cci > 100`. | | |
| TASK-034 | Implement `score_obv_roc(arrays) -> IndicatorSignal`. Import `pandas_ta` inside function body. OBV then 5-bar ROC. `bull` if positive, `bear` if negative. | | |
| TASK-035 | Implement `async def compute_scores(bars: list[OHLCVBar]) -> ScoreResult`. Call `bars_to_numpy()` once. Run all 8 scoring functions concurrently via `asyncio.gather()` + `run_in_executor`. Aggregate `bull_score`, `bear_score`, `strength_pct`. Compute `ema_zone`. `del arrays`, `gc.collect()`. Return `ScoreResult`. | | |

### Implementation Phase 3 — Gemini Decision & Risk Guard Agents

- GOAL-004: Implement the Gemini Flash Decision Agent (via `google-generativeai` REST SDK) and the hard-rule Risk Guard Agent. The Risk Guard reads daily simulated P&L from SQLite — not from Kite positions.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-036 | Create `src/utils/gemini_client.py` with `async def call_gemini(prompt: str, model: str = "gemini-2.5-flash") -> str`. Calls `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={GEMINI_API_KEY}` via `httpx.AsyncClient`. Set `timeout=10.0`. On HTTP error or timeout, log to SQLite `system_logs` and raise. | | |
| TASK-037 | Create `src/prompts/prompts.py` with string constants: `DECISION_AGENT_PROMPT_V1` (base), `DECISION_AGENT_PROMPT_V2` (with Quantitative Risk Context section injecting all quant metrics), `REFLECTION_AGENT_PROMPT_V1`. All use `str.format_map()` compatible named placeholders. Prompts instruct Gemini to return **valid JSON only** with no markdown fencing. | | |
| TASK-038 | Create `src/models/decision_models.py` with `DecisionInput(symbol, spot_price, option_ltp, score_result: ScoreResult, success_rate, regime, dte, iv, pcr, quant_metrics: QuantMetrics)` and `DecisionOutput(action: Literal["EXECUTE","SKIP","WAIT"], confidence: Literal["FULL_CONFIDENCE","LOW_CONFIDENCE"], reasoning: str, entry_price: float, tp_price: float, sl_price: float)`. `QuantMetrics` defined in TASK-052. | | |
| TASK-039 | Create `src/agents/decision_agent.py` with `async def run_decision_agent(input: DecisionInput) -> DecisionOutput`. Formats `DECISION_AGENT_PROMPT_V2`. Calls `call_gemini()`. Parses JSON via `pydantic.model_validate_json()`. On parse failure: retry once with `DECISION_AGENT_PROMPT_V1` (simpler prompt). On second failure: return `DecisionOutput(action="SKIP", confidence="LOW_CONFIDENCE", reasoning="Gemini parse failure")`. | | |
| TASK-040 | Create `src/agents/risk_guard_agent.py` with `async def check_risk(current_time: datetime) -> tuple[bool, str]`. Hard-block rules: (a) query SQLite `SELECT COALESCE(SUM(pnl),0) FROM trades WHERE date(entry_time)=date('now')` for `daily_pnl`; if `daily_pnl <= -(capital * 0.02)` → block `"Daily loss limit 2% breached"`; (b) `current_time >= 15:15 IST` → block `"Past MIS cutoff"`; (c) `current_time < 09:15 IST` → block `"Before market open"`; (d) fetch India VIX via `fetch_quote(["NSE:INDIA VIX"])`, if `vix > 25.0` → block `"India VIX > 25"`. Return `(True, "APPROVED")` only if all pass. **No Kite order API calls.** | | |
| TASK-041 | Create `src/utils/market_utils.py` with: (a) `async def compute_pcr(kite, nifty_spot) -> float` — fetches OI data via `kite.quote()` for ATM ±5 strikes, returns `put_oi / call_oi`; (b) `async def detect_regime(bars) -> tuple[str, float]` — returns `(regime_string, atr_14)`; (c) `async def get_success_rate() -> float` — queries SQLite last 20 closed trades, returns win rate; (d) `async def get_avg_rr() -> float` — queries SQLite last 20 closed trades, returns `avg(pnl_win) / abs(avg(pnl_loss))`. | | |

### Implementation Phase 4 — Quantitative Risk & Alpha Layer

- GOAL-005: Implement the institutional-grade Quant Risk Agent using pure NumPy GARCH, SQLite-backed Kelly, ATR-14, EV gating, and IV percentile filtering. All computations are local — no external libraries beyond numpy.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-042 | Create `src/quant/garch_lite.py` with `fit_garch_lite(returns: np.ndarray) -> tuple[float, float, float]`. Pure NumPy GARCH(1,1): (1) `var_0 = np.var(returns)`; (2) params `omega=1e-6, alpha=0.1, beta=0.85`; (3) 50-iteration coordinate descent: `h[t] = omega + alpha * r[t-1]**2 + beta * h[t-1]`; (4) log-likelihood `LL = -0.5 * sum(log(h) + r**2/h)`; (5) return `(sqrt(omega + alpha*r[-1]**2 + beta*h[-1]), omega, alpha, beta)`. Uses `float64` per GUD-007 exception. Memory: ~1.6 KB. | | |
| TASK-043 | Create `src/models/quant_models.py` with `QuantMetrics(atr_14: float, atr_sl_price: float, atr_tp_price: float, atr_sl_exceeds_limit: bool, kelly_fraction: float, kelly_quantity: int, kelly_fallback: bool, garch_vol_forecast: float, garch_vol_historical_mean: float, garch_vol_ratio: float, garch_size_reduction: bool, garch_engine: Literal["garch_lite","ewma_fallback"], ev: float, ev_positive: bool, iv_percentile: float, iv_blocked: bool, win_rate: float, avg_rr: float, trade_count_used: int)`. | | |
| TASK-044 | Create `src/quant/atr_calculator.py` with `compute_atr_sl_tp(arrays, entry_price, direction, multiplier_sl=1.5, multiplier_tp=3.0) -> tuple[float, float, float]`. Pure NumPy ATR-14: `TR = np.maximum(H-L, np.abs(H-prev_C), np.abs(L-prev_C))`, ATR via EWM with `span=14`. For BUY: `sl = entry - (1.5 × atr)`, `tp = entry + (3.0 × atr)`. For SELL: inverse. Validate `abs(entry - sl) / entry <= 0.02`. | | |
| TASK-045 | Create `src/quant/kelly_calculator.py` with `async def compute_kelly(capital: float, max_lots: int = 5) -> tuple[float, int, bool]`. Query SQLite: `SELECT outcome, pnl FROM trades WHERE status != 'OPEN' ORDER BY entry_time DESC LIMIT 20`. If `count < 10` → return `(0.0, 1, True)`. Compute `W = wins/count`, `R = avg_win_pnl / abs(avg_loss_pnl)`, `f* = (W*R - (1-W)) / R`, `f_half = 0.5 * f*`. Clamp to `[0.01, 0.20]`. `kelly_quantity = max(1, min(max_lots, floor(f_half * capital / option_lot_value)))`. | | |
| TASK-046 | Create `src/quant/garch_forecaster.py` with `compute_garch_forecast(arrays) -> tuple[float, float, float, str]`. Compute log returns from `arrays["close"]` (float64). Call `fit_garch_lite()` in `ThreadPoolExecutor` with 0.2s timeout per CON-007. On timeout: `sigma = np.std(returns[-20:]) * np.sqrt(375)`, `engine = "ewma_fallback"`. Compute `sigma_historical_mean` from SQLite `signal_audit` last 30 days `garch_vol_forecast` values. Return `(sigma_forecast, sigma_historical_mean, vol_ratio, engine)`. | | |
| TASK-047 | Create `src/quant/ev_calculator.py` with `compute_ev(win_rate: float, tp_distance: float, sl_distance: float) -> float`. `EV = (win_rate * tp_distance) - ((1 - win_rate) * sl_distance)`. Pure Python. | | |
| TASK-048 | Create `src/quant/iv_filter.py` with: (a) `async def compute_iv_percentile(symbol: str, current_iv: float) -> float` — queries SQLite `SELECT iv FROM iv_history WHERE symbol=? ORDER BY date DESC LIMIT 30`, computes `np.searchsorted(np.sort(iv_array), current_iv) / len(iv_array) * 100`; (b) `async def log_iv(symbol: str, iv: float)` — `INSERT OR REPLACE INTO iv_history`. | | |
| TASK-049 | Create `src/agents/quant_risk_agent.py` with `async def run_quant_risk_agent(bars, entry_price, direction, current_iv, symbol, capital) -> tuple[bool, str, QuantMetrics]`. Orchestrates: ATR → Kelly → GARCH (with size reduction if `vol_ratio > 2.0`) → EV → IV percentile. Block conditions: `atr_sl_exceeds_limit` → `"BLOCKED_ATR"`; `ev <= 0` → `"BLOCKED_EV"`; `iv_percentile > 80` → `"BLOCKED_IV"`. Return `(True, "APPROVED", quant_metrics)` if all pass. | | |
| TASK-050 | Create `src/quant/iv_history_seeder.py` — one-time script to backfill 30 days of IV history into SQLite `iv_history`. Fetches ATM option historical data from `kite.historical_data()` for each past trading day, computes realized IV as `np.std(log_returns) * np.sqrt(252 * 375)`, writes one row per day per symbol. Run once: `python src/quant/iv_history_seeder.py`. | | |

### Implementation Phase 5 — Simulation Execution & Monitor Agents

- GOAL-006: Implement the Simulation Execution Agent (creates simulated trade records in SQLite with synthetic order IDs) and the Simulation Monitor Agent (tracks live tick prices against simulated TP/SL levels, writes outcome to SQLite, stores every tick during the open position for post-hoc analysis).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-051 | Create `src/models/trade_models.py` with `TradeRecord` Pydantic model matching all columns of the SQLite `trades` table defined in TASK-008. Include `quant_metrics: QuantMetrics | None` as a transient field — flattened to `qm_*` columns on SQLite write. | | |
| TASK-052 | Create `src/agents/simulation_execution_agent.py` with `async def execute_simulation(decision: DecisionOutput, symbol: str, quantity: int, direction: Literal["BUY","SELL"], score_result: ScoreResult, regime: str, quant_metrics: QuantMetrics, spot_price: float, option_ltp: float, pcr: float, dte: int, iv: float) -> TradeRecord`. Actions: (1) generate `trade_id = f"SIM_{symbol}_{datetime.now(utc).strftime('%Y%m%d%H%M%S%f')}"`, (2) generate synthetic `entry_order_id = f"SIM_ENTRY_{uuid4().hex[:8]}"` and `sl_order_id = f"SIM_SL_{uuid4().hex[:8]}"`, (3) set `entry_price = option_ltp` (current live LTP from tick), (4) construct `TradeRecord` with `status = "OPEN"`, (5) INSERT all fields including all `qm_*` columns into SQLite `trades`, (6) call `register_sim_queue(trade_id)` from `tick_queue.py`, (7) return `TradeRecord`. **No Kite order API calls.** | | |
| TASK-053 | Create `src/agents/simulation_monitor_agent.py` with `async def monitor_simulation(trade: TradeRecord) -> TradeRecord`. Subscribes to `sim_tick_queues[trade.trade_id]`. On each tick received: (1) INSERT tick into SQLite `simulation_ticks` table per REQ-016; (2) check TP hit: `current_ltp >= trade.tp_price` (BUY) or `current_ltp <= trade.tp_price` (SELL) → set `outcome = "TP"`, `exit_price = current_ltp`; (3) check SL hit: `current_ltp <= trade.sl_price` (BUY) or `current_ltp >= trade.sl_price` (SELL) → set `outcome = "SL"`, `exit_price = current_ltp`; (4) trailing SL: if BUY and `current_ltp >= entry * 1.01` → move SL to `entry` (breakeven); if `current_ltp >= entry * 1.015` → move SL to `entry * 1.005`; (5) force-close at 15:15 IST → `outcome = "FORCE"`, `exit_price = current_ltp`; (6) on close: compute `pnl = (exit_price - entry_price) * quantity * lot_size * direction_multiplier`, `pnl_pct = pnl / (entry_price * quantity * lot_size) * 100`, UPDATE SQLite `trades` row, call `deregister_sim_queue(trade_id)`, return closed `TradeRecord`. | | |
| TASK-054 | Create `src/utils/force_close.py` with `async def force_close_all_simulations() -> None`. Queries SQLite `SELECT * FROM trades WHERE status = "OPEN"`. For each open trade: fetches current LTP via `fetch_quote([symbol])`, sets `exit_price = ltp`, `outcome = "FORCE"`, `status = "CLOSED_FORCE"`, computes `pnl`, UPDATEs SQLite. Calls `deregister_sim_queue(trade_id)`. Logs count of force-closed trades to `system_logs`. | | |

### Implementation Phase 6 — Reflection Agent & Audit Writer

- GOAL-007: Implement the Reflection Agent (Gemini Pro post-trade analysis → SQLite `strategy_memory`) and the Audit Writer (writes complete ML-ready row to `signal_audit` for every pipeline evaluation regardless of outcome).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-055 | Create `src/agents/audit_writer.py` with `async def write_signal_audit(audit_id: str, evaluated_at: str, symbol: str, direction: str, pipeline_outcome: str, block_reason: str | None, score_result: ScoreResult, market_context: dict, quant_metrics: QuantMetrics | None, decision: DecisionOutput | None, trade_id: str | None) -> None`. Constructs a flat dict from all inputs and INSERTs into SQLite `signal_audit` table per GUD-009. Called at the end of every pipeline cycle regardless of outcome (EXECUTE, SKIP, WAIT, BLOCKED_ATR, BLOCKED_EV, BLOCKED_IV, BLOCKED_RISK, BLOCKED_REGIME). | | |
| TASK-056 | Add `REFLECTION_AGENT_PROMPT_V1` to `src/prompts/prompts.py`. Placeholders: `{trade_json}`, `{entry_reasoning}`, `{outcome}`, `{regime}`, `{bull_score}`, `{ema_zone}`, `{kelly_fraction}`, `{garch_vol}`, `{ev}`, `{iv_percentile}`, `{win_rate_7d}`, `{win_rate_30d}`. Instructs Gemini Pro to return JSON: `entry_reasoning_correct: bool`, `most_predictive_indicators: list[str]`, `threshold_adjustment: dict[str, float]`, `memory_note: str`, `quant_model_accuracy: str`. | | |
| TASK-057 | Create `src/agents/reflection_agent.py` with `async def run_reflection(trade: TradeRecord) -> None`. (1) Compute `win_rate_7d` and `win_rate_30d` from SQLite. (2) Format `REFLECTION_AGENT_PROMPT_V1`. (3) Call `call_gemini(prompt, model="gemini-2.5-pro")`. (4) Parse JSON response. (5) Query SQLite `SELECT * FROM strategy_memory WHERE date = ?` for today. (6) Merge `threshold_adjustment` into existing JSON, append `memory_note`. (7) `INSERT OR REPLACE INTO strategy_memory`. | | |

### Implementation Phase 7 — Orchestrator & FastAPI Server

- GOAL-008: Wire all agents into the sequential pipeline consuming from `tick_queue`, implement all FastAPI endpoints including the analytics summary endpoint, and configure APScheduler for daily lifecycle management.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-058 | Create `src/orchestrator.py` with `async def run_pipeline(tick_data: dict) -> dict` and `async def consume_tick_queue() -> None` (infinite loop: `tick_data = await tick_queue.get()` → `run_pipeline(tick_data)`). Pipeline per PAT-001: (1) `fetch_ohlcv()`, (2) `compute_scores()`, (3) regime check per GUD-006 → if blocked, `write_signal_audit(outcome="BLOCKED_REGIME")` and return, (4) `detect_regime()`, `compute_pcr()`, `fetch_quote()` for spot/LTP/IV, (5) `run_quant_risk_agent()` → if blocked, `write_signal_audit(outcome=block_reason)` and return, (6) `run_decision_agent()` → if SKIP/WAIT, `write_signal_audit(outcome=action)` and return, (7) `check_risk()` → if blocked, `write_signal_audit(outcome="BLOCKED_RISK")` and return, (8) `execute_simulation()`, (9) `write_signal_audit(outcome="EXECUTE", trade_id=trade.trade_id)`, (10) `asyncio.create_task(monitor_simulation(trade))` with `run_reflection()` registered as callback on close, (11) `gc.collect()`. | | |
| TASK-059 | Create `src/main.py` with FastAPI app bound to `127.0.0.1:8080`. Lifespan context manager starts `consume_tick_queue()` as `asyncio.create_task()`. Endpoints: `GET /health` (RAM, DB size, open simulations count, uptime); `GET /trades/today` (all trades for today from SQLite); `GET /trades/{trade_id}` (single trade + its simulation ticks); `GET /strategy/memory` (latest strategy_memory row); `GET /quant/metrics/latest` (latest qm_* columns from trades); `GET /system/logs` (last 50 system_logs rows); `GET /analytics/summary` (see TASK-060); `GET /analytics/patterns` (see TASK-061). | | |
| TASK-060 | Create `src/analytics/summary.py` with `async def get_summary() -> dict` for `GET /analytics/summary` per REQ-019. Queries: (a) total simulations, wins, losses, forces, blocks; (b) overall win rate; (c) avg P&L per trade; (d) avg EV at entry for wins vs losses; (e) win rate by regime (`GROUP BY regime`); (f) win rate by `bull_score` (`GROUP BY bull_score`); (g) win rate by hour-of-day (`GROUP BY strftime('%H', entry_time)`); (h) indicator correlation with outcome — for each of 8 indicators, compute `count(outcome='TP' AND {indicator}_signal='bull') / count({indicator}_signal='bull')` from `signal_audit`. | | |
| TASK-061 | Create `src/analytics/patterns.py` with `async def get_patterns() -> dict` for `GET /analytics/patterns`. Queries `signal_audit` to identify: (a) top 3 indicator combinations (by bull_score composition) with highest win rate; (b) best-performing time windows (30-min buckets); (c) regime × ema_zone × bull_score combinations with win rate > 60% and count > 5; (d) average GARCH vol at winning vs losing entries; (e) IV percentile distribution at wins vs losses. Returns structured JSON suitable for direct consumption by a future ML feature engineering pipeline. | | |

### Implementation Phase 8 — Dashboard & Testing

- GOAL-009: Build the local static HTML dashboard, implement unit and integration tests, and create the architecture diagram.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-062 | Create `dashboard/index.html` — single static HTML file per CON-011. CDN Chart.js. Sections: (1) **Live Signal Panel** — 8 indicator badges (bull/bear/neutral), bull/bear scores, strength %, regime badge, EMA zone; (2) **Active Simulation** — current open trade (entry, TP, SL, current LTP, unrealized P&L, trailing SL level); (3) **Trade Log** — last 10 closed simulations with outcome, P&L, entry/exit, Gemini reasoning; (4) **Quant Metrics** — Kelly %, GARCH σ, EV, IV%ile, ATR-14, GARCH engine badge; (5) **Analytics** — win rate chart by hour, win rate by bull_score bar chart, regime win rate table; (6) **Daily Summary** — total sims, win rate, total P&L. Polls `/trades/today`, `/analytics/summary`, `/quant/metrics/latest` every 5 seconds. Open via `python3 -m http.server 3000` in `dashboard/` directory. | | |
| TASK-063 | Create `tests/unit/test_signal_agent.py` with parametrized tests for all 8 scoring functions using synthetic OHLCV bars with known indicator values. Assert correct `bull`/`bear`/`neutral` output and correct `value` field. | | |
| TASK-064 | Create `tests/unit/test_quant.py` with tests for: `compute_atr_sl_tp()` (known ATR → assert SL/TP distances), `compute_kelly()` (mock SQLite with known win/loss records → assert fraction), `fit_garch_lite()` (known returns → assert sigma > 0), `compute_ev()` (known W/R → assert EV sign). | | |
| TASK-065 | Create `tests/integration/test_pipeline.py` with full pipeline integration test: mock `fetch_ohlcv()` to return 220 synthetic bars with `bull_score=7`, mock `call_gemini()` to return `{"action":"EXECUTE",...}`, mock `check_risk()` to return `(True,"APPROVED")`. Assert `signal_audit` row written, `trades` row written with `status="OPEN"`, `trade_id` starts with `"SIM_"`. | | |
| TASK-066 | Create `tests/integration/test_simulation_monitor.py`: create an open `TradeRecord` in SQLite, push synthetic ticks to `sim_tick_queues[trade_id]` that cross the TP level, assert `trades` row updated to `status="CLOSED_TP"`, `outcome="TP"`, `pnl > 0`, and `simulation_ticks` rows written. | | |
| TASK-067 | Create `scripts/generate_diagram.py` using `diagrams` library. Renders the local-only pipeline: `KiteTicker (READ ONLY) → asyncio.Queue → Orchestrator → Signal Agent → Decision Agent (Gemini Flash) → Risk Guard → Quant Risk → Simulation Execution → Simulation Monitor → Reflection Agent (Gemini Pro) → SQLite → Dashboard`. Save as `plan/architecture.png`. Add `diagrams>=0.23.4` to `requirements.txt` as dev dependency. | | |

---

## 3. Alternatives

- **ALT-001**: Use Alpaca or Interactive Brokers API instead of Kite Connect v3 — rejected because the project is specifically designed for NIFTY/BANKNIFTY on NSE/NFO. Kite Connect is the most reliable, lowest-latency broker API for Indian markets with native WebSocket tick streaming.
- **ALT-002**: Use a rule-based engine instead of Gemini for the Decision Agent — rejected because the Gemini reasoning output is itself a valuable feature for the ML training dataset. The `gemini_reasoning` text stored in `signal_audit` can be embedded and used as a feature in future model training.
- **ALT-003**: Use PostgreSQL instead of SQLite — rejected because PostgreSQL requires a separate server process (~50–100 MB RAM). SQLite is serverless, zero-configuration, and fully sufficient for single-process intraday simulation with < 500 trades/day.
- **ALT-004**: Use LangChain instead of direct `google-generativeai` SDK — rejected because LangChain adds ~150 MB of dependencies and abstraction overhead. Direct `httpx` calls to the Gemini REST API achieve the same result with ~15 MB.
- **ALT-005**: Use `ta-lib` instead of `pandas-ta` — rejected because `ta-lib` requires a C library system dependency. `pandas-ta` is pure Python.
- **ALT-006**: Store simulation ticks in a separate CSV file per trade instead of SQLite `simulation_ticks` table — rejected because SQL queries over the tick table enable powerful post-hoc analysis (e.g., "how many ticks before TP was hit", "max adverse excursion before recovery") that would require custom parsing with CSV files.
- **ALT-007**: Use fixed fractional position sizing instead of Kelly Criterion — rejected because fixed fractional ignores empirical win rate and payoff ratio. Kelly Criterion produces the optimal simulated position size for geometric growth analysis, which is critical for the future ML model's reward signal.
- **ALT-008**: Use EWMA volatility instead of GARCH(1,1) — retained only as the CON-007 timeout fallback. GARCH(1,1) provides superior one-step-ahead forecasts for NIFTY intraday volatility clustering. The `garch_engine` field in `signal_audit` allows post-hoc comparison of GARCH vs EWMA prediction accuracy.
- **ALT-009**: Use a separate process for the Simulation Monitor instead of `asyncio.create_task()` — rejected because a separate process would require IPC (pipes or sockets) to receive tick data, adding complexity. `asyncio.create_task()` within the same event loop is sufficient and memory-efficient for monitoring up to 5 concurrent simulated positions.
- **ALT-010**: Store `signal_audit` as JSON blobs instead of flat columns — rejected because flat columns enable direct `SELECT`, `GROUP BY`, and `WHERE` queries for analytics per REQ-019 without JSON parsing overhead. The ML training pipeline can consume the table with a single `SELECT *` query.

---

## 4. Dependencies

- **DEP-001**: `kiteconnect>=5.0.1` — Zerodha's official Python client. Used **read-only**: `kite.instruments()`, `kite.historical_data()`, `kite.quote()`, `KiteTicker` WebSocket. Required by TASK-019 through TASK-024.
- **DEP-002**: `pandas-ta>=0.3.14b` — Technical analysis library for RSI, MACD, Stochastic, EMA, SuperTrend, CCI, OBV. Imported inside function bodies per CON-010. Required by TASK-027 through TASK-034.
- **DEP-003**: `google-generativeai>=0.7.0` — Lightweight Gemini REST API client (~15 MB). Used for Decision Agent (Flash) and Reflection Agent (Pro). Required by TASK-036, TASK-039, TASK-057.
- **DEP-004**: `aiosqlite>=0.20.0` — Async SQLite driver. Zero external dependencies. The sole persistence layer. Required by TASK-014 and all agents that read/write state.
- **DEP-005**: `python-dotenv>=1.0.0` — `.env` file loader and writer. Required by TASK-006, TASK-017.
- **DEP-006**: `apscheduler>=3.10.4` — In-process async job scheduler for
