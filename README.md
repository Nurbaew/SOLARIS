# SOLARIS

## Cross-Asset Pair Trading: Multi-Criteria Selection Framework (SOLARIS)

This repository contains the Python framework developed for empirical computations and validation of the research paper:  
**"Cross-Asset Pair Trading: A Multi-Criteria Selection Framework Using Cointegration, Mean Reversion, and Walk-Forward Validation"** *Prague University of Economics and Business (VŠE Praha)*

The framework expands upon previous sector-equity models by implementing a multi-criteria selection pipeline for cross-asset pairs (Analyzing Stocks, FX, Gold, and Cryptocurrencies) validated via a rolling Walk-Forward framework.

---

## Repository Structure

* `CSV/` — Contains configuration files, including `settings.csv` (thresholds and execution parameters), `TICKERS.csv`, and the generated `schedule.csv`.
* `DATA/` — Storage for preprocessed, synchronized, and cleaned price data across all asset classes (`EQUITY.xlsx`, `CRYPTO.xlsx`, `FOREX.xlsx`, `COMODITIES.xlsx`).
* `OUTPUT/` — Target directory for independent, timestamped simulation runs containing In-Sample and Out-of-Sample results.

---

## Core Scripts Description

### 1. SCHEDULE.ipynb (Walk-Forward Schedule Generator)
* **Purpose:** Establishes the temporal framework for the rolling backtest. It segments the historical data into sequential, non-overlapping evaluation slices to mathematically eliminate look-ahead bias during the optimization phase.
* **Mechanism:** Accepts dynamic user inputs via `input()` for the target chronological range (`start`/`end` years) and the training window depth (`is_window` in months). It computes the precise `OOS_START` and `OOS_END` for each month (handling leap years via `calendar.monthrange`), and then maps the corresponding historical `In-Sample` anchors.
* **Output:** Exports a structured matrix to `CSV/schedule.csv` using a semicolon (`;`) delimiter.

### 2. LOADER.ipynb (Multi-Asset ETL Pipeline)
* **Purpose:** Functions as the primary ETL (Extract, Transform, Load) engine designed to fetch cross-asset market data, resolve structural data discrepancies, and standardize heterogeneous trading calendars.
* **Mechanism:** * Parses `CSV/TICKERS.csv` using `keep_default_na=False` to prevent empty fields from being misinterpreted as NaN values.
  * Adjusts corporate equity equity tickers to match corporate mapping syntax (e.g., converting S&P 500 dot notation to Yahoo Finance dash format).
  * Implements token trimming (`.strip()`) to isolate raw ticker strings from accidental Whitespace padding.
  * Transforms raw index timestamps into atomic ISO dates (`YYYY-MM-DD`), laying the groundwork for mathematical alignment between 24/7 digital asset markets and standard weekday exchange sessions.
* **Output:** Generates four independent binary Excel sheets inside the `DATA/` directory: `EQUITY.xlsx`, `CRYPTO.xlsx`, `FOREX.xlsx`, and `COMODITIES.xlsx`.

---

## Pipeline & Project Roadmap

- [x] **Phase 1: Walk-Forward Schedule Generator (SCHEDULE)** Generates rolling, non-overlapping In-Sample and Out-of-Sample evaluation intervals (`CSV/schedule.csv`) based on user-defined window lengths to prevent look-ahead bias.

- [x] **Phase 2: Data Loader & Preprocessor (LOADER)** * **API Ingestion:** Multi-threaded historical data harvesting via `yfinance` API for S&P 500 equities, Forex pairs, precious metals, and major cryptocurrencies spanning 11 years (2015–2025).
  * **Data Hygiene:** Built-in sanitization for asset-specific ticker syntax (e.g., automated conversion of `BRK.B` dots to `BRK-B` dashes) and string formatting (`.strip()`).
  * **Calendar Standardization:** Index-level timestamp normalization converting datetime strings to clean ISO dates (`.date`).

- [ ] **Phase 3: In-Sample (IS) Engine** * Cross-asset pair construction (Stock–Gold, Crypto–Stock, FX–Commodity).
  * Statistical pipeline: ADF, Engle-Granger cointegration, OLS Hedge Ratio, Half-Life of mean reversion, and Hurst exponent.
  * Multi-criteria filtering based on thresholds from `CSV/settings.csv`.

- [ ] **Phase 4: Out-of-Sample (OOS) Backtest** * Forward performance simulation using rolling Z-score entry/exit rules and time-stops.
  * Beta-neutral and Dollar-neutral capital allocation regimes.
  * Performance logging (Sharpe ratio, Max Drawdown, Profit Factor, Turnover) exported to `OUTPUT/`.