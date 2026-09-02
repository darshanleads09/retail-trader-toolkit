Based on the context of our conversation, I'll generate the implementation plan for the Kite Connect Options Trading Bot we built. The ${input:PlanPurpose} resolves to: "Kite Connect .NET Core Options Trading Bot with TA-Lib indicators".

markdown---
goal: Kite Connect .NET Core Options Trading Bot with TA-Lib Technical Indicators
version: 1.0
date_created: 2026-09-02
last_updated: 2026-09-02
owner: Quant/Algo Trading Team
status: 'Planned'
tags: [feature, architecture, trading, dotnet, kite-connect, ta-lib, options]
---

# Introduction

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

This plan describes the end-to-end implementation of an automated options trading
bot built on .NET 8, the Kite Connect v3 REST + WebSocket API, and TA-Lib.NETCore.
The bot monitors a configurable underlying (default: NIFTY 50) on a 15-minute bar
cadence, evaluates entry signals using MA crossovers, SuperTrend, RSI, and Darvas
Box breakouts, manages open positions with a direction-aware trailing stop-loss
(SuperTrend + PSAR), and exits via SL hit, PSAR flip, SuperTrend flip, or Pivot
target. All order execution is routed through the Kite Connect v3 API.

---

## 1. Requirements & Constraints

- **REQ-001**: The system MUST authenticate with Kite Connect v3 using API Key +
  Access Token. A fresh access token must be obtained each trading day via the
  OAuth flow (`/connect/login` → request token → `POST /session/token`).
- **REQ-002**: The system MUST subscribe to live tick data via the Kite WebSocket
  (`wss://ws.kite.trade`) and aggregate ticks into OHLCV bars matching the
  configured interval (default: `15minute`).
- **REQ-003**: The system MUST calculate the following indicators on every bar
  close: SMA(7), SMA(21), SMA(44), RSI(14), SuperTrend(10, 3.0), PSAR(0.02, 0.2).
- **REQ-004**: The system MUST implement the Darvas Box detection algorithm with a
  configurable lookback window (default: 20 bars).
- **REQ-005**: The system MUST resolve the ATM option instrument token dynamically
  from the Kite instruments dump at session start, using nearest weekly expiry.
- **REQ-006**: The system MUST place MIS (intraday) market orders for CE/PE options
  via `POST /orders/regular`.
- **REQ-007**: The system MUST implement a direction-aware trailing SL: for CE
  positions the SL only moves UP; for PE positions the SL only moves DOWN.
- **REQ-008**: The system MUST log every trade entry and exit to a CSV file with
  columns: Timestamp, PositionType, Symbol, EntryPrice, InitialSL, TrailingSL,
  ExitReason.
- **REQ-009**: The system MUST support graceful shutdown on SIGINT (Ctrl+C),
  closing the WebSocket ticker before process exit.
- **REQ-010**: The system MUST warm up all indicators using `LookbackBars`
  (default: 200) of historical OHLCV data fetched via `GET /instruments/historical`
  before subscribing to live ticks.

- **SEC-001**: API Key, API Secret, and Access Token MUST be stored in
  `appsettings.json` and MUST NOT be hard-coded in source files.
- **SEC-002**: Access Token MUST be treated as a secret; the `appsettings.json`
  file MUST be listed in `.gitignore`.
- **SEC-003**: All Kite API calls MUST be made over HTTPS (TLS 1.2+).

- **CON-001**: The bot is constrained to a single open position at a time
  (no pyramiding). A new entry signal is ignored while `position.type != NONE`.
- **CON-002**: TA-Lib.NETCore requires the native TA-Lib C shared library
  (`libta_lib.so` on Linux, `ta_lib.dll` on Windows) to be present at runtime.
- **CON-003**: Kite Connect WebSocket supports a maximum of 3000 instrument
  subscriptions per connection; this bot uses 1–2 tokens.
- **CON-004**: Kite Connect rate limits REST API calls to 10 requests/second.
  Historical data fetches during warm-up must respect this limit.
- **CON-005**: The bot targets .NET 8 LTS only. No .NET Framework or .NET 6/7
  compatibility is required.

- **GUD-001**: All services must be registered in the DI container
  (`Microsoft.Extensions.DependencyInjection`) as singletons.
- **GUD-002**: All log output must use `Microsoft.Extensions.Logging.ILogger<T>`;
  no `Console.WriteLine` in service classes.
- **GUD-003**: Indicator arrays must be aligned to input bar length with `NaN`
  padding for leading values where TA-Lib produces no output.

- **PAT-001**: Follow the Repository/Service pattern: `KiteService` owns all
  Kite API I/O; `TradingEngine` owns all strategy logic; no cross-cutting concerns.
- **PAT-002**: The `on_bar_close` main loop pattern from the pseudocode must be
  preserved exactly: entry is only checked when `position.type == NONE`; trailing
  SL update always precedes exit check.
- **PAT-003**: SuperTrend must be computed manually (ATR-based) since TA-Lib does
  not provide it natively. ATR values must be sourced from `Core.Atr()`.

---

## 2. Implementation Steps

### Implementation Phase 1 — Project Scaffold & Configuration

- GOAL-001: Create the .NET 8 solution structure, NuGet references, and
  configuration files so that all subsequent phases have a compilable baseline.

| Task     | Description                                                                                                                                                                                                                          | Completed | Date |
|----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|------|
| TASK-001 | Create solution file: `dotnet new sln -n TradingBot`. Create console project: `dotnet new console -n TradingBot -f net8.0`. Add project to solution: `dotnet sln add TradingBot/TradingBot.csproj`.                                  |           |      |
| TASK-002 | Add NuGet packages to `TradingBot.csproj`: `KiteConnect@4.0.1`, `TALib.NETCore@1.0.0`, `Microsoft.Extensions.Configuration.Json@8.0.0`, `Microsoft.Extensions.DependencyInjection@8.0.0`, `Microsoft.Extensions.Logging.Console@8.0.0`, `Newtonsoft.Json@13.0.3`. |           |      |
| TASK-003 | Create `appsettings.json` at `TradingBot/appsettings.json` with sections `Kite` (ApiKey, ApiSecret, AccessToken, RequestToken) and `Strategy` (Underlying, TradingSymbol, Exchange, SpotExchange, Interval, LookbackBars, SupertrendPeriod, SupertrendMultiplier, PsarAcceleration, PsarMaximum, DarvasLookback, Quantity, LogPath). Set `CopyToOutputDirectory` to `Always` in `.csproj`. |           |      |
| TASK-004 | Create `TradingBot/Config/AppConfig.cs` defining `KiteConfig` and `StrategyConfig` POCO classes with properties matching every key in `appsettings.json`. All properties must have non-null default values.                          |           |      |
| TASK-005 | Create `.gitignore` at solution root. Add entries: `appsettings.json`, `*.user`, `bin/`, `obj/`, `trades.csv`.                                                                                                                       |           |      |

### Implementation Phase 2 — Domain Models

- GOAL-002: Define all domain model classes used across services so that Phase 3
  and Phase 4 can reference them without circular dependencies.

| Task     | Description                                                                                                                                                                                                                                                                  | Completed | Date |
|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|------|
| TASK-006 | Create `TradingBot/Models/Bar.cs`. Fields: `int Index`, `DateTime Timestamp`, `double Open`, `double High`, `double Low`, `double Close`, `long Volume`. No methods.                                                                                                         |           |      |
| TASK-007 | Create `TradingBot/Models/DarvasBox.cs`. Fields: `double High`, `double Low`, `int StartIndex`, `bool IsActive`.                                                                                                                                                             |           |      |
| TASK-008 | Create `TradingBot/Models/Position.cs`. Enum `PositionType { None, CE, PE }`. Class `Position` with fields: `PositionType PositionType`, `double EntryPrice`, `double InitialSl`, `double TrailingSl`, `int EntryBarIndex`, `DarvasBox? DarvasBoxUsed`, `string? OptionSymbol`, `uint InstrumentToken`. Method `Reset()` sets all fields to defaults. |           |      |
| TASK-009 | Create `TradingBot/Models/Indicators.cs`. Enum `SupertrendDirection { Bullish, Bearish }`. Class `SupertrendPoint { double Value; SupertrendDirection Direction }`. Class `Indicators` with array fields: `double[] Ma7`, `Ma21`, `Ma44`, `Rsi14`, `Psar`; `SupertrendPoint[] Supertrend`. All arrays default to `[]`. |           |      |

### Implementation Phase 3 — Indicator Service

- GOAL-003: Implement `IndicatorService` which wraps TA-Lib calls and the manual
  SuperTrend algorithm, producing a fully populated `Indicators` snapshot from a
  `IReadOnlyList<Bar>`.

| Task     | Description
