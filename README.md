# SOLARIS

![SOLARIS](COVER.jpg)

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
│   ├── settings.csv               ← thresholds and execution parameters
│   └── schedule.csv               ← generated walk-forward schedule
├── DATA/
│   ├── EQUITY.xlsx                ← raw equity price data
│   ├── CRYPTO.xlsx                ← raw crypto price data
│   ├── FOREX.xlsx                 ← raw FX price data
│   ├── COMODITIES.xlsx            ← raw commodities price data
│   ├── FULL_Forward_Fill.xlsx     ← merged (outer join, all calendar days)
│   └── FULL_Trading_Calendar.xlsx ← merged (left join, equity trading days only)
├── OUTPUT/                        ← timestamped simulation runs (IS + OOS results)
├── SCHEDULE.ipynb                 ← walk-forward schedule generator
├── LOADER.ipynb                   ← main ETL pipeline
├── IS-Engine.ipynb                ← in-sample pair selection (correlation + cointegration)
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

### 2. LOADER.ipynb — Multi-Asset ETL Pipeline
Fetches cross-asset market data, resolves structural discrepancies, and standardizes trading calendars.
- Parses `CSV/Tickers.csv` with `keep_default_na=False`
- Converts `BRK.B` → `BRK-B` for yfinance compatibility
- Applies `.strip()` to remove whitespace from tickers
- Normalizes timestamps to ISO format (`YYYY-MM-DD`)
- Produces two calendar-aligned master files: forward-fill and trading-calendar versions

### 3. IS-Engine.ipynb — In-Sample Pair Selection
Funnel of independent filter functions, chained on one walk-forward window at a time.
- `filter_correlation()` — Pearson correlation on **log returns**, cross-asset only (excludes Equity-Equity)
- `filter_cointegration()` — Engle-Granger test on **price levels**, run only on pairs that survived correlation
- Thresholds (`min_correlation`, `p_value`) read from `CSV/settings.csv`
- Still to add: Half-Life and Hurst exponent filters, then the loop over all `schedule.csv` windows

---

## Pipeline

```
Phase 1 — SCHEDULE                    ✅ done
Phase 2 — LOADER                      ✅ done
Phase 3 — IS Engine                   🔧 in development (correlation + cointegration done)
Phase 4 — OOS Backtest                🔧 in development
```
