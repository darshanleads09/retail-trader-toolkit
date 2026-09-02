This is a substantial implementation. I'll deliver it as a complete multi-file .NET Core solution. Given the size, I'll write all files in sequence — let me know if you want me to pause and explain any section.

Project Structure
TradingBot/
├── TradingBot.csproj
├── appsettings.json
├── Program.cs
├── Config/AppConfig.cs
├── Models/Bar.cs
├── Models/Position.cs
├── Models/DarvasBox.cs
├── Models/Indicators.cs
├── Services/KiteService.cs
├── Services/IndicatorService.cs
├── Services/DarvasBoxService.cs
├── Services/PivotService.cs
└── Services/TradingEngine.cs


TradingBot.csproj
xml<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="KiteConnect"          Version="4.0.1" />
    <PackageReference Include="TALib.NETCore"         Version="1.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging.Console"    Version="8.0.0" />
    <PackageReference Include="Newtonsoft.Json"       Version="13.0.3" />
  </ItemGroup>
</Project>

appsettings.json
json{
  "Kite": {
    "ApiKey":      "YOUR_API_KEY",
    "ApiSecret":   "YOUR_API_SECRET",
    "AccessToken": "",
    "RequestToken": ""
  },
  "Strategy": {
    "Underlying":        "NIFTY 50",
    "TradingSymbol":     "NIFTY",
    "Exchange":          "NFO",
    "SpotExchange":      "NSE",
    "Interval":          "15minute",
    "LookbackBars":      200,
    "SupertrendPeriod":  10,
    "SupertrendMultiplier": 3.0,
    "PsarAcceleration":  0.02,
    "PsarMaximum":       0.2,
    "DarvasLookback":    20,
    "Quantity":          50,
    "LogPath":           "trades.csv"
  }
}

Config/AppConfig.cs
csharpnamespace TradingBot.Config;

public class KiteConfig
{
    public string ApiKey      { get; set; } = "";
    public string ApiSecret   { get; set; } = "";
    public string AccessToken { get; set; } = "";
    public string RequestToken { get; set; } = "";
}

public class StrategyConfig
{
    public string Underlying           { get; set; } = "NIFTY 50";
    public string TradingSymbol        { get; set; } = "NIFTY";
    public string Exchange             { get; set; } = "NFO";
    public string SpotExchange         { get; set; } = "NSE";
    public string Interval             { get; set; } = "15minute";
    public int    LookbackBars         { get; set; } = 200;
    public int    SupertrendPeriod     { get; set; } = 10;
    public double SupertrendMultiplier { get; set; } = 3.0;
    public double PsarAcceleration     { get; set; } = 0.02;
    public double PsarMaximum          { get; set; } = 0.2;
    public int    DarvasLookback       { get; set; } = 20;
    public int    Quantity             { get; set; } = 50;
    public string LogPath              { get; set; } = "trades.csv";
}

Models/Bar.cs
csharpnamespace TradingBot.Models;

public class Bar
{
    public int      Index     { get; set; }
    public DateTime Timestamp { get; set; }
    public double   Open      { get; set; }
    public double   High      { get; set; }
    public double   Low       { get; set; }
    public double   Close     { get; set; }
    public long     Volume    { get; set; }
}

Models/DarvasBox.cs
csharpnamespace TradingBot.Models;

public class DarvasBox
{
    public double High      { get; set; }
    public double Low       { get; set; }
    public int    StartIndex { get; set; }
    public bool   IsActive  { get; set; }
}

Models/Position.cs
csharpnamespace TradingBot.Models;

public enum PositionType { None, CE, PE }

public class Position
{
    public PositionType PositionType   { get; set; } = PositionType.None;
    public double       EntryPrice     { get; set; }
    public double       InitialSl      { get; set; }
    public double       TrailingSl     { get; set; }
    public int          EntryBarIndex  { get; set; }
    public DarvasBox?   DarvasBoxUsed  { get; set; }
    public string?      OptionSymbol   { get; set; }
    public uint         InstrumentToken { get; set; }

    public void Reset()
    {
        PositionType    = PositionType.None;
        EntryPrice      = 0;
        InitialSl       = 0;
        TrailingSl      = 0;
        EntryBarIndex   = 0;
        DarvasBoxUsed   = null;
        OptionSymbol    = null;
        InstrumentToken = 0;
    }
}

Models/Indicators.cs
csharpnamespace TradingBot.Models;

public enum SupertrendDirection { Bullish, Bearish }

public class SupertrendPoint
{
    public double              Value     { get; set; }
    public SupertrendDirection Direction { get; set; }
}

public class Indicators
{
    public double[]           Ma7        { get; set; } = [];
    public double[]           Ma21       { get; set; } = [];
    public double[]           Ma44       { get; set; } = [];
    public double[]           Rsi14      { get; set; } = [];
    public SupertrendPoint[]  Supertrend { get; set; } = [];
    public double[]           Psar       { get; set; } = [];
}

Services/IndicatorService.cs
csharpusing TALib;
using TradingBot.Models;
using TradingBot.Config;

namespace TradingBot.Services;

public class IndicatorService
{
    private readonly StrategyConfig _cfg;

    public IndicatorService(StrategyConfig cfg) => _cfg = cfg;

    // ----------------------------------------------------------------
    // Build all indicators from a bar series
    // ----------------------------------------------------------------
    public Indicators Calculate(IReadOnlyList<Bar> bars)
    {
        int n = bars.Count;
        double[] closes = bars.Select(b => b.Close).ToArray();
        double[] highs  = bars.Select(b => b.High).ToArray();
        double[] lows   = bars.Select(b => b.Low).ToArray();

        return new Indicators
        {
            Ma7        = Sma(closes, 7),
            Ma21       = Sma(closes, 21),
            Ma44       = Sma(closes, 44),
            Rsi14      = Rsi(closes, 14),
            Supertrend = CalculateSupertrend(highs, lows, closes,
                             _cfg.SupertrendPeriod, _cfg.SupertrendMultiplier),
            Psar       = CalculatePsar(highs, lows,
                             _cfg.PsarAcceleration, _cfg.PsarMaximum)
        };
    }

    // ----------------------------------------------------------------
    // Simple Moving Average via TA-Lib
    // Returns array aligned to input length (leading values = NaN)
    // ----------------------------------------------------------------
    private static double[] Sma(double[] src, int period)
    {
        int outBegIdx, outNbElement;
        double[] output = new double[src.Length];
        Array.Fill(output, double.NaN);

        double[] result = new double[src.Length];
        Core.Sma(0, src.Length - 1, src, period,
                 out outBegIdx, out outNbElement, result);

        for (int i = 0; i < outNbElement; i++)
            output[outBegIdx + i] = result[i];

        return output;
    }

    // ----------------------------------------------------------------
    // RSI via TA-Lib
    // ----------------------------------------------------------------
    private static double[] Rsi(double[] src, int period)
    {
        int outBegIdx, outNbElement;
        double[] output = new double[src.Length];
        Array.Fill(output, double.NaN);

        double[] result = new double[src.Length];
        Core.Rsi(0, src.Length - 1, src, period,
                 out outBegIdx, out outNbElement, result);

        for (int i = 0; i < outNbElement; i++)
            output[outBegIdx + i] = result[i];

        return output;
    }

    // ----------------------------------------------------------------
    // Parabolic SAR via TA-Lib
    // ----------------------------------------------------------------
    private static double[] CalculatePsar(double[] highs, double[] lows,
                                          double acceleration, double maximum)
    {
        int outBegIdx, outNbElement;
        double[] output = new double[highs.Length];
        Array.Fill(output, double.NaN);

        double[] result = new double[highs.Length];
        Core.Sar(0, highs.Length - 1, highs, lows,
                 acceleration, maximum,
                 out outBegIdx, out outNbElement, result);

        for (int i = 0; i < outNbElement; i++)
            output[outBegIdx + i] = result[i];

        return output;
    }

    // ----------------------------------------------------------------
    // SuperTrend — manual implementation (ATR-based)
    // Not in TA-Lib; computed from scratch using TA-Lib ATR
    // ----------------------------------------------------------------
    private static SupertrendPoint[] CalculateSupertrend(
        double[] highs, double[] lows, double[] closes,
        int period, double multiplier)
    {
        int n = closes.Length;
        var result = new SupertrendPoint[n];
        for (int i = 0; i < n; i++)
            result[i] = new SupertrendPoint { Value = double.NaN,
                                              Direction = SupertrendDirection.Bullish };

        // Get ATR from TA-Lib
        int outBegIdx, outNbElement;
        double[] atrRaw = new double[n];
        Core.Atr(0, n - 1, highs, lows, closes, period,
                 out outBegIdx, out outNbElement, atrRaw);

        double[] atr = new double[n];
        Array.Fill(atr, double.NaN);
        for (int i = 0; i < outNbElement; i++)
            atr[outBegIdx + i] = atrRaw[i];

        double upperBand = 0, lowerBand = 0;
        double prevUpperBand = 0, prevLowerBand = 0;
        var prevDir = SupertrendDirection.Bullish;

        for (int i = period; i < n; i++)
        {
            if (double.IsNaN(atr[i])) continue;

            double hl2 = (highs[i] + lows[i]) / 2.0;
            double basicUpper = hl2 + multiplier * atr[i];
            double basicLower = hl2 - multiplier * atr[i];

            // Final upper band
            upperBand = (basicUpper < prevUpperBand || closes[i - 1] > prevUpperBand)
                        ? basicUpper : prevUpperBand;

            // Final lower band
            lowerBand = (basicLower > prevLowerBand || closes[i - 1] < prevLowerBand)
                        ? basicLower : prevLowerBand;

            SupertrendDirection dir;
            if (result[i - 1].Direction == SupertrendDirection.Bearish)
                dir = closes[i] > upperBand
                      ? SupertrendDirection.Bullish
                      : SupertrendDirection.Bearish;
            else
                dir = closes[i] < lowerBand
                      ? SupertrendDirection.Bearish
                      : SupertrendDirection.Bullish;

            result[i] = new SupertrendPoint
            {
                Value     = dir == SupertrendDirection.Bullish ? lowerBand : upperBand,
                Direction = dir
            };

            prevUpperBand = upperBand;
            prevLowerBand = lowerBand;
            prevDir       = dir;
        }

        return result;
    }

    // ----------------------------------------------------------------
    // Convenience: last valid value helpers
    // ----------------------------------------------------------------
    public static double LastValid(double[] arr)
    {
        for (int i = arr.Length - 1; i >= 0; i--)
            if (!double.IsNaN(arr[i])) return arr[i];
        return double.NaN;
    }

    public static double PrevValid(double[] arr)
    {
        bool foundFirst = false;
        for (int i = arr.Length - 1; i >= 0; i--)
        {
            if (double.IsNaN(arr[i])) continue;
            if (!foundFirst) { foundFirst = true; continue; }
            return arr[i];
        }
        return double.NaN;
    }
}

Services/DarvasBoxService.cs
csharpusing TradingBot.Models;
using TradingBot.Config;

namespace TradingBot.Services;

/// <summary>
/// Darvas Box: a consolidation zone defined by a swing high that holds
/// for N bars, with a floor set by the lowest low during that period.
/// </summary>
public class DarvasBoxService
{
    private readonly int _lookback;
    private DarvasBox? _activeBox;

    public DarvasBoxService(StrategyConfig cfg) => _lookback = cfg.DarvasLookback;

    public DarvasBox? ActiveBox => _activeBox;

    public void Update(IReadOnlyList<Bar> bars)
    {
        if (bars.Count < _lookback + 1) return;

        int last = bars.Count - 1;

        // Identify the highest high in the lookback window (excluding current bar)
        double boxHigh = double.MinValue;
        int    boxHighIdx = last - 1;

        for (int i = last - _lookback; i < last; i++)
        {
            if (bars[i].High > boxHigh)
            {
                boxHigh    = bars[i].High;
                boxHighIdx = i;
            }
        }

        // The box is "confirmed" if no bar in the window after the high
        // has exceeded that high (i.e., the high has held)
        bool highHeld = true;
        for (int i = boxHighIdx + 1; i < last; i++)
        {
            if (bars[i].High > boxHigh) { highHeld = false; break; }
        }

        if (!highHeld)
        {
            _activeBox = null;
            return;
        }

        // Box floor = lowest low from boxHighIdx to last-1
        double boxLow = bars
            .Skip(boxHighIdx)
            .Take(last - boxHighIdx)
            .Min(b => b.Low);

        _activeBox = new DarvasBox
        {
            High       = boxHigh,
            Low        = boxLow,
            StartIndex = boxHighIdx,
            IsActive   = true
        };
    }
}

Services/PivotService.cs
csharpusing TradingBot.Models;

namespace TradingBot.Services;

/// <summary>
/// Classic pivot point levels (Floor Pivots) computed from the
/// previous session's High / Low / Close.
/// </summary>
public class PivotService
{
    private double _pivot;
    private double _r1, _r2, _r3;
    private double _s1, _s2, _s3;

    public void Update(Bar previousSessionBar)
    {
        double h = previousSessionBar.High;
        double l = previousSessionBar.Low;
        double c = previousSessionBar.Close;

        _pivot = (h + l + c) / 3.0;
        _r1    = 2 * _pivot - l;
        _r2    = _pivot + (h - l);
        _r3    = h + 2 * (_pivot - l);
        _s1    = 2 * _pivot - h;
        _s2    = _pivot - (h - l);
        _s3    = l - 2 * (h - _pivot);
    }

    /// <summary>Returns the nearest resistance level above current price.</summary>
    public double NextResistance(double currentPrice)
    {
        double[] levels = [_r1, _r2, _r3];
        return levels.Where(r => r > currentPrice)
                     .DefaultIfEmpty(_r3)
                     .Min();
    }

    /// <summary>Returns the nearest support level below current price.</summary>
    public double NextSupport(double currentPrice)
    {
        double[] levels = [_s1, _s2, _s3];
        return levels.Where(s => s < currentPrice)
                     .DefaultIfEmpty(_s3)
                     .Max();
    }
}

Services/KiteService.cs
csharpusing KiteConnect;
using TradingBot.Config;
using TradingBot.Models;
using Microsoft.Extensions.Logging;

namespace TradingBot.Services;

public class KiteService
{
    private readonly Kite        _kite;
    private readonly KiteConfig  _cfg;
    private readonly StrategyConfig _sCfg;
    private readonly ILogger<KiteService> _log;
    private KiteTicker?          _ticker;

    // Raised when a new completed bar is available
    public event Action<Bar>? OnBarClose;

    // Internal tick aggregation
    private readonly Dictionary<uint, List<Tick>> _tickBuffer = new();
    private DateTime _currentBarTime = DateTime.MinValue;
    private readonly TimeSpan _barDuration;

    public KiteService(KiteConfig cfg, StrategyConfig sCfg,
                       ILogger<KiteService> log)
    {
        _cfg   = cfg;
        _sCfg  = sCfg;
        _log   = log;
        _kite  = new Kite(cfg.ApiKey, Debug: false);
        _kite.SetAccessToken(cfg.AccessToken);

        _barDuration = sCfg.Interval switch
        {
            "minute"    => TimeSpan.FromMinutes(1),
            "3minute"   => TimeSpan.FromMinutes(3),
            "5minute"   => TimeSpan.FromMinutes(5),
            "10minute"  => TimeSpan.FromMinutes(10),
            "15minute"  => TimeSpan.FromMinutes(15),
            "30minute"  => TimeSpan.FromMinutes(30),
            "60minute"  => TimeSpan.FromHours(1),
            "day"       => TimeSpan.FromDays(1),
            _           => TimeSpan.FromMinutes(15)
        };
    }

    // ----------------------------------------------------------------
    // Authentication — call once at startup
    // ----------------------------------------------------------------
    public void Authenticate()
    {
        if (!string.IsNullOrEmpty(_cfg.AccessToken))
        {
            _kite.SetAccessToken(_cfg.AccessToken);
            _log.LogInformation("Using cached access token.");
            return;
        }

        var session = _kite.GenerateSession(_cfg.RequestToken, _cfg.ApiSecret);
        _kite.SetAccessToken(session["access_token"].ToString()!);
        _log.LogInformation("Authenticated. Access token set.");
    }

    // ----------------------------------------------------------------
    // Historical bars for indicator warm-up
    // ----------------------------------------------------------------
    public List<Bar> GetHistoricalBars(uint instrumentToken, int count)
    {
        var to   = DateTime.Now;
        var from = to.AddDays(-count * 2); // over-fetch to account for weekends

        var raw = _kite.GetHistoricalData(
            instrumentToken.ToString(),
            from, to,
            _sCfg.Interval,
            continuous: false,
            oi: false);

        return raw.TakeLast(count)
                  .Select((c, i) => new Bar
                  {
                      Index     = i,
                      Timestamp = c.TimeStamp,
                      Open      = (double)c.Open,
                      High      = (double)c.High,
                      Low       = (double)c.Low,
                      Close     = (double)c.Close,
                      Volume    = (long)c.Volume
                  })
                  .ToList();
    }

    // ----------------------------------------------------------------
    // Instrument lookup helpers
    // ----------------------------------------------------------------
    public uint GetInstrumentToken(string exchange, string tradingSymbol)
    {
        var instruments = _kite.GetInstruments(exchange);
        var inst = instruments.FirstOrDefault(i =>
            i.TradingSymbol.Equals(tradingSymbol, StringComparison.OrdinalIgnoreCase));
        return inst != null ? (uint)inst.InstrumentToken : 0u;
    }

    /// <summary>
    /// Finds the ATM option instrument for the given underlying.
    /// expiry: nearest weekly/monthly expiry date.
    /// </summary>
    public (uint token, string symbol) GetAtmOptionToken(
        string optionType,   // "CE" or "PE"
        double spotPrice,
        DateTime expiry)
    {
        var instruments = _kite.GetInstruments(_sCfg.Exchange);

        // Round to nearest 50 for NIFTY, 100 for BANKNIFTY etc.
        double strikeStep = _sCfg.TradingSymbol.Contains("BANK") ? 100 : 50;
        double atmStrike  = Math.Round(spotPrice / strikeStep) * strikeStep;

        var opt = instruments.FirstOrDefault(i =>
            i.Name.Equals(_sCfg.TradingSymbol, StringComparison.OrdinalIgnoreCase) &&
            i.InstrumentType.Equals(optionType, StringComparison.OrdinalIgnoreCase) &&
            Math.Abs(i.Strike - (decimal)atmStrike) < 1 &&
            i.Expiry.HasValue &&
            i.Expiry.Value.Date == expiry.Date);

        if (opt == null)
            throw new InvalidOperationException(
                $"No {optionType} option found for {_sCfg.TradingSymbol} " +
                $"strike {atmStrike} expiry {expiry:dd-MMM-yyyy}");

        return ((uint)opt.InstrumentToken, opt.TradingSymbol);
    }

    /// <summary>Returns the nearest upcoming expiry date.</summary>
    public DateTime GetNearestExpiry()
    {
        var instruments = _kite.GetInstruments(_sCfg.Exchange);
        return instruments
            .Where(i => i.Name.Equals(_sCfg.TradingSymbol,
                            StringComparison.OrdinalIgnoreCase)
                     && i.Expiry.HasValue
                     && i.Expiry.Value.Date >= DateTime.Today)
            .Select(i => i.Expiry!.Value.Date)
            .Distinct()
            .OrderBy(d => d)
            .First();
    }

    // ----------------------------------------------------------------
    // Order placement
    // ----------------------------------------------------------------
    public string PlaceMarketOrder(string tradingSymbol, string transactionType,
                                   int quantity)
    {
        var response = _kite.PlaceOrder(
            Exchange:          _sCfg.Exchange,
            TradingSymbol:     tradingSymbol,
            TransactionType:   transactionType,   // "BUY" or "SELL"
            Quantity:          quantity,
            OrderType:         Constants.ORDER_TYPE_MARKET,
            Product:           Constants.PRODUCT_MIS,
            Validity:          Constants.VALIDITY_DAY,
            Variety:           Constants.VARIETY_REGULAR
        );

        string orderId = response["order_id"].ToString()!;
        _log.LogInformation("Order placed: {Symbol} {Type} qty={Qty} orderId={Id}",
                            tradingSymbol, transactionType, quantity, orderId);
        return orderId;
    }

    // ----------------------------------------------------------------
    // WebSocket ticker — subscribes and aggregates ticks into bars
    // ----------------------------------------------------------------
    public void StartTicker(uint[] tokens)
    {
        _ticker = new KiteTicker(_cfg.ApiKey, _cfg.AccessToken);

        _ticker.OnTick    += OnTick;
        _ticker.OnConnect += () =>
        {
            _log.LogInformation("Ticker connected. Subscribing to {Count} tokens.", tokens.Length);
            _ticker.Subscribe(tokens);
            _ticker.SetMode(tokens, Constants.MODE_FULL);
        };
        _ticker.OnClose   += () => _log.LogWarning("Ticker disconnected.");
        _ticker.OnError   += e  => _log.LogError("Ticker error: {Err}", e);
        _ticker.OnReconnect += () => _log.LogInformation("Ticker reconnected.");
        _ticker.OnNoReconnect += () => _log.LogCritical("Ticker gave up reconnecting.");

        _ticker.EnableReconnect(interval: 5, retries: 50);
        _ticker.Connect();
    }

    public void StopTicker() => _ticker?.Close();

    // ----------------------------------------------------------------
    // Tick → Bar aggregation
    // ----------------------------------------------------------------
    private readonly object _lock = new();
    private Bar? _buildingBar;

    private void OnTick(Tick[] ticks)
    {
        foreach (var tick in ticks)
        {
            lock (_lock)
            {
                DateTime barStart = FloorToBar(tick.Timestamp ?? DateTime.UtcNow);

                if (_buildingBar == null)
                {
                    _buildingBar = StartNewBar(tick, barStart);
                    _currentBarTime = barStart;
                    continue;
                }

                if (barStart > _currentBarTime)
                {
                    // Completed bar — fire event
                    var completed = _buildingBar;
                    OnBarClose?.Invoke(completed);

                    _buildingBar    = StartNewBar(tick, barStart);
                    _currentBarTime = barStart;
                }
                else
                {
                    // Update current bar
                    _buildingBar.High   = Math.Max(_buildingBar.High, (double)tick.High);
                    _buildingBar.Low    = Math.Min(_buildingBar.Low,  (double)tick.Low);
                    _buildingBar.Close  = (double)tick.LastPrice;
                    _buildingBar.Volume = (long)tick.VolumeTraded;
                }
            }
        }
    }

    private static Bar StartNewBar(Tick tick, DateTime barStart) => new()
    {
        Timestamp = barStart,
        Open      = (double)tick.LastPrice,
        High      = (double)tick.High,
        Low       = (double)tick.Low,
        Close     = (double)tick.LastPrice,
        Volume    = (long)tick.VolumeTraded
    };

    private DateTime FloorToBar(DateTime dt)
    {
        long ticks = dt.Ticks / _barDuration.Ticks;
        return new DateTime(ticks * _barDuration.Ticks, dt.Kind);
    }
}

Services/TradingEngine.cs
csharpusing TradingBot.Config;
using TradingBot.Models;
using TradingBot.Services;
using Microsoft.Extensions.Logging;

namespace TradingBot.Services;

public class TradingEngine
{
    private readonly KiteService      _kite;
    private readonly IndicatorService _indSvc;
    private readonly DarvasBoxService _darvasSvc;
    private readonly PivotService     _pivotSvc;
    private readonly StrategyConfig   _cfg;
    private readonly ILogger<TradingEngine> _log;

    private readonly List<Bar> _bars = new();
    private readonly Position  _position = new();
    private int _barCounter = 0;

    // Cached indicator snapshot for current bar
    private Indicators _ind = new();

    public TradingEngine(
        KiteService kite,
        IndicatorService indSvc,
        DarvasBoxService darvasSvc,
        PivotService pivotSvc,
        StrategyConfig cfg,
        ILogger<TradingEngine> log)
    {
        _kite      = kite;
        _indSvc    = indSvc;
        _darvasSvc = darvasSvc;
        _pivotSvc  = pivotSvc;
        _cfg       = cfg;
        _log       = log;
    }

    // ----------------------------------------------------------------
    // Warm-up: load historical bars before going live
    // ----------------------------------------------------------------
    public void WarmUp(uint spotToken)
    {
        _log.LogInformation("Warming up with {N} historical bars...", _cfg.LookbackBars);
        var history = _kite.GetHistoricalBars(spotToken, _cfg.LookbackBars);

        foreach (var bar in history)
        {
            bar.Index = _barCounter++;
            _bars.Add(bar);
        }

        // Seed pivot from last completed session
        if (_bars.Count >= 2)
            _pivotSvc.Update(_bars[^2]);

        _log.LogInformation("Warm-up complete. {N} bars loaded.", _bars.Count);
    }

    // ----------------------------------------------------------------
    // Called by KiteService.OnBarClose for every completed bar
    // ----------------------------------------------------------------
    public void OnBarClose(Bar bar)
    {
        bar.Index = _barCounter++;
        _bars.Add(bar);

        // Keep memory bounded
        if (_bars.Count > _cfg.LookbackBars + 50)
            _bars.RemoveAt(0);

        // Recalculate all indicators
        _ind = _indSvc.Calculate(_bars);

        // Update Darvas box
        _darvasSvc.Update(_bars);

        // Update pivot (use yesterday's bar as proxy for previous session)
        if (_bars.Count >= 2)
            _pivotSvc.Update(_bars[^2]);

        // Main strategy loop
        if (_position.PositionType == PositionType.None)
            CheckEntry(bar);
        else
        {
            UpdateTrailingSl(bar);
            CheckExit(bar);
        }
    }

    // ================================================================
    // ENTRY LOGIC
    // ================================================================
    private void CheckEntry(Bar bar)
    {
        int last = _bars.Count - 1;

        // ---- CE (uptrend) ----
        if (MaCrossedUp(_ind.Ma7, _ind.Ma21))
        {
            if (SupertrendConfirmsWithinWindow("bullish", bars: 2))
            {
                if (bar.Close > IndicatorService.LastValid(_ind.Ma44))
                {
                    double rsi = IndicatorService.LastValid(_ind.Rsi14);
                    if (rsi >= 40 && rsi <= 70)
                    {
                        var box = _darvasSvc.ActiveBox;
                        if (box == null || bar.Close > box.High)
                        {
                            EnterPosition(PositionType.CE, bar, box);
                            return;
                        }
                    }
                }
            }
        }

        // ---- PE (downtrend) ----
        if (MaCrossedDown(_ind.Ma7, _ind.Ma21))
        {
            if (SupertrendConfirmsWithinWindow("bearish", bars: 2))
            {
                if (bar.Close < IndicatorService.LastValid(_ind.Ma44))
                {
                    double rsi = IndicatorService.LastValid(_ind.Rsi14);
                    if (rsi >= 30 && rsi <= 60)
                    {
                        var box = _darvasSvc.ActiveBox;
                        if (box == null || bar.Close < box.Low)
                        {
                            EnterPosition(PositionType.PE, bar, box);
                        }
                    }
                }
            }
        }
    }

    private void EnterPosition(PositionType type, Bar bar, DarvasBox? box)
    {
        double stValue = CurrentSupertrendValue();

        _position.PositionType  = type;
        _position.EntryPrice    = bar.Close;
        _position.EntryBarIndex = bar.Index;
        _position.DarvasBoxUsed = box;

        if (type == PositionType.CE)
        {
            double structuralSl = box?.Low ?? stValue;
            _position.InitialSl = Math.Min(structuralSl, stValue);
        }
        else
        {
            double structuralSl = box?.High ?? stValue;
            _position.InitialSl = Math.Max(structuralSl, stValue);
        }

        _position.TrailingSl = _position.InitialSl;

        // Resolve option instrument
        try
        {
            var expiry = _kite.GetNearestExpiry();
            var (token, symbol) = _kite.GetAtmOptionToken(
                type == PositionType.CE ? "CE" : "PE",
                bar.Close, expiry);

            _position.InstrumentToken = token;
            _position.OptionSymbol    = symbol;

            _kite.PlaceMarketOrder(symbol, "BUY", _cfg.Quantity);

            _log.LogInformation(
                "ENTRY {Type} | Symbol={Symbol} | EntryPrice={Price:F2} | SL={Sl:F2}",
                type, symbol, bar.Close, _position.InitialSl);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "Failed to place entry order. Resetting position.");
            _position.Reset();
        }
    }

    // ================================================================
    // TRAILING STOP LOSS
    // ================================================================
    private void UpdateTrailingSl(Bar bar)
    {
        double stValue   = CurrentSupertrendValue();
        double psarValue = CurrentPsarValue();

        if (_position.PositionType == PositionType.CE)
        {
            // SL only moves UP
            if (stValue > _position.TrailingSl)
                _position.TrailingSl = stValue;

            if (psarValue < bar.Close && psarValue > _position.TrailingSl)
                _position.TrailingSl = psarValue;
        }
        else if (_position.PositionType == PositionType.PE)
        {
            // SL only moves DOWN
            if (stValue < _position.TrailingSl)
                _position.TrailingSl = stValue;

            if (psarValue > bar.Close && psarValue < _position.TrailingSl)
                _position.TrailingSl = psarValue;
        }
    }

    // ================================================================
    // EXIT LOGIC
    // ================================================================
    private void CheckExit(Bar bar)
    {
        if (_position.PositionType == PositionType.CE)
        {
            if (bar.Low <= _position.TrailingSl)
            { ClosePosition("SL_HIT"); return; }

            if (PsarFlippedAbovePrice(bar))
            { ClosePosition("PSAR_FLIP"); return; }

            if (SupertrendFlipped("bearish"))
            { ClosePosition("SUPERTREND_FLIP"); return; }

            if (bar.High >= _pivotSvc.NextResistance(bar.Close))
            { ClosePosition("PIVOT_TARGET"); return; }
        }
        else if (_position.PositionType == PositionType.PE)
        {
            if (bar.High >= _position.TrailingSl)
            { ClosePosition("SL_HIT"); return; }

            if (PsarFlippedBelowPrice(bar))
            { ClosePosition("PSAR_FLIP"); return; }

            if (SupertrendFlipped("bullish"))
            { ClosePosition("SUPERTREND_FLIP"); return; }

            if (bar.Low <= _pivotSvc.NextSupport(bar.Close))
            { ClosePosition("PIVOT_TARGET"); return; }
        }
    }

    private void ClosePosition(string reason)
    {
        _log.LogInformation(
            "EXIT {Type} | Reason={Reason} | Symbol={Symbol} | TrailSL={Sl:F2}",
            _position.PositionType, reason, _position.OptionSymbol, _position.TrailingSl);

        try
        {
            if (!string.IsNullOrEmpty(_position.OptionSymbol))
                _kite.PlaceMarketOrder(_position.OptionSymbol, "SELL", _cfg.Quantity);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "Failed to place exit order for {Symbol}",
                          _position.OptionSymbol);
        }

        LogTrade(reason);
        _position.Reset();
    }

    // ================================================================
    // INDICATOR HELPERS
    // ================================================================

    private bool MaCrossedUp(double[] fast, double[] slow)
    {
        double fastNow  = IndicatorService.LastValid(fast);
        double fastPrev = IndicatorService.PrevValid(fast);
        double slowNow  = IndicatorService.LastValid(slow);
        double slowPrev = IndicatorService.PrevValid(slow);

        return fastPrev <= slowPrev && fastNow > slowNow;
    }

    private bool MaCrossedDown(double[] fast, double[] slow)
    {
        double fastNow  = IndicatorService.LastValid(fast);
        double fastPrev = IndicatorService.PrevValid(fast);
        double slowNow  = IndicatorService.LastValid(slow);
        double slowPrev = IndicatorService.PrevValid(slow);

        return fastPrev >= slowPrev && fastNow < slowNow;
    }

    private bool SupertrendConfirmsWithinWindow(string direction, int bars)
    {
        var st = _ind.Supertrend;
        int start = Math.Max(0, st.Length - bars);
        var window = st.Skip(start).Take(bars);

        return direction == "bullish"
            ? window.Any(p => p.Direction == SupertrendDirection.Bullish)
            : window.Any(p => p.Direction == SupertrendDirection.Bearish);
    }

    private bool SupertrendFlipped(string toDirection)
    {
        var st = _ind.Supertrend;
        if (st.Length < 2) return false;

        var curr = st[^1];
        var prev = st[^2];

        return toDirection == "bearish"
            ? prev.Direction == SupertrendDirection.Bullish
              && curr.Direction == SupertrendDirection.Bearish
            : prev.Direction == SupertrendDirection.Bearish
              && curr.Direction == SupertrendDirection.Bullish;
    }

    private double CurrentSupertrendValue()
    {
        var st = _ind.Supertrend;
        for (int i = st.Length - 1; i >= 0; i--)
            if (!double.IsNaN(st[i].Value)) return st[i].Value;
        return double.NaN;
    }

    private double CurrentPsarValue() =>
        IndicatorService.LastValid(_ind.Psar);

    private bool PsarFlippedAbovePrice(Bar bar)
    {
        double psarNow  = IndicatorService.LastValid(_ind.Psar);
        double psarPrev = IndicatorService.PrevValid(_ind.Psar);
        // PSAR was below price, now above price
        return psarPrev < bar.Close && psarNow > bar.Close;
    }

    private bool PsarFlippedBelowPrice(Bar bar)
    {
        double psarNow  = IndicatorService.LastValid(_ind.Psar);
        double psarPrev = IndicatorService.PrevValid(_ind.Psar);
        return psarPrev > bar.Close && psarNow < bar.Close;
    }

    // ================================================================
    // TRADE LOGGING
    // ================================================================
    private void LogTrade(string reason)
    {
        var line = string.Join(",",
            DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"),
            _position.PositionType,
            _position.OptionSymbol ?? "",
            _position.EntryPrice.ToString("F2"),
            _position.InitialSl.ToString("F2"),
            _position.TrailingSl.ToString("F2"),
            reason);

        File.AppendAllText(_cfg.LogPath, line + Environment.NewLine);
    }
}

Program.cs
csharpusing Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using TradingBot.Config;
using TradingBot.Services;

// ----------------------------------------------------------------
// Configuration
// ----------------------------------------------------------------
var config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false)
    .Build();

var kiteConfig    = config.GetSection("Kite").Get<KiteConfig>()!;
var strategyConfig = config.GetSection("Strategy").Get<StrategyConfig>()!;

// ----------------------------------------------------------------
// DI Container
// ----------------------------------------------------------------
var services = new ServiceCollection()
    .AddLogging(b => b.AddConsole().SetMinimumLevel(LogLevel.Information))
    .AddSingleton(kiteConfig)
    .AddSingleton(strategyConfig)
    .AddSingleton<KiteService>()
    .AddSingleton<IndicatorService>()
    .AddSingleton<DarvasBoxService>()
    .AddSingleton<PivotService>()
    .AddSingleton<TradingEngine>()
    .BuildServiceProvider();

var logger  = services.GetRequiredService<ILogger<Program>>();
var kite    = services.GetRequiredService<KiteService>();
var engine  = services.GetRequiredService<TradingEngine>();

// ----------------------------------------------------------------
// Authenticate
// ----------------------------------------------------------------
kite.Authenticate();

// ----------------------------------------------------------------
// Resolve spot instrument token (e.g. NIFTY 50 on NSE)
// ----------------------------------------------------------------
uint spotToken = kite.GetInstrumentToken(
    strategyConfig.SpotExchange,
    strategyConfig.Underlying);

logger.LogInformation("Spot token for {Sym}: {Token}",
    strategyConfig.Underlying, spotToken);

// ----------------------------------------------------------------
// Warm up indicators with historical data
// ----------------------------------------------------------------
engine.WarmUp(spotToken);

// ----------------------------------------------------------------
// Wire live bar events
// ----------------------------------------------------------------
kite.OnBarClose += bar =>
{
    try   { engine.OnBarClose(bar); }
    catch (Exception ex)
    { logger.LogError(ex, "Unhandled error in OnBarClose"); }
};

// ----------------------------------------------------------------
// Start WebSocket ticker
// ----------------------------------------------------------------
kite.StartTicker(new[] { spotToken });

logger.LogInformation("Bot running. Press Ctrl+C to stop.");

// ----------------------------------------------------------------
// Graceful shutdown
// ----------------------------------------------------------------
var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;
    logger.LogInformation("Shutdown requested...");
    kite.StopTicker();
    cts.Cancel();
};

await Task.Delay(Timeout.Infinite, cts.Token)
          .ContinueWith(_ => { });

logger.LogInformation("Bot stopped.");

Would you like me to explain or break down the code?
A few important notes before you deploy:


Access Token flow — Kite Connect requires a fresh access_token each day via browser login. You'll need to add a small OAuth redirect handler or manually paste the token into appsettings.json each morning.


TALib.NETCore requires the native TA-Lib C library to be present on the host. On Linux: apt install libta-lib-dev; on Windows, copy the DLL alongside the binary.


Paper-trade first — replace PlaceMarketOrder calls with log statements until you've validated signal quality on historical data.


Option symbol resolution — GetAtmOptionToken assumes NIFTY strike steps of 50. Adjust strikeStep for BANKNIFTY (100) or other underlyings.

