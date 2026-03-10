# Convertible Bond Arbitrage Backtest

Systematic, delta‑hedged **convertible bond arbitrage** project.

We build a Treasury/BBB discount curve, price embedded equity options with a **Cox–Ross–Rubinstein (CRR)** binomial tree, construct **bond floors + fair values**, and run a **multi‑bond backtest** with full P&L attribution and alpha/beta vs **S&P 500** and a **60/40 portfolio**.

---

## 1. Project Overview

- **Goal**: Test a **delta‑hedged long–short** strategy that exploits **structural mispricings** between model‑implied fair value and market prices of U.S. investment‑grade convertible bonds.
- **Universe**: IG convertibles (e.g., MTH, DUK, NEE, DLR, KRG, FRT, PPL2, REXR1/2, VTR, WEC3, etc.).
- **Horizon**: Daily data from **2023‑01‑01 to 2026‑02‑27** (configurable).
- **Key steps**:
  1. Build a **daily zero‑curve** and BBB spread → `final_discount_rates_daily.csv`.
  2. Price embedded equity options with a **CRR binomial tree** (American, soft call handling).
  3. Compute **bond floors**, **option values**, and **fair values**.
  4. Run a **position‑level backtest**: long bond + short stock, dynamic delta hedging.
  5. Analyse **P&L attribution** and **CAPM alpha/beta** vs S&P 500 and a 60/40 benchmark.

---

## 2. Strategy in One Sentence

> A **systematic, dynamically delta‑hedged long–short strategy** that exploits **structural mispricings** between **model‑implied fair value** and **market prices** of convertible bonds, with residual gamma exposure and controlled credit/interest‑rate risk via the bond floor.

---

## 3. Data Pipeline

### Raw Inputs

- **Static bond terms**  
  `data/raw/bond_static/CONV_BOND_DATA.xlsx`  
  (Ticker, coupon, maturity, conversion ratio, conversion price, etc.)

- **Bond price time series**  
  `data/raw/bond_dynamic/{TICKER}_bond.xlsx`  
  (Daily clean/dirty prices per bond.)

- **Equity / implied volatility**  
  `data/raw/stock/{TICKER}_equity.xlsx`  
  (Daily stock close and 12M ATM implied vol.)

- **Yield curve / credit spread**  
  Treasury & BBB data → cleaned in `yield_curve.ipynb`.

- **Benchmark returns**  
  `code/sp500_and_6040_daily_returns_20230101_20260227.csv`  
  (Daily S&P 500 and 60/40 portfolio returns.)

### Cleaned Outputs

- `data/clean/yield_curve_clean.csv`  
- `data/clean/final_discount_rates_daily.csv` (risk‑free and BBB discount rates)
- `data/clean/option_values/{TICKER}_options.csv`
- `data/clean/bond_floors/{TICKER}_bond_floor.csv`
- `data/clean/fair_value/{TICKER}_fair_value.csv`
- `data/clean/fair_value_all.csv`
- `data/clean/backtest/portfolio_nav.csv`, `pnl_daily.csv`, `trades.csv`, `performance.csv`

---

## 4. Modules / Notebooks

- **`code/yield_curve.ipynb`**  
  - Cleans Treasury and credit spread data.  
  - Builds a daily **zero‑coupon curve** and BBB discount rates.  
  - Outputs `final_discount_rates_daily.csv`.

- **`code/option_pricer.ipynb`**  
  - CRR **binomial tree** for American calls on the equity.  
  - Handles:
    - Dynamic maturity,
    - Interpolated risk‑free rate from the zero‑curve,
    - 12M ATM implied vol,
    - Optional soft call feature.  
  - Outputs daily **option values and delta** per bond.

- **`code/bond_floor_calculation.ipynb`**  
  - Computes the **straight‑bond floor**: PV of all coupons + principal, discounted with (ZC Treasury + unified BBB spread).  
  - Ignores issuer calls for the floor (price to maturity).

- **`code/fair_price_calculation.ipynb`**  
  - Merges **bond floor** and **option value** streams:  
    `fair_value = clean_floor + option_value_pct`.  
  - Computes daily **mispricing**: `fair_value_clean − market_clean_price`.  
  - Flags **signal days** where mispricing exceeds a threshold (e.g. > 3% of par).  
  - Writes per‑bond fair value CSVs and a combined `fair_value_all.csv`.  
  - Builds a **recommended bond universe** for backtesting based on:
    - history length,
    - average mispricing,
    - signal frequency.

- **`code/bond_backtest.ipynb`**  
  - Core **backtest engine**:
    - Long **convertible bond**, short **underlying stock**.
    - Position sizing: fixed notional per bond (e.g. $1M).
    - **Delta hedging**: `shares_short = delta × conversion_ratio × N_bonds`, with rebalancing when:
      - |Δ_new − Δ_old| > bond‑specific `rebal_trigger`, or  
      - |stock move| > 10% in a day.
    - Exit rules:
      - Mispricing convergence,
      - Max holding period,
      - Maturity buffer,
      - `no_data` (missing market data),
      - End of sample.
  - Produces:
    - Portfolio NAV and daily returns,
    - Full **P&L attribution**:
      - coupon, bond MTM, short leg, gamma, borrow, rebate, TC, cash income,
    - **Per‑bond breakdown** (trades, win rate, total P&L, avg entry mispricing).
  - Diagnostics:
    - Daily return distribution,
    - Max drawdown window,
    - Big‑move days,
    - Bond MTM vs theoretical MTM check,
    - Gamma / residual P&L decomposition.

- **`code/build_6040_and_sp500_returns_fixed.py`**  
  - (Optional helper) builds **S&P 500** and **60/40** daily return series from Yahoo/ETF data.

---

## 5. Backtest Summary (current configuration)

- **Period**: 2023‑01‑03 to 2026‑02‑27 (~3.1 years)  
- **Starting capital**: $10,000,000  
- **Position size**: $1,000,000 per bond  
- **Avg capital deployed**: ~61.5%  
- **Total return**: ~**20.8%**  
- **Ann. return**: ~**6.2%**  
- **Ann. volatility**: ~**3.8%**  
- **Sharpe (daily, rf from curve)**: ~**0.43**  
- **Max drawdown**: ~**−1.44%**  
- **Total trades**: 78 positions, 56.4% win rate, profit factor ~3.0

**P&L attribution (cumulative)**:

- Coupon income:      +$734k  
- Bond MTM:           +$895k  
- Short leg:          −$324k  
- Gamma:              +$598k  
- Cash income:        +$690k  
- Borrow costs:       −$24k  
- Short rebate:       +$330k  
- Transaction costs:  −$833k  
- **Net P&L**:        +$2.08m  

---

## 6. Alpha / Beta vs Benchmarks (CAPM, excess returns)

Using **daily excess returns** (strategy − rf, factor − rf), where rf is the 1‑year risk‑free from `final_discount_rates_daily.csv`:

- **vs S&P 500 (single‑factor CAPM)**  
  - Alpha (annualized): ~**7.8%**  
  - Beta: ~**−0.07**  
  - Corr(strategy, S&P): ~**−0.27**

- **vs 60/40 portfolio (single‑factor CAPM)**  
  - Alpha (annualized): ~**7.9%**  
  - Beta: ~**−0.10**  
  - Corr(strategy, 60/40): ~**−0.26**

- **Two‑factor (S&P 500 + 60/40)**  
  - Alpha (annualized): ~**7.7%**  
  - Betas close to 0 (slightly negative vs S&P, near‑zero vs 60/40).

Interpretation: the strategy shows **low correlation and low (slightly negative) beta** to broad equity and balanced benchmarks, with a **meaningful positive excess‑return alpha** over this period.

---

## 7. How to Run

1. **Set up environment**

   ```bash
   pip install pandas numpy scipy statsmodels yfinance matplotlib
