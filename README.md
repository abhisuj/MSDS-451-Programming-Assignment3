# MSDS 451 — Programming Assignment 3

Systematic trading strategies: Momentum, Mean Reversion, and a Regime-Switching allocator evaluated on US equities. This repo includes a Jupyter Notebook (`MSDS_451_Programming_Assignment_3.ipynb`) that builds and compares four strategies, generates tear sheets, and benchmarks against SPY/QQQ/IEF/BRK-A.

## Overview

Strategies implemented in the notebook:

- Monthly Momentum: rank assets by recent return and allocate to top performers.
- Dual Moving-Average (Momentum): go long when a short SMA crosses above a long SMA; otherwise reduce/exit exposure.
- Mean Reversion (Z-score): fade short-term price extremes relative to a rolling mean/std (go long oversold, short overbought) with entry/exit bands.
- Regime-Switching (Allocator): detect market states using a proxy index (Berkshire Hathaway, BRK.A) with volatility, drawdown, and breakout tests. Allocate to momentum in trending/stable periods, to mean reversion in calm mean-reverting periods, and cut risk in crisis.

Benchmarks and diagnostics:

- Benchmarks: SPY, QQQ, IEF, BRK-A (for comparison); BRK_A used as market proxy inside Zipline.
- Metrics: cumulative return, annual return/volatility, Sharpe, max drawdown, alpha/beta vs SPY.
- Plots: daily returns vs benchmarks and log-scale equity curves.

## Environment Setup (Windows)

You can run everything inside the provided notebook (cells install required packages), or set up the environment from a terminal.

Required Python packages:

- zipline-reloaded
- quandl
- pyfolio-reloaded
- yfinance
- empyrical
- pandas, numpy, matplotlib

PowerShell commands (optional if you run the first cell in the notebook):

```powershell
# 1) Create/activate your preferred environment (conda or venv recommended)
# conda create -n msds451 python=3.10 -y
# conda activate msds451

# 2) Install dependencies
pip install zipline-reloaded quandl pyfolio-reloaded yfinance empyrical matplotlib pandas numpy

# 3) Set your Quandl API key for this session
$env:QUANDL_API_KEY = "<YOUR_QUANDL_API_KEY>"

# 4) Ingest the Quandl bundle for Zipline (first time only)
zipline ingest -b quandl
```

Notes

- In the notebook, the first few cells use `%pip install` and set the Quandl key via Python `os.environ`—either path works.
- Zipline uses `BRK_A` (underscore) as the Berkshire symbol, while Yahoo Finance uses `BRK-A` (dash). The notebook handles both appropriately.
- If you encounter binary build issues on native extensions, a Conda environment is recommended. As a fallback, WSL can also work well with Zipline.

## Data

- Zipline data is sourced via the `quandl` bundle (ingested once, then cached locally).
- Benchmark prices (SPY, QQQ, IEF, BRK-A) are pulled via `yfinance` for summary and plotting.

## Performance and Data Limitations

- Keep the portfolio small (≈3 tickers) on local laptops. Zipline’s in-memory data handling and history windows can consume significant RAM and increase runtime as the universe grows. Reducing the universe materially improves stability and speed.
- Google Colab often crashes for this workflow. The combination of limited RAM, ephemeral storage (no persistent `zipline ingest` cache), and missing symbols in the Quandl bundle can terminate the runtime unexpectedly. Local Conda environments are more reliable.
- Quandl/WIKI bundle coverage ends in 2018. Historical equities data in the standard `quandl` bundle is not available beyond 2018, so the backtest end date is capped at 2018-12-31.
- Missing or delisted symbols: Some tickers may be absent or renamed in the bundle. Prefer liquid, long-lived symbols and verify with small test runs. The notebook uses `data.can_trade(asset)` guards, but preventing missing symbols up front reduces failures.

### Minimal universe example (3 tickers)

Paste this in the notebook where `tickers` is defined to keep runs fast and memory-light:

```python
# Keep the universe tiny on local machines to avoid long runtimes and OOMs
tickers = ['AAPL', 'WMT', 'XOM']

# Optional: sanity-check that the symbols exist in the ingested Quandl bundle
try:
	from zipline.api import symbol
	test_assets = [symbol(s) for s in tickers]
	print("Symbols OK in bundle:", [a.symbol for a in test_assets])
except Exception as e:
	print("Symbol lookup error in bundle:", e)
	print("Consider replacing missing tickers or reducing the universe.")
```

## How to Run the Notebook

1. Open `MSDS_451_Programming_Assignment_3.ipynb` in VS Code or Jupyter.
2. Run cells in order:
	 - Install packages and imports
	 - Set Quandl API key and run `zipline ingest -b quandl` (first run only)
	 - Configure dates, universe, and strategy definitions
	 - Execute the four backtests (momentum, dual MA, mean reversion, regime-switching)
	 - Generate PyFolio tear sheets and the comparison tables/plots
3. Review the “Strategy vs Benchmark Performance” table and the daily/log equity curve charts.

## Experimental Design

- Universe: a selected set of large-cap tickers (e.g., AAPL, WMT, XOM; expand as needed).
- Backtest window: 1998-01-01 to 2018-12-31; live/out-of-sample marker around 2006-02-27 for PyFolio reporting.
- Frequency: daily; capital base: 10,000 USD.
- Costs: Zipline commission/slippage models applied where specified.
- Regime proxy: BRK_A (Zipline symbol). Regime logic uses:
	- Realized volatility and drawdown thresholds to identify “crisis” (de-risk).
	- Stable volatility to favor mean reversion.
	- Breakout condition and lookback return to favor momentum.

## Results Summary and Findings

High-level findings from the experiment (qualitative, not point-estimates):

- Mean Reversion (Z-score): delivered steadier, modest returns with relatively low volatility and smaller drawdowns. Upside was limited during persistent trends. Sensitive to transaction costs due to higher turnover.
- Momentum (Monthly Ranking & Dual MA): produced stronger compounding in trending markets but experienced larger drawdowns and whipsaws around regime changes.
- Regime-Switching Allocator: more resilient in volatile periods. It cut exposure when the BRK_A proxy signaled stress (elevated realized volatility or large drawdown), leaned into mean reversion in calm mean-reverting conditions, and used momentum only when breakout strength and stability were present. Net effect: better risk-adjusted performance and fewer large equity drawdowns vs. a single pure sleeve.

What to use in volatile markets?

- Pure momentum can struggle in choppy volatility (frequent reversals), while mean reversion can help in range-bound spikes but risks fighting persistent moves. The regime-switching framework provided the best balance by throttling risk in crisis and selecting the appropriate sleeve for the regime.

Metrics monitored in the notebook:

- Cumulative and annual returns
- Annualized volatility
- Sharpe ratio
- Max drawdown
- Alpha and beta vs. SPY
- (Optional extensions) Turnover, cost-adjusted returns, and “regime hit rate” (how often the active sleeve outperforms its alternative)

## Reproducibility Tips

- Use the same start/end dates and live_start_date for consistent reporting.
- Keep symbol conventions straight: `BRK_A` (Zipline) vs `BRK-A` (Yahoo Finance).
- The notebook normalizes timezones for return series when combining sources.
- If `yfinance` throttles, re-run after a short delay or cache prices locally.

## Troubleshooting

- Quandl key not found: ensure `$env:QUANDL_API_KEY` is set (or set it inside the notebook via `os.environ`).
- Zipline ingest errors: run from an activated environment with `zipline-reloaded` installed; try `zipline bundles` to verify; re-run `zipline ingest -b quandl`.
- PyFolio errors: confirm the returns series is a pandas Series with a DatetimeIndex and no timezone, or use the provided `sanitize_series` helper.
- Windows build issues: prefer a Conda env; consider WSL if native builds fail.

## Next Steps

- Expand the universe and add sector/ETF proxies.
- Add robust transaction cost models and slippage stress tests.
- Perform walk-forward validation on regime thresholds and position sizing.
- Explore alternative risk proxies (e.g., VIX, credit spreads) and ensemble regime detectors.

## Use of AI Assistance

AI tools were used to assist with code drafting, documentation, and summarization in this assignment. All generated content was reviewed, edited, and validated by the author. No confidential or proprietary data were provided to AI services. Analytical conclusions and implementation choices remain the author’s responsibility.

## Important Disclosure (Not Financial Advice)

This project is for educational and research purposes only and does not constitute investment advice, an offer, or a solicitation to buy or sell any security or strategy. Backtested results are hypothetical, subject to hindsight bias and data/survivorship limitations, and may differ materially from real trading outcomes. Trading involves substantial risk, including the possible loss of principal. Past performance is not indicative of future results. Always conduct your own due diligence and consider consulting a qualified financial professional.

---

For questions or improvements, feel free to extend the notebook and update this README with your findings.