# Commodity-Linked FX Underreaction Capstone Plan

**Project title:** Do commodity price shocks predict subsequent returns in commodity-linked currencies?

**Strategy type:** Momentum-style macro-informed predictive signal

**Core idea:**
Test whether large moves in relevant commodities are only gradually incorporated into the FX rates of commodity-linked currencies, creating a short-horizon predictive signal. The project focuses on **underreaction**, not overreaction: if a relevant commodity rises materially and the associated currency has not yet fully responded, the currency may continue appreciating over the next period.

---

# 1. Hypothesis Pitch

## What pattern or inefficiency you believe exists
Commodity-linked currencies such as **CAD**, **NOK**, and **AUD** may **underreact** to economically relevant commodity price shocks. For example, a sharp rise in oil may not be fully reflected immediately in **CAD/USD** or **USD/CAD**. As a result, recent commodity returns may help predict future FX returns over short horizons.

## Why you think it might be true
This should be grounded in an economic and microstructure story rather than pure pattern recognition.

### Economic mechanism
- Commodity price moves affect **terms of trade**, expected export revenues, fiscal outlook, and growth expectations for commodity-exporting economies.
- These macro effects should, in principle, support the domestic currency.

### Why the reaction may be delayed
- Cross-asset information may not be incorporated instantaneously.
- FX traders may initially focus more on broad dollar moves, rates, or risk sentiment than on country-specific commodity shocks.
- Macro information may diffuse gradually across markets.
- Market participants may require confirmation that a commodity move is persistent rather than transitory.

## What would prove you wrong
State this clearly before coding.

You should be willing to reject the hypothesis if:
- lagged commodity returns do **not** predict future FX returns out-of-sample;
- any apparent predictability disappears after basic transaction cost assumptions;
- results are driven by a single episode or one currency only;
- the signal fails when using a clean train/test split or walk-forward validation;
- the relationship is unstable across reasonable parameter choices.

## Recommended framing for your project
A good one-sentence hypothesis is:

> Recent returns in economically relevant commodities predict short-horizon returns in commodity-linked currencies because FX markets underreact to terms-of-trade information, and this effect should be strongest in liquid commodity-exporting developed-market currencies.

## Research to do before coding
- Read short summaries or practitioner commentary on:
  - commodity currencies;
  - terms of trade and exchange rates;
  - oil-CAD and oil-NOK relationships;
  - metals/AUD relationships.
- Find 2-4 credible references you can cite in the report.
- Do not overdo the literature review; just gather enough to defend the intuition.

## Work steps
1. Finalise the exact hypothesis wording.
2. Decide whether to use **daily** or **weekly** frequency.
3. Decide whether to model all currencies together or each separately.
4. Fix the forecast horizon in advance.

## Things to look out for
- Do not claim causality too strongly.
- Avoid saying “commodity prices determine FX.” Say they may contain predictive information.
- Keep the hypothesis narrow and testable.

---

# 2. Data Pipeline

## Recommended universe
Keep the universe intentionally small and economically motivated.

### FX series
Use 3 liquid developed-market commodity-linked currencies:
- **CAD/USD** or **USD/CAD**
- **NOK/USD** or **USD/NOK**
- **AUD/USD**

Choose quote conventions carefully and keep them consistent. If using **USD/CAD** and **USD/NOK**, rising values mean USD strength rather than domestic currency strength, so your signal sign must be adjusted.

### Commodity series
Use one relevant commodity proxy per currency:
- **CAD**: WTI crude oil
- **NOK**: Brent crude oil
- **AUD**: industrial metals proxy, copper proxy, or a broad metals ETF if iron ore is hard to obtain consistently

## Suggested sources
Accessible public sources are sufficient.

- **Yahoo Finance / yfinance** for FX and commodity ETFs/futures proxies
- **FRED** if you want supporting macro controls such as dollar index or interest rates
- Optional: central bank or official sources for contextual discussion, not necessarily for the main signal

## Data engineering plan

### Step 1: Download raw data
Download adjusted close or close series for all FX and commodity assets over a sufficiently long period, for example from around 2010 onward if available and reliable.

### Step 2: Standardise timestamps
- Convert dates to a common timezone-neutral date index.
- Use a common business-day calendar.
- Investigate missing values rather than silently dropping them.

### Step 3: Align market calendars
FX and commodities may have different holiday calendars and slightly different trading conventions.

You should:
- merge series on common dates;
- document how many observations are lost;
- avoid forward-filling returns;
- if filling prices for calendar alignment, do so cautiously and explain why.

### Step 4: Create return series
Use log returns or simple returns consistently.

For example:
- daily commodity returns;
- daily FX returns;
- later aggregate these into rolling multi-day signals if needed.

### Step 5: Build lagged features only
All predictors must be lagged so they would have been known at decision time.

Example features:
- trailing 5-day commodity return
- trailing 20-day commodity return
- trailing 5-day FX return
- trailing 20-day realised FX volatility
- optional broad USD move over recent days

### Step 6: Store cleaned snapshots
Save:
- raw downloaded files;
- cleaned aligned panel;
- final modelling dataset with features and labels.

This helps with reproducibility and submission requirements.

## Research to do here
- Check ticker quality and history length for each asset.
- Confirm whether the commodity proxy is sensible for the currency.
- Review whether adjusted or unadjusted prices are appropriate for your chosen instruments.

## Work steps
1. Select tickers and verify data coverage.
2. Download all series programmatically.
3. Clean and align timestamps.
4. Construct return series.
5. Create lagged features.
6. Save cleaned datasets to the data folder.
7. Write short markdown notes in the notebook documenting each pipeline choice.

## Things to look out for
- **Look-ahead bias** when using same-day closes to predict same-day FX returns.
- Quote inversion issues for FX pairs.
- Different market close times.
- Missing data and silent row drops.
- Revisions to manually downloaded macro data if used.

---

# 3. Strategy and Backtest

## Strategy classification relative to the rubric
This project is best described as a:

**Momentum-style macro-informed predictive signal**

It is not generic price momentum in the traded asset itself. Instead, it uses **commodity momentum as a predictor of future FX momentum**.

## Basic signal design
The cleanest beginner version is:
- if the relevant commodity has risen strongly over the lookback window, go long the linked currency for the next holding period;
- if the relevant commodity has fallen strongly, go short the linked currency.

## A practical implementation

### Option A: Per-currency standalone strategy
For each currency:
1. Compute trailing commodity return over a fixed lookback window, such as 20 trading days.
2. Standardise the signal using a rolling z-score.
3. If the z-score is above a threshold, take a long position in the currency.
4. If below a negative threshold, take a short position.
5. Hold for a fixed horizon, such as 5 or 20 trading days.

### Option B: Cross-sectional ranking across three currencies
At each rebalance date:
1. Compute each currency’s commodity signal.
2. Rank currencies by signal strength.
3. Go long the top one or two and short the bottom one or two.

This is elegant, but for your level, **Option A** is simpler and easier to defend.

## Recommended final strategy
I would recommend the following:

### Instruments
- AUD/USD, USD/CAD, USD/NOK or equivalent with sign adjusted carefully
- respective commodity proxies for each

### Frequency
- rebalance **weekly** to reduce noise and turnover

### Signal
- trailing **20-day commodity return**
- optionally normalised by trailing commodity volatility

### Trading rule
- long domestic currency when lagged commodity signal is strongly positive
- short domestic currency when lagged commodity signal is strongly negative
- otherwise stay flat

### Position sizing
Keep this simple:
- equal notional per trade, or
- inverse-volatility scaling using trailing FX realised volatility

### Benchmark comparisons
Compare the strategy against:
- buy-and-hold FX exposure if meaningful;
- a no-signal baseline;
- possibly naive FX momentum if you want one simple benchmark.

## Backtest design

### Train/test split
Use a proper out-of-sample design.

Example:
- **In-sample:** 2011-2018
- **Out-of-sample:** 2019-2026

Or use walk-forward validation:
- estimate thresholds and scaling choices using only past data;
- roll forward and test on the next block.

### Parameter tuning
Keep parameter search minimal.

For example, only test:
- lookback windows: 10, 20, 40 days
- holding periods: 5, 10, 20 days
- threshold levels: 0, 0.5, 1 standard deviations

Then pick one configuration based on in-sample logic and keep it fixed out-of-sample.

### Performance metrics
At minimum include:
- cumulative return
- annualised return
- annualised volatility
- Sharpe ratio
- maximum drawdown
- turnover

Useful additions:
- hit rate
- average gain/loss
- downside deviation or Sortino
- subperiod performance by regime

### Transaction costs
Even rough assumptions are important.

Use simple assumptions such as:
- fixed basis-point cost per rebalance in FX;
- higher cost for NOK than AUD if you want to show nuance;
- sensitivity table using low/base/high cost assumptions.

## Research to do here
- Decide whether weekly or monthly rebalancing is more defendable.
- Review how much turnover your chosen signal will create.
- Check whether predictive relationships differ by currency.

## Work steps
1. Define the exact signal mathematically.
2. Write a backtest function using only lagged information.
3. Add transaction costs.
4. Run in-sample tests.
5. Freeze the specification.
6. Evaluate out-of-sample.
7. Plot equity curves and rolling Sharpe or drawdown.
8. Summarise results in a small table near the top of the report.

## Things to look out for
- Avoid tuning too many windows.
- Ensure positions are entered after the signal is observed.
- If using weekly data, make sure the return horizon matches the rebalance convention.
- Do not cherry-pick the best-performing currency and then pretend that was the plan all along.

---

# 4. Methodology Defense

This section should justify every major design decision.

## How to defend your choices

### Why this universe?
- The universe is intentionally small and hypothesis-driven, not generic.
- CAD, NOK, and AUD are liquid developed-market currencies with widely discussed commodity exposure.
- This avoids survivorship bias issues common in stock universes.

### Why these commodities?
- Oil has a plausible macro link to CAD and NOK.
- Metals proxies are a practical approximation for Australia’s commodity exposure.
- You are choosing economically relevant drivers rather than arbitrary predictors.

### Why this frequency?
- Weekly or short-horizon trading reduces microstructure noise relative to daily while preserving the underreaction hypothesis.
- It also keeps turnover and implementation complexity manageable.

### Why this train/test split?
- The split ensures all tuning is done before evaluation.
- Out-of-sample performance is necessary because in-sample profitability may reflect noise.

### Why this signal construction?
- Trailing commodity returns are interpretable and observable in real time.
- A simple signal is more defendable and less vulnerable to overfitting than a complex model.

### Why these cost assumptions?
- FX is generally liquid, but trading is not free.
- Cost assumptions should be conservative enough to avoid overstating strategy viability.

## Research to do here
- Think through the logic of each choice before writing code.
- Be able to explain what would happen if a choice changed.

## Work steps
1. Write one paragraph defending each major design choice.
2. For each choice, write one sentence on an alternative you considered and why you rejected it.
3. Make sure your report language matches what your code actually does.

## Things to look out for
- Do not use sophisticated language to cover weak design.
- If a proxy is imperfect, say so plainly.
- Acknowledge what your setup does not capture, such as full terms-of-trade complexity.

---

# 5. Pitfall Checklist

Use the five required subsections exactly.

## Look-ahead bias

### How to check
- Verify all features are lagged relative to trade execution.
- If using close-to-close returns, ensure the signal at date t only drives trades from t+1 onward.
- Audit the code for any rolling-window computations that inadvertently include future observations.

### What to report
- State exactly when signals are observed and when positions are entered.
- Mention any time-zone or market-close mismatch issue and how you handled it.

### Things to look out for
- Same-day commodity return predicting same-day FX return is usually not permissible unless timing is explicitly justified.

## Survivorship bias

### How to check
- Explain that the universe consists of continuously traded G10-type FX pairs and commodity proxies rather than delisted equities.
- Acknowledge that while survivorship bias is much smaller here, proxy selection still matters.

### What to report
- State why survivorship bias is limited in this design.
- If you use ETFs rather than futures, note that instrument history may still constrain the sample.

### Things to look out for
- Do not say “not applicable” without explanation.

## Overfitting / data-snooping

### How to check
- Keep the parameter grid small.
- Record how many variations you tried.
- Select parameters using only in-sample results.
- Confirm that performance remains directionally similar under nearby settings.

### What to report
- Number of lookbacks, thresholds, and holding periods tested.
- Whether the selected strategy was chosen based on economic plausibility or only the best Sharpe.

### Things to look out for
- If the strategy only works for one exact window, be cautious.

## Transaction costs and slippage

### How to check
- Estimate turnover from rebalancing and position changes.
- Apply a per-trade cost assumption.
- Run sensitivity analysis.

### What to report
- Base-case transaction cost assumption and rationale.
- How results change under higher-cost assumptions.

### Things to look out for
- High turnover can destroy apparent profitability.
- NOK may be less liquid than major pairs such as EUR/USD.

## Out-of-sample validation

### How to check
- Split the sample before testing.
- Do not retune after seeing test results.
- Optionally use walk-forward validation.

### What to report
- Exact in-sample and out-of-sample dates.
- Comparison of key metrics across both periods.
- Whether the edge weakens, disappears, or remains stable.

### Things to look out for
- A weaker but still positive out-of-sample result is acceptable.
- A dramatic collapse should lead to a more cautious conclusion.

---

# 6. Efficiency Touchpoint

## Most computationally expensive step
In this project, the most expensive step is likely not extreme. It will probably be one of:
- rolling feature construction across multiple windows and assets;
- walk-forward backtesting across parameter configurations.

If you have:
- $$n$$ observations,
- $$a$$ assets,
- $$p$$ parameter combinations,

then a simple walk-forward evaluation is approximately:

$$
O(nap)
$$

This is fully acceptable at your scale because the asset universe is very small.

## How to discuss this in the report
You can say:
- the project avoids expensive pairwise asset computations by using a deliberately small universe;
- vectorised pandas/numpy operations reduce runtime;
- parameter search was intentionally restricted to avoid unnecessary computation and data-snooping.

## Research to do here
- Time your main backtest cell or function.
- Identify which part of the notebook takes longest.

## Work steps
1. Measure runtime for data preparation and backtest.
2. Note the dominant operation.
3. Write a short paragraph explaining complexity and why it is acceptable.

## Things to look out for
- Do not overstate technical sophistication here.
- Simplicity is fine as long as you understand it.

---

# 7. Risk Touchpoint

## Main risks of the strategy
This strategy is exposed to several real risks beyond standard backtest statistics.

### Macro regime risk
Commodity-currency relationships can weaken or break during periods dominated by:
- central bank surprises;
- broad USD moves;
- geopolitical shocks;
- global risk-off episodes.

### Correlation breakdown
A commodity may stop being a reliable short-horizon driver of the currency.
For example, oil may rise while CAD does not respond because rates or broader dollar strength dominate.

### Concentration risk
With only three currencies, the strategy is not highly diversified.
A large portion of performance may come from one currency or one macro episode.

### Volatility and gap risk
FX can reprice quickly after macro announcements. A signal based on gradual underreaction may fail in shock periods.

## What bad scenario could hurt the strategy most?
A plausible bad scenario is:
- commodity prices rally,
- but the USD also rallies sharply due to global risk aversion or rate repricing,
- causing commodity-linked currencies to fall despite positive commodity signals.

This would directly challenge the signal’s assumption that commodity information dominates near-term FX moves.

## Position sizing discussion
You should mention:
- whether you use equal weights or inverse-vol scaling;
- whether gross exposure is capped;
- whether you allow multiple simultaneous positions.

## Research to do here
- Look at strategy performance in stress periods.
- Examine whether losses cluster in strong USD regimes.

## Work steps
1. Compute drawdowns and worst periods.
2. Review whether losses align with obvious macro stress episodes.
3. Write a short paragraph linking these losses to the strategy’s structural weaknesses.

## Things to look out for
- Do not just say “risk is max drawdown.”
- Show that you understand why the strategy could fail economically.

---

# 8. Reflection

## What you may learn
Likely learnings include:
- an economically sensible relationship is not automatically tradable;
- signal timing and implementation details matter enormously;
- cross-asset predictive effects may be unstable across regimes;
- clean backtesting is harder than generating an attractive hypothesis.

## What might be wrong in your original assumptions
You may find that:
- commodity information is already incorporated too quickly to exploit at your horizon;
- the relationship exists only during some macro regimes;
- the effect is visible in raw correlations but too weak after costs;
- broad dollar moves dominate country-specific commodity effects.

## Natural next steps
If the project is promising, future extensions could include:
- adding a regime filter based on VIX or dollar strength;
- using interest-rate differentials as an additional conditioning variable;
- comparing linear signals with a simple ML classifier;
- extending the universe carefully to NZD or selected EM commodity exporters.

## Work steps
1. Keep notes while building the project.
2. Record what changed from your original idea.
3. Write the reflection after reviewing the full results, not before.

## Things to look out for
- The reflection should be honest rather than defensive.
- A negative result is acceptable if the test was fair.

---

# Practical Notebook Workflow

Although the report must follow the rubric headings, your notebook should likely proceed as follows:

## Notebook section order
1. **Data acquisition**
2. **Data cleaning and alignment**
3. **Exploratory analysis**
4. **Feature engineering**
5. **Signal construction**
6. **Backtest implementation**
7. **Performance evaluation**
8. **Pitfall checks**
9. **Risk analysis**

## What to do in exploratory analysis
Before backtesting, produce a few simple charts:
- price and return plots for each FX and commodity series;
- rolling correlations between commodity and FX returns;
- scatterplots of lagged commodity returns versus future FX returns;
- summary statistics by asset.

This will help you understand the data and support the hypothesis discussion.

---

# Suggested Minimal Final Specification

If you want the simplest defendable version of the whole project, I would use:

## Universe
- AUD/USD with copper or metals proxy
- USD/CAD with WTI crude
- USD/NOK with Brent crude

## Frequency
- weekly rebalance

## Signal
- trailing 20-day commodity return, lagged by one period

## Rule
- if signal is in top positive region, long domestic currency;
- if in bottom negative region, short domestic currency;
- otherwise flat.

## Position sizing
- equal weight across active positions

## Backtest
- in-sample parameter selection on early years
- fixed strategy evaluation out-of-sample
- include transaction costs and turnover

This version is simple enough to complete cleanly and sophisticated enough to satisfy the brief if written well.

---

# Final Recommendation

For your level, the most important thing is not making the model more complex. It is making the project:
- specific;
- economically motivated;
- cleanly implemented;
- explicitly out-of-sample;
- honest about risks and failure modes.

If you execute this carefully, it will look substantially better than a generic technical-analysis project and will fit your interest in macro, risk, and trading-oriented research.
