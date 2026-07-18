# SOLARIS

<img src="COVER.jpg" alt="SOLARIS" width="420">

## Cross-Asset Pair Trading: Multi-Criteria Selection Framework

This repository contains the Python framework developed for empirical computations and validation of the research paper:

> **"Cross-Asset Pair Trading: A Multi-Criteria Selection Framework Using Cointegration, Mean Reversion, and Walk-Forward Validation"**
> Prague University of Economics and Business (VŠE Praha)

The framework implements a multi-criteria statistical pipeline for cross-asset pairs (S&P 500 Equities, FX, Commodities, and Cryptocurrencies), validated via a rolling Walk-Forward framework.

---

## Repository Structure

```
SOLARIS/
├── CSV/
│   ├── Tickers.csv                ← asset tickers by class (EQUITY, CRYPTO, FOREX, COMODITIES)
│   ├── settings.csv               ← active scenario + filter thresholds
│   └── schedule.csv               ← generated walk-forward schedule
├── DATA/
│   ├── EQUITY.xlsx                ← raw equity price data
│   ├── CRYPTO.xlsx                ← raw crypto price data
│   ├── FOREX.xlsx                 ← raw FX price data
│   ├── COMODITIES.xlsx            ← raw commodities price data
│   ├── FULL_Forward_Fill.xlsx     ← merged (outer join, all calendar days)
│   ├── FULL_Trading_Calendar.xlsx ← merged (left join, equity trading days only)
│   └── Output/
│       └── {scenario_name}/{scenario_name}.xlsx  ← IS-Engine results, one file per scenario run
├── SCHEDULE.ipynb                 ← walk-forward schedule generator
├── LOADER.ipynb                   ← main ETL pipeline
├── IS-Engine.ipynb                ← in-sample pair selection (4-filter funnel)
└── README.md
```

---

## Asset Universe

| Class | Tickers | Source |
|-------|---------|--------|
| Equity | ~500 S&P 500 constituents | yfinance |
| Crypto | BTC-USD, ETH-USD, SOL-USD | yfinance |
| FX | EUR/USD, GBP/USD, USD/JPY, AUD/USD, USD/CAD | yfinance |
| Commodities | GC=F (Gold), SI=F (Silver) | yfinance |

---

## Cross-Asset Pair Structure

Only Equity-Equity pairs are excluded (too many, not the research focus). Every other combination — including same-class pairs like Crypto-Crypto or Commodities-Commodities — is allowed.

|  | Equity | Crypto | FX | Commodities |
|--|:--:|:--:|:--:|:--:|
| **Equity** | ❌ | ✅ | ✅ | ✅ |
| **Crypto** | - | ✅ | ✅ | ✅ |
| **FX** | - | - | ✅ | ✅ |
| **Commodities** | - | - | - | ✅ |

---

## Scripts

### 1. SCHEDULE.ipynb — Walk-Forward Schedule Generator
Generates sequential, non-overlapping IS/OOS evaluation windows to eliminate look-ahead bias.
- **Inputs:** start year, end year, IS window length (months)
- **Output:** `CSV/schedule.csv`
- **Current schedule:** OOS 2017–2025, IS lookback 12 months, monthly step → 108 iterations

### 2. LOADER.ipynb — Multi-Asset ETL Pipeline
Fetches cross-asset market data, resolves structural discrepancies, and standardizes trading calendars.
- Parses `CSV/Tickers.csv` with `keep_default_na=False`
- Converts `BRK.B` → `BRK-B` for yfinance compatibility
- Applies `.strip()` to remove whitespace from tickers
- Normalizes timestamps to ISO format (`YYYY-MM-DD`)
- Produces two calendar-aligned master files: forward-fill and trading-calendar versions

### 3. IS-Engine.ipynb — In-Sample Pair Selection
Funnel of four independent filter functions, run for every window in `schedule.csv` and accumulated into a single output file per scenario.

1. `filter_correlation()` — Pearson correlation on **log returns**, cross-asset only (excludes Equity-Equity)
2. `filter_cointegration()` — Engle-Granger test, tested in **both directions** (A~B and B~A, following Chan's CADF approach); the better-fitting direction becomes `Asset_A` (dependent) / `Asset_B` (regressor)
3. `filter_half_life()` — Half-Life of mean reversion (Chan, 2013); also stores the OLS hedge ratio (`Beta`, `Intercept`) used to build the spread
4. `filter_hurst()` — generalized Hurst exponent (variance-of-lags method, Chan 2013), confirms H < 0.5

Thresholds and the active scenario name are read from `CSV/settings.csv`. Output columns: `Iteration, Asset_A, Asset_B, Type_A, Type_B, Pearson_Corr, P_Value, Half_Life, Beta, Intercept, Hurst`, saved to `DATA/Output/{scenario_name}/{scenario_name}.xlsx` — `Beta`/`Intercept` let the OOS engine reconstruct the same spread without retraining.

**Presets** (thresholds calibrated on real IS-window data; `min_correlation`/`p_value` grounded in Liu, `Half_Life`/`Hurst` bounds in Chan):

| Parameter | EASY | MEDIUM | HARD |
|---|---|---|---|
| min_correlation | 0.10 | 0.20 | 0.30 |
| p_value | 0.10 | 0.05 | 0.01 |
| min_half_life | 1 | 2 | 3 |
| max_half_life | 15 | 11 | 8 |
| min_hurst | 0 | 0 | 0 |
| max_hurst | 0.40 | 0.35 | 0.28 |

---

## Pipeline

```
Phase 1 — SCHEDULE                    ✅ done
Phase 2 — LOADER                      ✅ done
Phase 3 — IS Engine                   ✅ done (Correlation → Cointegration → Half-Life → Hurst)
Phase 4 — OOS Backtest                🔧 not started
```
