# Commodity-Linked FX Underreaction

**Do large commodity moves predict next-week returns in CAD, NOK and AUD?**

Capstone project for the Data Structures and Algorithms module, Imperial College Business School.
Author: Tsz Lok Pierson Chu.

## The idea

Commodity exporters' currencies should strengthen when their main export gets more valuable. If FX markets take
several days to fully price in a large commodity move, the currency should keep drifting in the same direction
over the following week, and that drift could be traded. The project tests this on three pairs:

| Currency | Linked commodity | Tickers (Yahoo Finance) |
|---|---|---|
| Canadian dollar | WTI crude oil | `USDCAD=X`, `CL=F` |
| Norwegian krone | Brent crude oil | `USDNOK=X`, `BZ=F` |
| Australian dollar | Copper | `AUDUSD=X`, `HG=F` |

The Invesco DB US Dollar Index Bullish Fund (`UUP`) is used as a control for broad US dollar moves.

## The strategy

For each currency separately:

1. Sum the commodity's daily log returns over the last 10, 20 or 40 trading days, using data only up to the day
   before the decision.
2. Convert that move into a z-score against the past 252 trading days.
3. Go long the currency if z is above a threshold (0, 0.5 or 1.0), short if below minus the threshold, flat otherwise.
4. Rebalance every Friday and hold for one week, at a fixed 1/3 weight per currency, with transaction costs of
   2bp (CAD, AUD) and 5bp (NOK) per position change.

All 9 lookback × threshold combinations were ranked on 2011–2018 (in-sample) only. The winner, a 40-day lookback with
a 1.0 threshold, was frozen and tested once on 2019–2026 (out-of-sample).

## Results

| Metric | In-sample (2011–2018) | Out-of-sample (2019–2026) |
|---|---|---|
| Cumulative return | 13.8% | −7.9% |
| Annualised return | 1.7% | −1.1% |
| Annualised volatility | 4.3% | 5.3% |
| Sharpe ratio | 0.38 | −0.20 |
| t-statistic (mean weekly return) | 1.12 | −0.48 |
| Max drawdown | −9.3% | −20.1% |

**The hypothesis is not supported.** Neither period is statistically distinguishable from zero, none of the 9
configurations made money out-of-sample, and the strategy loses money even with zero transaction costs. It also
underperforms simply holding the three currencies (out-of-sample Sharpe −0.07) and is barely better than plain FX
momentum (−0.24). Stripping the US dollar component out of the commodity signal does not rescue it.

The notebook also covers:
- data cleaning, including bad FX ticks and WTI's negative price in April 2020;
- a futures-roll check comparing WTI and Brent;
- a look-ahead corruption test;
- a one-day timestamp offset found in Yahoo's FX data;
- drawdown attribution;
- the cost of the parameter search.

## Repository contents

| Path | What it is |
|---|---|
| `notebook.ipynb` | The full pipeline: data acquisition → cleaning → signal → backtest → benchmarks → pitfall checks |
| `data/raw/` | Snapshot of the seven Yahoo Finance series as downloaded (2010-01-04 to 2026-09-14) |
| `data/processed/aligned_prices.csv` | The cleaned, calendar-aligned price panel |
| `capstone_hypothesis_pitch.docx` | The written report (working version) |
| `capstone_report.docx` | Earlier report draft |
| `capstone_study_guide.md` | Notes explaining every step of the project |
| `commodity_fx_underreaction_capstone_plan.md` | The original project plan |
| `capstone_project_rubric.pdf` | The assignment brief |

## Running it

Requires Python 3.12 with `numpy`, `pandas`, `matplotlib` and `yfinance`, plus Jupyter:

```bash
pip install numpy pandas matplotlib yfinance jupyter
jupyter notebook notebook.ipynb
```

`USE_SNAPSHOT = True` (the default) loads the files in `data/raw/`, so every number above is reproduced exactly
without network access. Setting it to `False` re-downloads any series whose snapshot is more than 5 days old and
overwrites the file in `data/raw/`, so results will then differ from those reported.
