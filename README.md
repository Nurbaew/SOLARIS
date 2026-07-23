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
│   ├── settings.csv               ← active IS scenario + filter thresholds
│   ├── settings_oos.csv           ← active OOS scenario + trading/positioning parameters
│   └── schedule.csv               ← generated walk-forward schedule
├── DATA/
│   ├── EQUITY.xlsx                ← raw equity price data
│   ├── CRYPTO.xlsx                ← raw crypto price data
│   ├── FOREX.xlsx                 ← raw FX price data
│   ├── COMODITIES.xlsx            ← raw commodities price data
│   ├── FULL_Forward_Fill.xlsx     ← merged (outer join, all calendar days)
│   ├── FULL_Trading_Calendar.xlsx ← merged (left join, equity trading days only)
│   └── Output/
│       └── {IS_scenario}/
│           ├── IS_{IS_scenario}.xlsx           ← IS-Engine results, one file per scenario run
│           └── {z_window}/
│               └── OOS_{OOS_scenario}.xlsx     ← OOS-Engine results, one file per (IS scenario, z_window, OOS scenario)
├── SCHEDULE.ipynb                 ← walk-forward schedule generator
├── LOADER.ipynb                   ← main ETL pipeline
├── IS-Engine.ipynb                ← in-sample pair selection (4-filter funnel)
├── OOS-Engine.ipynb               ← out-of-sample walk-forward backtest engine
└── README.md
```

---

## Asset Universe

| Class | Tickers | Source |
|-------|---------|--------|
| Equity | 503 S&P 500 constituents | yfinance |
| Crypto | BTC-USD, ETH-USD, SOL-USD, XRP-USD, LTC-USD, DOGE-USD, ADA-USD, BNB-USD | yfinance |
| FX | EUR/USD, GBP/USD, USD/JPY, AUD/USD, USD/CAD, USD/CHF, NZD/USD, EUR/GBP, EUR/JPY | yfinance |
| Commodities | GC=F (Gold), SI=F (Silver), PL=F (Platinum), PA=F (Palladium), HG=F (Copper) | yfinance |

525 tickers total. Note: some crypto tickers have shorter history on Yahoo Finance than the 2015 dataset start — BTC/LTC go back to 2015, ETH/XRP/ADA/DOGE/BNB start 2017-11-09, SOL starts 2020-04-10. Windows before an asset's start date simply exclude it (handled by `dropna` in `filter_correlation()`).

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

Thresholds and the active scenario name are read from `CSV/settings.csv`. Output file `DATA/Output/{scenario_name}/IS_{scenario_name}.xlsx` has three sheets:
- `Results` — selected pairs: `Iteration, Asset_A, Asset_B, Type_A, Type_B, Pearson_Corr, P_Value, Half_Life, Beta, Intercept, Hurst` (`Beta`/`Intercept` let the OOS engine reconstruct the same spread without retraining)
- `Funnel_Summary` — average pairs surviving each filter stage, aggregated across all iterations
- `Funnel_Per_Iteration` — same funnel breakdown per individual walk-forward iteration

**Presets** (5 levels, named after the thesis's own PURE/EASY/MEDIUM/HARD/ULTRA scale; thresholds are our own — calibrated on real IS-window data, `min_correlation`/`p_value` grounded in Liu, `Half_Life`/`Hurst` bounds in Chan):

| Parameter | PURE | EASY | MEDIUM | HARD | ULTRA |
|---|---|---|---|---|---|
| min_correlation | 0.10 | 0.15 | 0.20 | 0.25 | 0.30 |
| p_value | 0.10 | 0.075 | 0.05 | 0.025 | 0.01 |
| min_half_life | 1 | 1.5 | 2 | 2.5 | 3 |
| max_half_life | 15 | 13 | 11 | 9 | 8 |
| min_hurst | 0 | 0 | 0 | 0 | 0 |
| max_hurst | 0.40 | 0.37 | 0.35 | 0.31 | 0.28 |

### 4. OOS-Engine.ipynb — Out-of-Sample Walk-Forward Backtest
Day-by-day simulation over the pairs selected by IS-Engine, run separately for every walk-forward iteration in `schedule.csv`.

- **Positioning:** beta-neutral by default (`Qty_B = |Beta| × Qty_A`, using the hedge ratio stored by IS-Engine); dollar-neutral available as a toggle (`positioning_mode`) for ablation testing.
- **Entry:** rolling Z-score of the spread (`z_window` days, continuous — not reset at month boundaries), entry when `|z| > z_entry` and a slot is free (`max_open_pairs`).
- **Exit:** convergence (`|z| <= z_exit`) or time-based stop-loss (`duration > half_life × hl_multiplier`); a pair that times out is banned from re-entry until `|z|` returns to `z_exit`.
- **Trailing tail:** after the last scheduled iteration, the engine keeps simulating (no new entries) until every open position closes naturally, instead of cutting off at an arbitrary date.
- **Guard rail:** a minimum hedge-ratio check (`min_hedge_ratio_pct`) rejects near-degenerate hedges where one leg's notional is negligible relative to the other (matters for extreme cross-asset beta values, e.g. altcoin vs. large-cap equity).

Thresholds and the active scenario are read from `CSV/settings_oos.csv`. Output file `DATA/Output/{IS_scenario}/{z_window}/OOS_{OOS_scenario}.xlsx` has five sheets: `OOS_Trades`, `Yearly_Stats`, `Summary`, `Exit_Analysis`, `Open_Positions`.

**Presets** (5 levels, separate axis from the IS presets above — these control trading strictness, not pair-selection strictness):

| Parameter | PURE | EASY | MEDIUM | HARD | ULTRA |
|---|---|---|---|---|---|
| z_entry | 1.5 | 1.75 | 2.0 | 2.25 | 2.5 |
| z_exit | 0.75 | 0.5 | 0.25 | 0.1 | 0 |
| hl_multiplier | 4.0 | 3.25 | 2.5 | 1.75 | 1.0 |
| max_open_pairs | 15 | 12 | 10 | 7 | 5 |

Full evaluation matrix: 5 IS presets × 5 OOS presets × 2 z-windows (30, 60) = 50 runs, plus a dollar-neutral ablation run on selected configurations.

---

## Pipeline

```
Phase 1 — SCHEDULE                    ✅ done
Phase 2 — LOADER                      ✅ done
Phase 3 — IS Engine                   ✅ done (Correlation → Cointegration → Half-Life → Hurst)
Phase 4 — OOS Engine                  ✅ done (beta-neutral walk-forward backtest, trailing tail)
Phase 5 — Full IS × OOS grid          ✅ done (25 configurations × z_window=60; robustness check on z_window=30)
Phase 6 — Article writeup             🔧 in progress
```
