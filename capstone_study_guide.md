# Capstone Study Guide: Commodity-Linked FX Underreaction

A guide to understanding, explaining and defending this capstone. It is written for someone who knows Python and industrial-engineering thinking (SPC, DOE, process validation) but has no background in finance or quant trading.

**How the documents fit together**

| File | What it is | How this guide refers to it |
|---|---|---|
| `capstone_report.docx` | The written report, in the rubric's 8 numbered sections | **Report §1–§8** |
| `notebook.ipynb` | The code, in named sections (Data Acquisition through Efficiency Touchpoint) | **Notebook *section name***, plus function names |
| `capstone_project_rubric.pdf` | What the professor grades against | **Rubric §1–§8** |
| `commodity_fx_underreaction_capstone_plan.md` | The original planning doc the project grew from | **Plan** |
| `capstone_hypothesis_pitch.docx` | Your own write-up of the report, in your words; the report now uses its headings | **Your draft** |

The report's sections are numbered (§1–§8). The notebook's sections used to be lettered A–I and now use names, so this guide refers to them by name, shortened as below. The change logs in Part 8 still use the old letters, because they describe the notebook as it was at the time.

| Old letter | Notebook heading | Short name in this guide |
|---|---|---|
| A | Data Acquisition | *Data Acquisition* |
| B | Data Cleaning and Alignment | *Data Cleaning* |
| C | Exploratory Analysis | *Exploratory Analysis* |
| D | Signal Construction | *Signal Construction* |
| E | Backtest and Results | *Backtest and Results* |
| F | Benchmarks and Transaction Cost Sensitivity | *Benchmarks* |
| G | Why the Strategy Failed: Risk and Attribution | *Why It Failed* |
| H | Pitfall Checks | *Pitfall Checks* |
| I | Efficiency Touchpoint | *Efficiency Touchpoint* |

**How the report is laid out.** The report follows the headings of your draft (`capstone_hypothesis_pitch.docx`). The numbered section titles and the five pitfall subheadings match the rubric exactly; the other subheadings are yours. When this guide quotes a label in brackets, such as Report §3 "Strategy" ("Validation design"), the first part is the subheading and the second is the label that opens the paragraph.

| Section | Subheading | What's under it | Covered in this guide |
|---|---|---|---|
| 1. Hypothesis Pitch | — | The pitch and 3 falsification criteria | Part 2 |
| 2. Data Pipeline | Universe and sources | Universe and sources, Sign convention, Outputs | 1.1–1.2, 3.1 |
| | Real-world messiness | Cleaning and calendar alignment; bad ticks and the negative WTI price; Futures rolls | 3.2, 6.3, 6.4 |
| 3. Strategy and Backtest | Strategy | Feature construction, Strategy design, Validation design, the 9-configuration table | 3.4, 4.1 |
| | Results | Results table and notes, interpretation, where the P&L came from | 4.2, 4.3, 4.7 |
| | Backtest | Benchmarks, transaction cost sensitivity, falsification criterion 3 | 4.4–4.6 |
| 4. Methodology Defense | Universe | Why these three currencies | Part 7 |
| | Train/Test Split | Why the 2019 split, not walk-forward | Part 7, 6.7 |
| | Transaction Cost Assumptions | Costs; "Why spot returns, without carry" | Part 7, 4.5 |
| | Signal Construction Reasoning | Signal; "Why a one-week holding period"; "Why this position sizing"; "What would break" | Part 7, 6.2 |
| 5. Pitfall Checklist | Look-ahead bias, Survivorship bias, Overfitting/data-snooping, Transaction costs and slippage, Out-of-sample validation | One answer each | Part 6 |
| 6. Efficiency Touchpoint | — | Grid search cost | 6.8 |
| 7. Risk Touchpoint | — | Leverage, concentration, the drawdown | Part 5 |
| 8. Reflection | — | Yours to write | 8.6 (material and next steps) |

---

## Part 0: The project in 30 seconds and in 2 minutes

**30 seconds.** Countries like Canada, Norway and Australia earn much of their income by exporting commodities: oil, and metals such as copper. When those commodity prices jump, the country's currency should strengthen. My hypothesis was that currency traders take several days to fully price in a big commodity move, so a big move today would predict the currency drifting the same way next week. I built a trading strategy to exploit that, tuned it on 2011–2018, and tested it once on 2019–2026. It made a small profit in the tuning period and lost money in the test period, but neither result is statistically distinguishable from luck, and each period's result came mostly from one or two macro shocks. The loss is not explained by trading costs, and all 9 parameter settings I tried failed out-of-sample. My conclusion is that there is no reliable evidence of the effect, and certainly not at a tradeable level.

**2 minutes (add these points).**
- *Mechanism:* higher export prices improve a country's "terms of trade", which means more export revenue, a better government budget and stronger growth expectations. All of that supports the currency. The *delay* comes from limited attention: FX traders mostly watch the US dollar, interest rates and risk sentiment, so commodity-specific news spreads into the currency gradually.
- *Design:* each currency is traded on its own. The signal is a z-score of the linked commodity's recent return. The position is held for one week, a horizon fixed in advance by the hypothesis's own wording. Only 2 parameters were searched (a 3×3 grid, 9 combinations), costs were included during tuning, and there was a single train/test split.
- *Trust:* a corruption test proves the code never uses future data. Bad data ticks are filtered, and the negative oil price in April 2020 is handled explicitly. Two benchmarks show the commodity signal adds nothing over simpler alternatives. A direct test shows that dollar-adjusting the signal doesn't rescue it.
- *Honest caveats:* CAD and NOK are both oil bets (their positions are 0.82 correlated), so this is closer to 2 independent tests than 3. And the in-sample "edge" was never significant (t ≈ 1.1), so the right summary is "no evidence in either period", not "an edge that collapsed".

---

## Part 1: The finance you need (and nothing more)

### 1.1 Currencies and how they are quoted

An exchange rate is the price of one currency in another. The quote `USDCAD = 1.35` means **1 US dollar buys 1.35 Canadian dollars**.

- If `USDCAD` **rises** to 1.40, one USD now buys more CAD. The **USD strengthened and CAD weakened**.
- `USDNOK` works the same way (Norwegian krone).
- `AUDUSD = 0.65` is quoted the other way round: 1 Australian dollar buys 0.65 USD. If it **rises**, the **AUD strengthened**.

This inconsistency is a classic source of bugs, and it is why the notebook has a `flip_fx_sign` flag in the `UNIVERSE` dict (Notebook *Data Acquisition*) and negates CAD and NOK returns (Notebook *Data Cleaning*, the "Construction of 3 tables" cell). After the flip, **a positive number always means the local currency got stronger against the USD**. Report §2 "Universe and sources" ("Sign convention") states the same thing.

In FX you are always long one currency and short another. "Long CAD" here means *long CAD, short USD*: you profit if CAD strengthens against the dollar.

### 1.2 Commodity currencies and terms of trade

- **Terms of trade** = (price of what a country exports) / (price of what it imports). If oil rises and Canada exports a lot of oil, Canada's terms of trade improve.
- **Commodity currency** is market slang for a currency whose value tends to move with the country's main exports. CAD, NOK and AUD are the textbook examples.
- The pairs used:
  - **CAD ↔ WTI crude** (`CL=F`). WTI is the North American oil benchmark, and crude is Canada's largest export.
  - **NOK ↔ Brent crude** (`BZ=F`). Brent is the North Sea benchmark, and Norway produces North Sea oil.
  - **AUD ↔ copper** (`HG=F`). This is an *imperfect* proxy. Australia's biggest exports are iron ore, coal and LNG, but iron ore has no clean, liquid futures series on yfinance. Report §1 and §4 both say this openly.

### 1.3 Futures and "front-month continuous" series

Oil and copper prices here come from **futures**: contracts to buy or sell the commodity at a set future date. Each contract expires, so data vendors stitch the nearest-to-expiry ("front-month") contracts together into one continuous series. The stitch point is called a **roll**.

Yahoo's `CL=F`, `BZ=F` and `HG=F` are **not back-adjusted**. On a roll day the series simply jumps from the old contract's price to the new one's, and that **roll gap** shows up as a one-day "return" nobody could have earned. (An earlier version of the report wrongly called these series "already roll-adjusted". The −$37.63 print proves otherwise: it is the price of the expiring May 2020 contract itself.) A roll gap is a permanent **step**, not a spike-then-revert, so the median filter can't see it. Notebook *Pitfall Checks* therefore has a separate roll check that compares WTI with Brent (see 6.3).

In **April 2020**, with storage full during COVID, the expiring WTI contract traded at **−$37.63**: sellers paid buyers to take the oil. Notebook *Data Cleaning* detects this and masks it, because `log(negative)` is undefined.

### 1.4 Returns, log returns and basis points

- **Simple return:** `P_t / P_{t-1} − 1`.
- **Log return:** `ln(P_t / P_{t-1})`. For small moves it is almost the same as the simple return, and it has one very convenient property: **log returns add up over time**. A 5-day return is the sum of five daily log returns. This is why `trailing_return()` is just `returns.rolling(window).sum()`.
- **Basis point (bp):** 0.01%. A 2 bp cost on a $1,000,000 trade is $200.

### 1.5 Long, short, flat and position

- **Long (+1):** you gain if the price goes up.
- **Short (−1):** you gain if it goes down.
- **Flat (0):** no position and no exposure.

`positions_from_zscore()` (Notebook *Signal Construction*) outputs exactly these three values.

### 1.6 Performance metrics (with IE analogies)

| Metric | Meaning | IE analogy | Where computed |
|---|---|---|---|
| **Cumulative return** | Total growth over the period (`(1+r).prod() − 1`) | Total yield over the run | `performance_metrics()`, Notebook *Backtest and Results* |
| **Annualised return** | Cumulative return converted to a per-year rate | Throughput normalised per year | same |
| **Annualised volatility** | Standard deviation of weekly returns × √52 | Process σ | same |
| **Sharpe ratio** | Annualised return ÷ annualised volatility | **Signal-to-noise ratio**: reward per unit of variability | same |
| **Max drawdown** | Worst peak-to-trough fall of the equity curve | Worst cumulative deficit before recovery | same |
| **Hit rate** | Share of weeks with a positive return | Yield / first-pass rate | same (see the caveat below) |
| **t-statistic** | Mean weekly return ÷ its standard error. Below about 2 in absolute value means "could easily be zero" | A hypothesis test on whether a process shift is real | `mean_return_tstat()`, Notebook *Backtest and Results* significance check |

**How Sharpe is defined here:** `performance_metrics()` uses the *compounded* annual return ÷ annualised volatility, with no risk-free rate subtracted. That is normal for FX strategies, because the long and short currency legs finance each other. The more common arithmetic version (mean weekly return × 52 ÷ volatility) gives 0.40 / −0.17 instead of 0.38 / −0.20. The report states the definition, in the note under the Report §3 results table.

**Calibrating Sharpe:** about 0.3–0.5 is roughly what a broad stock index has earned over the long run. Above 1 is considered strong for a single strategy. Anything negative means the strategy lost money. The in-sample 0.38 is "modest"; the out-of-sample −0.20 is "loses money".

**Sharpe vs significance (the rule of thumb to remember):** t ≈ Sharpe × √(years). Eight years at Sharpe 0.38 gives t ≈ 1.1, which is **not significant**. To reach t = 2 at that Sharpe you would need about 28 years of data. That is why the in-sample result is weak evidence even before the out-of-sample test.

> [!warning] Hit-rate caveat you should be ready to explain
> `hit_rate = (r > 0).mean()` counts **every** week, including weeks where all three currencies were flat and the return was exactly 0. Those weeks count as "misses". With a 1.0σ threshold the portfolio was completely flat in **34.1% of in-sample weeks** and **41.7% of out-of-sample weeks**. That is why the reported hit rate (37.7% IS, 30.3% OOS) looks so low for a strategy that made money in-sample. Counting only weeks with at least one open position, the hit rate is **57.2% IS and 51.9% OOS**. These figures are printed by the cell right after `results_table` in Notebook *Backtest and Results*, and the report states them in a note under the §3 results table. Buy-and-hold is never flat, which is why its hit rate (49.4%) looks so much higher.

### 1.7 Backtesting vocabulary

- **Backtest:** simulate a trading rule on historical data as if you had traded it at the time.
- **In-sample (IS):** the data you are allowed to look at while designing and tuning (here 2011–2018). In IE terms, the pilot run you tune the process on.
- **Out-of-sample (OOS):** data held back and used **once**, after everything is frozen (2019–2026). In IE terms, the confirmation run. If the effect does not show up in the confirmation run, your tuning found noise.
- **Look-ahead bias:** using information you could not have had at decision time. In IE terms, deciding at the start of a shift using inspection results from the end of that shift.
- **Survivorship bias:** studying only things that survived, such as today's S&P 500 members, and missing the companies that went bankrupt. In IE terms, estimating machine reliability only from machines still in service.
- **Overfitting / data-snooping:** trying many variants and keeping the best one, so the "best" is partly luck.
- **Transaction costs:** every trade pays a spread or fee. In IE terms, a **changeover cost**: you pay each time you switch position, so a rule that switches often has to earn more to break even.
- **Benchmark:** a simpler alternative you must beat for the idea to add value.

---

## Part 2: The thesis, and what informed it

### 2.1 The hypothesis as written (Report §1)

> Large moves in a commodity-exporting country's key export commodity predict appreciation in that country's currency over the following week, because FX markets incorporate terms-of-trade information gradually rather than instantly.

The rubric (Rubric §1) asks for three things. Here is how the pitch answers each:

| Rubric asks | Your answer (Report §1) |
|---|---|
| **What** pattern | A large commodity move predicts the linked currency drifting the same way over the next week |
| **Why** it might be true | *Economic:* export revenue, fiscal outlook and growth expectations improve, which supports the currency. *Behavioural:* limited attention and confirmation-seeking make repricing spread over several days |
| **What would prove you wrong** | 3 falsification criteria (see 2.3) |

Two refinements make the pitch sharper than a generic idea:
1. **"Only in large moves, not small ones."** Small commodity wiggles are noise; big ones are news that takes time to absorb. This is why the strategy has a z-score *threshold*: it trades only when the move is unusually large. In SPC terms, act only on points outside the control limits.
2. **An honest prior.** The pitch predicts in advance that any edge will be small and may not survive costs, because FX and commodities are some of the most heavily watched markets in the world. That turned out to be right, and saying so beforehand makes the null result credible rather than an excuse.

### 2.2 The logic chain

```
Commodity price jumps (e.g. oil +15% in a month)
        │
        ▼
Exporter's terms of trade improve
(more export revenue, better fiscal position, stronger growth outlook)
        │
        ▼
Fair value of the currency rises  ◄── this part is well established
        │
        ▼
BUT FX traders are focused on USD / rates / risk sentiment,
and want confirmation the move is persistent
        │
        ▼
Currency moves toward fair value over several days, not instantly  ◄── THIS is the hypothesis
        │
        ▼
So: after a big commodity move, go long/short the currency for a week
and capture the remaining drift
```

The key thing to tell a professor: **it is well established that commodity prices and commodity currencies move together at the same time. The hypothesis is only about the delay.** If the currency reacts on the same day, no one can profit from it, because by the time you see the commodity move the currency has already moved.

### 2.3 Falsification criteria, and what happened to each

Report §1 lists three. This table shows where each was actually tested. Be ready for a professor to walk down this list.

| # | Criterion | Tested where | Outcome |
|---|---|---|---|
| 1 | Lagged commodity moves don't predict next-week FX out-of-sample | Notebook *Backtest and Results* `results_table`; *Pitfall Checks* grid table | **Triggered.** OOS Sharpe −0.20, and 0 of 9 configurations were positive OOS |
| 2 | Profits disappear after transaction costs | Notebook *Benchmarks* cost sensitivity | **Worse than triggered.** Even with zero costs, OOS Sharpe is −0.14. There was no profit for costs to remove |
| 3 | Predictability disappears once the signal is dollar-adjusted (just the USD cycle) | Notebook *Why It Failed*, the `dollar_adjusted()` cell | **Inconclusive.** The adjusted signal behaves much like the raw one (IS Sharpe 0.38 vs 0.46; both fail OOS). But "predictability disappears" can only be judged if there was significant predictability to start with, and there wasn't (IS t ≈ 1.1). What it does show: dollar-adjusting doesn't rescue the strategy (see 4.6) |

So criteria 1 and 2 reject the hypothesis, and criterion 3 can't be decided because there was no significant effect to explain. The story: **the effect isn't there, and stripping out the dollar doesn't bring it back.**

> [!note] A criterion that was dropped
> The pitch originally had a fourth criterion, numbered 3: "the reverse relationship (FX predicts commodities) is stronger". It was tested with weekly information coefficients in both directions and was never triggered, because neither direction predicted the other. It was removed in the simplification round (8.4) as the least informative test, and the old criterion 4 (dollar) became criterion 3. Be ready to say this if asked; see the Part 9 question "Maybe FX predicts commodities instead?"

### 2.4 Where the idea came from and how it evolved

The Plan file shows the starting point: "commodity momentum as a predictor of future FX momentum" (Plan §3). That is *not* ordinary price momentum in the asset itself, which is what separates it from the "generic, reflexive ideas" the rubric warns against (Rubric §1). A cross-asset, economically motivated link is the novel angle.

Decisions where the final version **departed from the plan**. These are good evidence that the thinking is yours:

| Plan suggested | Final project | Why (and where it is argued) |
|---|---|---|
| Search holding periods 5/10/20 days | **Fixed at 1 week**, not searched | The hypothesis says "over the following week", so searching the horizon would be data-snooping (Report §4 "Signal Construction Reasoning" ("Why a one-week holding period")) |
| 20-day lookback | Lookback **searched** over {10, 20, 40} | Kept the search small; 40 won in-sample |
| Walk-forward *or* fixed split | **Fixed split** | Walk-forward re-opens parameter selection at every refit, which multiplies the implicit variations tried (Report §4) |
| Equal-weight *or* inverse-vol sizing | **Fixed 1/3 weight** | Simpler, and one fewer moving part to justify (Report §4) |
| Falsification: "driven by one episode or one currency" | Replaced with the USD-cycle criterion (a reverse-causality criterion was also added, then dropped in 8.4) | More specific to the economic story. (It is now also *checked*: the per-year P&L table in Notebook *Backtest and Results* shows the result isn't driven by one currency, but it *is* driven by one or two episodes. See 4.3) |

> [!tip] Prepare your personal angle
> The rubric says the idea must be "genuinely yours" and is assessed "via the quality and specificity of your 'why'" (Rubric, Deliverables checklist). Have one or two sentences ready on *why this question interested you*. Only you can supply that, and it is worth rehearsing.

### 2.5 Literature that matches the reasoning (not currently cited)

The report has **no References section**, and a professor asking "what informed your thesis?" may expect some grounding. The papers below are well known and line up closely with the argument. **I'm listing these from memory, so read each abstract before citing, and only cite what you have actually looked at.**

*The economic link (commodity prices ↔ currencies):*
- Chen, Y.-C. & Rogoff, K. (2003). "Commodity currencies." *Journal of International Economics*, 60(1). Shows that AUD, CAD and NZD real exchange rates track world prices of their commodity exports.
- Cashin, P., Céspedes, L. F. & Sahay, R. (2004). "Commodity currencies and the real exchange rate." *Journal of Development Economics*, 75(1). The same link across many commodity-exporting countries.

*The behavioural mechanism (gradual information diffusion / underreaction):*
- Hong, H. & Stein, J. C. (1999). "A unified theory of underreaction, momentum trading, and overreaction in asset markets." *Journal of Finance*, 54(6). The theory that information spreads slowly across investors, which produces drift.
- Hong, H., Torous, W. & Valkanov, R. (2007). "Do industries lead stock markets?" *Journal of Financial Economics*, 83(2). Cross-market diffusion: returns in one market predict another with a lag.
- Driesprong, G., Jacobsen, B. & Maat, B. (2008). "Striking oil: Another puzzle?" *Journal of Financial Economics*, 89(2). Oil price changes predict *stock* returns with a lag, attributed to underreaction. This is the closest analogue to your idea, applied to equities instead of FX.

*Why a null result was always likely (use these in the Reflection):*
- Ferraro, D., Rogoff, K. & Rossi, B. (2015). "Can oil prices forecast exchange rates?" *Journal of International Money and Finance*, 54. Finds a strong **same-day** oil→CAD relationship but little **lagged** predictive power, which is essentially your result.
- Chen, Y.-C., Rogoff, K. & Rossi, B. (2010). "Can exchange rates forecast commodity prices?" *Quarterly Journal of Economics*, 125(3). Finds the **reverse** direction: commodity currencies predict commodity prices. This was your original reverse-causality criterion, since dropped (2.3). It's still the paper to cite if a professor asks whether causality might run the other way.

A strong way to frame it to a professor: *"The contemporaneous link is well documented. Underreaction to cross-asset information has been found in equities. I tested whether it carries over to FX at a weekly horizon, and consistent with Ferraro, Rogoff & Rossi, it doesn't survive out-of-sample."*

---

## Part 3: Walking through the pipeline (Notebook *Data Acquisition* to *Signal Construction*, Report §2)

Think of this as a production line: **raw material (prices) → inspection/cleaning → feature machining (signals) → assembly (positions) → QC (backtest and pitfall checks).**

### 3.1 Notebook: Data Acquisition

**What the code does:**
- `UNIVERSE` maps each currency to its FX ticker, its commodity ticker and whether to flip the sign.
- `DOLLAR_INDEX_TICKER = "UUP"` is an ETF that rises when the USD strengthens against a basket of major currencies. It is the **control variable** for "is this just the dollar?"
- `download_or_load_raw()` **caches** each series to `data/raw/*.csv`. With `USE_SNAPSHOT = True` (the default) it **always** loads the submitted snapshot when one exists. With `USE_SNAPSHOT = False` it re-downloads any snapshot more than 5 days stale (`STALE_AFTER_DAYS`).

**Why:**
- Reproducibility. The rubric requires a data snapshot (Rubric §2), and a grader who re-runs the notebook must get the same numbers as the report. Before `USE_SNAPSHOT` was added, re-running more than 5 days after the snapshot would silently re-download everything and change every result.
- The staleness check (now opt-in) fixed a real bug. Its docstring explains that old cached files had silently stopped months short while fresh downloads ran to the present, creating a spurious gap. That is a good "real-world messiness" story to tell. The date-range printout in the next cell confirms all snapshots end together.

**Output to know:** about 4,200 rows per series, from 2010-01-04 to 2026-09-14.

### 3.2 Notebook: Data Cleaning and Alignment

**Step 1: Calendar alignment.**
```python
calendar = prices[commodity_tickers].dropna(how="all").index
aligned = prices.reindex(calendar)
```
FX trades almost around the clock, but futures exchanges close on holidays. The panel is anchored to **days when at least one commodity traded**, because a commodity move is only *observable* on those days. Strictly, it's the **union** of the three commodity calendars (`how="all"`): a day when only Brent traded is kept, and WTI just gets forward-filled for it. It's "restrictive" only compared with FX's almost continuous calendar.

**Step 2: Bounded forward-fill.**
```python
aligned = aligned.ffill(limit=2)
```
Missing values before the fill (notebook output): UUP 4, USDCAD 8, USDNOK 6, AUDUSD 7, WTI 1, **Brent 32**, copper 1. Brent's 32 are scattered vendor gaps between 2010 and early 2014, none longer than 2 consecutive trading days. They are not a holiday pattern. The fill is capped at 2 days so that an isolated gap gets bridged, while a real multi-day outage would stay visible. After the fill, **0 rows were dropped**. The printout proves nothing was silently discarded, which is exactly what Rubric §2 asks for.

**Step 3: FX bad-tick removal (`detect_and_despike`).**
A 3-day centred **median filter**: flag a price if it deviates more than 5% from the median of itself and its two neighbours, then replace it with that median. In IE terms, this is a glitch filter on a sensor signal.
- A **spike-then-revert** (bad print) differs a lot from its neighbours' median, so it gets caught.
- A **real sustained move** stays close to its new neighbour, so the median follows it and it is left alone.
- It found 2 bad ticks, both in USDNOK: **2020-03-20** (7.735, replaced with the local median 10.779) and **2021-12-27** (7.999, replaced with 8.830).
- The same function is reused in Notebook *Pitfall Checks* on commodities at a 15% threshold, because commodities genuinely move more per day.

**Step 4: The negative oil price.**
`aligned.where(aligned > 0)` masks WTI's −37.63 on 2020-04-20 to NaN *before* taking logs. That removes **two** log returns, 20 and 21 April, and the cell now prints both dates.

> [!important] The knock-on effect you should be able to explain
> Pandas rolling windows return NaN if *any* value in the window is missing. So the 40-day trailing sum is NaN for 40 days after the gap, and the 252-day z-score built on it is NaN for about 252 days after that. **CAD therefore has no signal, and no position, for 59 of the 403 out-of-sample weeks (May 2020 to June 2021).** One bad price silently removed a currency for over a year. Notebook *Pitfall Checks* ("Data handling: the negative WTI print") re-runs everything with the bad price *bridged* instead (17 April's price carried forward one day). In-sample is unchanged, the frozen configuration's OOS Sharpe moves from −0.20 to −0.17, and all 9 configurations are still negative OOS. The headline keeps the masked version because it doesn't substitute a price, and the check shows the choice doesn't change the conclusion.

**Step 5: Log returns and sign flip.** This builds three frames: `domestic_fx_returns`, `commodity_returns` and `dollar_index_return`. The `describe()` output shows mean daily FX returns are slightly negative for all three currencies, meaning they drifted weaker against the USD over 2010–2026. That foreshadows the results.

**Step 6: The one-day lag (the most important convention in the project).**
```python
def trailing_return(returns, window):
    return returns.rolling(window).sum()
# always used as: trailing_return(r, w).shift(1)
```
`.shift(1)` means the value on row *t* is the trailing sum ending on *t−1*. Anything used on day *t* was known **before day t started**. The demo table in the notebook shows the "lagged_feature_used_on_t" column equals the row *above's* trailing sum. It is defined in **one** place so there is only one lag to get right.

### 3.3 Notebook: Exploratory Analysis

Three charts:
1. **Price paths:** commodity (orange) vs FX (blue) per currency, as a sanity check.
2. **60-day rolling correlation of same-day returns:** checks whether the link is stable. It starts around 0.6–0.7 in 2010, then drops to around ±0.2. The markdown cell after the chart now explains that the drop is a **data-timestamp effect**, not an economic change. The check itself is in Notebook *Pitfall Checks* (see 6.2 below).
3. **Scatter of lagged 20-day commodity return vs next 5-day FX return:** a first visual test of the hypothesis. The clouds are essentially round blobs with no obvious slope, which is an early hint that the signal is weak.

### 3.4 Notebook: Signal Construction

**The signal, step by step** (`zscore_signal()`):
1. `raw_lagged = trailing_return(commodity_returns, lookback).shift(1)`: the commodity's return over the last `lookback` days, lagged.
2. Standardise it against its own trailing **252-day** (about 1 trading year) mean and standard deviation to get a **z-score**.
3. `positions_from_zscore()`: **long** if z > threshold, **short** if z < −threshold, otherwise **flat**.

**SPC analogy:** the 252-day window is the baseline for estimating the process mean and σ. The threshold sets the control limits (0σ, 0.5σ or 1σ). The strategy only acts on out-of-control points, meaning unusually large commodity moves *for that commodity's recent regime*. Oil at 40% annual volatility and copper at 20% each get their own yardstick, so "large" means the same thing for both.

> [!note] A subtlety a sharp professor might probe
> The z-score subtracts the **trailing mean** of the 40-day return. If oil has been rising steadily all year, a normal positive 40-day return has z ≈ 0, so no trade. The signal therefore responds to moves that are large *relative to the recent trend*, not just large in absolute terms. That is consistent with "large moves are news", but you should be able to say it.

**Parameters (declared before seeing any result):**
```python
STANDARDISATION_WINDOW = 252
LOOKBACK_GRID  = [10, 20, 40]     # days of commodity return
THRESHOLD_GRID = [0.0, 0.5, 1.0]  # z-score entry cutoff
POSITION_WEIGHT = 1/3
IN_SAMPLE_END = "2018-12-31";  OOS_START = "2019-01-01"
BASE_COST_BPS = {"CAD": 2.0, "AUD": 2.0, "NOK": 5.0}
```
In DOE terms, this is a **3×3 full factorial on two factors**.

**Why the in-sample period effectively starts in March 2011:** the signal needs `lookback` days to form the trailing return, then 252 more to form the z-score. With the 40-day lookback, that is about 292 trading days of warm-up, so the first tradeable week is **2011-03-11**. That is why Report §3 says "2011–2018" even though the data starts in 2010.

**Weekly rebalance mechanics** (`weekly_backtest()`):
```python
weekly_position = position_daily.resample("W-FRI").last()   # signal as of Friday
weekly_position_active = weekly_position.shift(1)           # held during NEXT week
weekly_fx_return = domestic_fx_returns[ccy].resample("W-FRI").sum()
gross = weekly_position_active * weekly_fx_return * POSITION_WEIGHT
turnover = weekly_position_active.diff().abs()               # 0, 1 (open/close) or 2 (flip long↔short)
cost = turnover * POSITION_WEIGHT * cost_bps / 1e4
```

Timeline for one decision:

```
        Week W                                   Week W+1
Mon  Tue  Wed  Thu  Fri          Mon  Tue  Wed  Thu  Fri
                 │    │           └────────── position held ──────────┘
                 │    └─ the Friday row's signal is read here
                 └─ ...but because of .shift(1), that signal only
                    contains commodity data through THURSDAY
```

So the signal is fully known before week W+1 starts. Report §5 ("Look-ahead bias") calls this the "second one-week shift". The skipped Friday turns out to matter a lot, because the FX data is stamped about a day late (see 6.2).

**Portfolio return:** the sum across the 3 currencies of position × weekly FX log return × 1/3, minus costs. Summing log returns across assets is an approximation, but it is negligible at weekly FX move sizes. The notebook no longer says this, so be ready to say it if asked.

**Two assumptions to be ready to explain** (the notebook no longer describes them; carry is covered in Report §4):
- **Spot returns only, no carry.** Holding a currency also earns or pays the interest-rate difference between the two countries. The backtest ignores it. Because the strategy is long and short about equally often, carry largely nets out for *it*, but it matters for the always-long buy-and-hold benchmark. Report §4 covers it under "Transaction Cost Assumptions" ("Why spot returns, without carry").
- **A cost edge case.** `turnover = position.diff().abs()` gives NaN, filled to 0, when a currency moves between "no signal" (NaN) and a position. So the first entry, and CAD's exit and re-entry around the masked WTI print, are never charged. That's at most a handful of trades, worth well under 0.01%, so it is documented rather than fixed.

---

## Part 4: Results (Notebook *Backtest and Results* to *Benchmarks*, Report §3)

### 4.1 The grid search (Notebook *Backtest and Results*, the cell that prints "9 configurations evaluated")

All 9 combinations run **once**, and results are stored in `grid_results`. Selection reads **only** `is_sharpe`:

| lookback | threshold | IS Sharpe | IS cum. return |
|---|---|---|---|
| **40** | **1.0** | **0.381** | **13.8%** |
| 40 | 0.0 | 0.370 | 22.0% |
| 40 | 0.5 | 0.355 | 17.8% |
| 10 | 1.0 | −0.022 | −0.6% |
| 20 | 1.0 | −0.313 | −9.5% |
| 20 | 0.0 | −0.366 | −17.2% |
| 20 | 0.5 | −0.471 | −18.0% |
| 10 | 0.5 | −0.494 | −17.0% |
| 10 | 0.0 | −0.552 | −24.2% |

**How to read it:** even in-sample, only the 40-day lookbacks made money. The threshold barely mattered at 40 days (0.35–0.38). DOE reading: the lookback factor dominates and the threshold effect is small. This is weak evidence *against* the "only large moves" part of the pitch, because trading every move (threshold 0) did about as well in-sample as trading only 1σ moves. Out-of-sample the 1.0 threshold was the least bad of the three 40-day settings (−0.20 vs −0.26 and −0.48), which is mild support. Say both.

Selection used **net-of-cost** Sharpe on purpose (Report §3 "Strategy" ("Validation design")): ranking on gross returns could pick a configuration that looks good only because it trades a lot.

### 4.2 Headline results (Report §3 "Results" table = Notebook *Backtest and Results* `results_table`)

| Metric | In-sample 2011–2018 | Out-of-sample 2019–2026 |
|---|---|---|
| Cumulative return | 13.8% | **−7.9%** |
| Annualised return | 1.7% | −1.1% |
| Annualised volatility | 4.3% | 5.3% |
| Sharpe | 0.38 | **−0.20** |
| t-statistic (mean weekly return) | 1.12 | −0.48 |
| Max drawdown | −9.3% | **−20.1%** |
| Hit rate (all weeks) | 37.7% | 30.3% |
| Weeks | 408 | 403 |

The t-statistic row is new (Notebook *Backtest and Results* significance check; also in the Report §3 table). See 4.7 for what it means.

### 4.3 The equity curve and where the P&L came from (Notebook *Backtest and Results*, chart plus the new `yearly_pnl` table)

The per-year and per-currency numbers below are now **printed in the notebook** (the per-year table cell right after the equity curve) and quoted in Report §3 "Results", so you can cite them.

- **In-sample:** slow grind up in 2011–2013, then a **big jump in late 2014 to early 2015**. That is the oil crash (WTI roughly halved between mid-2014 and early 2015). The strategy went short CAD and NOK and was paid. **57% of all in-sample profit came from that single 12-month window (July 2014 – June 2015).** Then it drifted sideways and down from 2016 to 2018.
- **Out-of-sample:** −3.5% in 2019, then a **sharp spike in 2020** (COVID crash, **+13.3%** for the year), roughly flat through 2022, then a **steep decline from 2023** (2023: −10.8%, 2025: −5.3%). **All the out-of-sample years other than 2020 together lost 20.5%.**

| Year | 2011 | 2012 | 2013 | 2014 | 2015 | 2016 | 2017 | 2018 |
|---|---|---|---|---|---|---|---|---|
| Portfolio | +3.2% | +2.2% | +1.6% | **+5.9%** | +0.3% | −2.3% | +0.6% | +2.2% |

| Year | 2019 | 2020 | 2021 | 2022 | 2023 | 2024 | 2025 | 2026 (to Sep) |
|---|---|---|---|---|---|---|---|---|
| Portfolio | −3.5% | **+13.3%** | −2.2% | +1.5% | **−10.8%** | −0.5% | −5.3% | +0.5% |

*Interpretation to have ready:* the strategy's results line up with **a few big macro shocks** (the 2014 oil crash, 2020 COVID), not with a steady weekly underreaction. That concentration is exactly what you'd expect from a result that isn't statistically significant (4.7). It is also the "driven by a single episode" warning from the Plan's falsification list.

*By currency:* all three currencies made money in-sample (CAD +5.0%, NOK +7.2%, AUD +1.4%, summed weekly net contributions) and all three lost out-of-sample (CAD −2.4%, NOK −3.8%, AUD −0.9%). So the result is **not driven by one currency**, but it **is** driven by one or two episodes.

### 4.4 Benchmarks (Notebook *Benchmarks* `benchmark_table`)

The question: is the commodity signal adding anything?

| OOS 2019–2026 | Strategy | Buy-and-hold FX | Naive FX momentum |
|---|---|---|---|
| Cumulative return | −7.9% | −4.9% | −9.8% |
| Sharpe | −0.20 | −0.07 | −0.24 |
| Volatility | 5.3% | 8.8% | 5.5% |
| Max drawdown | −20.1% | −22.5% | −18.7% |

- **Buy-and-hold** is always long all three at 1/3. It lost too, because all three currencies weakened against the USD. It is the answer to "maybe *any* FX exposure lost money".
- **Naive FX momentum** uses the *same* `run_strategy` machinery but passes `signal_source=domestic_fx_returns`, so it trades each currency on its own past return instead of the commodity's. This isolates the commodity information: if the commodity signal were special, it should clearly beat this.
- **Verdict:** the strategy is worse than just holding and only slightly better than plain FX momentum. The commodity link is not adding value (Report §3 "Backtest" ("Benchmarks, out-of-sample")).
- **Caveats** (the first is stated in both Notebook *Benchmarks* and Report §3; the second only in Report §3): (1) buy-and-hold is fully invested every week, while the strategy is fully flat in 42% of OOS weeks. So compare **Sharpe** (which doesn't depend on position size), not cumulative return or drawdown. (2) Spot-only returns flatter buy-and-hold most, because it is always long the three currencies and would have paid carry for much of 2019–2026.

### 4.5 Cost sensitivity (Notebook *Benchmarks*, the `cost_scenarios` cell)

| Cost multiplier | 0× (gross) | 0.5× | 1× (base) | 3× |
|---|---|---|---|---|
| OOS Sharpe | −0.14 | −0.17 | −0.20 | −0.31 |

**The key sentence:** *the strategy loses money out-of-sample even with zero trading costs.* So "it would have worked with a cheaper broker" is ruled out. The failure is in the signal, not the costs (Report §3 "Backtest" and §5 "Transaction costs and slippage").

### 4.6 Falsification criterion 3 (Notebook *Why It Failed*, Report §3 "Backtest" ("Falsification criterion 3"))

**Criterion 3: is the signal just the dollar in disguise?**
Oil and copper are priced in US dollars, so when the dollar strengthens, commodity prices tend to fall mechanically. A signal built on raw commodity returns could therefore be a hidden "dollar momentum" signal.

*The test* (`dollar_adjusted()` in *Why It Failed*): for each day, regress the commodity return on the dollar ETF (UUP) return over the trailing 252 days to get a slope β, using data up to the previous day only. Then keep only the residual, `r_t − β_{t−1} · usd_t`, which is the part of the commodity move not explained by the dollar. Re-run the frozen strategy on that residual, with nothing re-tuned. In IE terms, this is like removing a known common-cause factor (ambient temperature, say) from a sensor reading before using it as a control signal.

| Over the 785 common weeks | Raw signal | Dollar-adjusted signal |
|---|---|---|
| IS Sharpe | 0.46 | 0.38 |
| OOS Sharpe | −0.20 | −0.27 |
| Grid configs positive OOS | 0 of 9 | 0 of 9 |

(The raw IS Sharpe is 0.46 rather than 0.38 here because the comparison window starts in September 2011, when the adjusted signal first exists.)

*Reading (corrected):* the adjusted signal behaves much like the raw one in-sample, and both fail out-of-sample. But be careful how you phrase it. Criterion 3 asks whether predictability *disappears* after adjustment, and there was no *significant* predictability to begin with (IS t ≈ 1.1, see 4.7). So the test is **inconclusive** about whether the in-sample result was a dollar effect. The safe claim: **dollar-adjusting the signal doesn't rescue the strategy out-of-sample.**

> [!warning] A claim that was removed
> Earlier versions of the notebook and report said "the dollar hurt the strategy by moving the currencies being traded, not by contaminating the signal." The first half of that turned out to be wrong once positions were checked (see 5.3). Don't repeat it.

*(The old reverse-causality test, `weekly_ic()`, was removed from the notebook in the simplification round. Its result is kept in the Part 9 answer in case you're asked.)*

### 4.7 Statistical significance (Notebook *Backtest and Results*, the `significance_table` cell)

This is the most important framing change from the review. The old wording was "a modest in-sample edge that collapsed dramatically out-of-sample". The numbers don't support "edge" or "collapse":

| | In-sample | Out-of-sample |
|---|---|---|
| Mean weekly return | 3.3 bp | −1.8 bp |
| Arithmetic Sharpe | 0.40 | −0.17 |
| **t-statistic** | **1.12** | **−0.48** |
| Weeks | 408 | 403 |

t-statistic of the IS − OOS difference in mean weekly return: **1.08**.

*Reading:* neither period is distinguishable from zero, and the drop between them is only about one standard error. So:
- The in-sample result was **never strong evidence**. That isn't a flaw in the method; 8 years is simply too short to detect a Sharpe of about 0.4 (you'd need about 28 years for t = 2).
- The out-of-sample result is a **failure to find predictability**, not a measured collapse of something real.
- **The in-sample grid already pointed this way:** only the three 40-day configurations made money in-sample, and all six 10- and 20-day ones lost. Those shorter windows are the ones that match the pitch's "priced in over several days" mechanism. So the relationship was unstable across reasonable settings *before* the test period was opened. Report §3 and §5 (Overfitting) now say so.

*IE analogy:* you ran a pilot, saw a small improvement within the noise band, then the confirmation run showed nothing. The right conclusion is not "the improvement went away" but "there was never a detectable improvement."

---

## Part 5: Why it failed (Notebook *Why It Failed*, Report §7)

### 5.1 Leverage
A fixed 1/3 weight per active currency gives gross exposure between 0 and 1.0×, so there is **no leverage**.

### 5.2 Concentration: 3 currencies ≠ 3 independent bets
Weekly position correlations (Notebook *Why It Failed* output):

|  | CAD | NOK | AUD |
|---|---|---|---|
| CAD | 1.00 | **0.82** | 0.27 |
| NOK | **0.82** | 1.00 | 0.30 |
| AUD | 0.27 | 0.30 | 1.00 |

WTI and Brent are both crude oil (their daily returns are about 0.88 correlated), so CAD and NOK nearly always get the same signal. **It is really two oil bets plus one copper bet.** In 32.3% of weeks, 2 or more positions point the same way.

### 5.3 The worst drawdown, measured (corrected)
Notebook *Why It Failed* finds the OOS max drawdown: **−20.1%, from 2021-08-20 to 2026-04-03**. Over that window:
- CAD −9.1%, NOK −8.6%, AUD −4.3% (all weakened against the USD; average **−7.4%**)
- Dollar index ETF (UUP) **+25.1%**

**The tempting story is wrong.** The old version said a dollar bull market "dragged all three positions down together". That only works if the strategy was mostly *long* the currencies. The new attribution table in Notebook *Why It Failed* checks what it actually held:

| In the drawdown window | Long weeks | Short weeks | Flat weeks | Avg position | P&L on longs | P&L on shorts | Net P&L |
|---|---|---|---|---|---|---|---|
| CAD | 44 | 45 | 153 | −0.00 | −2.2% | −0.6% | −3.2% |
| NOK | 45 | 47 | 150 | −0.01 | −2.2% | **−7.1%** | **−10.3%** |
| AUD | 43 | 52 | 147 | −0.04 | −1.7% | **−4.0%** | −6.2% |

(P&L in portfolio terms, i.e. already scaled by the 1/3 weight. Only 42.4% of the 276 active currency-weeks made money.)

*Reading:*
- The strategy was long about as often as short, with an **average position close to zero**. A strategy with almost no net exposure can't lose 20% just because the currencies fell.
- It lost money on its **shorts** too, and for NOK and AUD it lost *more* on shorts than on longs.
- **So the drawdown is a signal failure:** the commodity signal pointed the wrong way more often than the right way. It is not a dollar-exposure failure.
- **UUP overstates the dollar move.** It is a futures-based ETF, so its return includes the USD's interest-rate advantage (large in 2023–25), and its basket is mostly euro. The three traded currencies fell only 7.4% on average.

*Is a dollar rally still a risk?* Yes, structurally. In about a third of weeks two or more positions point the same way, and a dollar shock in such a week would hit them all at once. It just isn't what produced *this* drawdown. Report §7 now says exactly this.

*IE analogy:* the line had a bad month while raw-material prices rose, but the material mix was balanced across suppliers, so the cost rise can't explain the losses. The process itself (the signal) was making bad calls.

### 5.4 An operational risk that surfaced
Report §7 now also notes that one bad data print (the negative WTI price) silently took CAD out of the strategy for 59 weeks. In live trading, that would be an unmonitored gap in exposure. It is a good example of risk that isn't about markets at all.

---

## Part 6: Is the result trustworthy? (Notebook *Pitfall Checks* to *Efficiency Touchpoint*, Report §5–§6)

The rubric says a negative result is fine **only if the method was a fair test** (Rubric §8). This section is where you prove it was.

### 6.1 Look-ahead bias: the corruption test (Notebook *Pitfall Checks*)
```python
decision_date = pd.Timestamp("2021-06-25")
corrupted_returns.loc[corrupted_returns.index > decision_date] = 999.0
assert np.isclose(original_signal, corrupted_signal)   # 0.108926 == 0.108926 → PASS
```
Overwrite **every future data point** with garbage and confirm the signal on the decision date doesn't change. If any `shift` or `rolling` anywhere in the chain peeked forward even one day, the 999s would leak in. It is a **property test**: it checks the behaviour rather than trusting that the code reads correctly. It is the strongest kind of evidence you can give.

*What it does **not** prove:* that the dates on the raw data mean what we think they mean. That is the next check.

### 6.2 Look-ahead bias, part 2: timestamp alignment (Notebook *Pitfall Checks*, Report §4 "Signal Construction Reasoning" ("What would break") and §5 "Look-ahead bias")
Two rows with the same date don't necessarily describe the same moment. FX trades around the clock, so a daily FX "close" is just a vendor convention, while futures and the UUP ETF close in the US afternoon. The check correlates each daily FX return at *t* with the commodity and dollar returns at *t−1*, *t* and *t+1*:

| Correlation of FX(t) with… | other at t−1 | other at t (same day) | other at t+1 |
|---|---|---|---|
| Own commodity, Jan–Aug 2010 | 0.07–0.26 | **0.51–0.61** | ≈0–0.17 |
| Own commodity, 2011 onward | **0.30–0.38** | 0.04–0.09 | ≈0 |
| Dollar index, 2011 onward | **0.52–0.66** | 0.04–0.06 | ≈0 |

*Reading:* from late 2010, Yahoo's FX row dated *t* reflects roughly the US session of *t−1*. The dollar index is the cleanest evidence, because these currencies move mechanically with the dollar, so its correlation peaks wherever the true alignment is.

*Why it doesn't break the strategy:* the Friday signal contains commodity data only through Thursday. Under the offset, Thursday's commodity move lines up with the FX row dated Friday, which belongs to week W and is never traded. Trading starts with Monday's row of week W+1. The buffer is hours, not a full day, but it is never negative. Robustness check with extra lag:

| Extra lag | IS Sharpe | OOS Sharpe |
|---|---|---|
| 0 (headline) | 0.38 | −0.20 |
| +1 day | 0.41 | −0.27 |
| +2 days | 0.21 | −0.11 |

*The cautionary point:* a **daily** version of this strategy would have seen a 0.3 correlation between yesterday's commodity return and today's FX return and mistaken it for strong underreaction. It's a timestamp artefact. This is great material for your Reflection ("what was wrong in my assumptions about the data").

### 6.3 Survivorship bias, and the futures-roll check (Notebook *Pitfall Checks*)
**Argued not applicable:** survivorship bias needs a hand-picked list of assets where some dropped out, such as delisted stocks. CAD, NOK and AUD existed for the whole period, and continuous futures series don't "die".

**The related issue: rolls (corrected).** The old text said Yahoo's futures were "already roll-adjusted" and that the median filter checked for roll problems. Both claims were wrong:
1. The series are **raw front-month splices**, not back-adjusted (see 1.3). The −$37.63 print is the expiring May 2020 contract's own price.
2. The median filter only catches **spike-then-revert** prints. A roll gap is a permanent **step**, and a step stays close to its 3-point median, so the filter is blind to it by design.

**Check 1: median filter at 15% (still there, now correctly described).** It flagged one day, **WTI 2020-03-18** (20.37 vs local median 25.22). By hand: 26.95 → 20.37 → 25.22 → 22.43, a real, chaotic COVID/price-war week that **did not revert**. That's market history, not a glitch, so it was left untouched.

**Check 2: WTI–Brent gap (new, designed for steps).** WTI and Brent are both crude, so their daily returns are normally close (the gap has a 1.22% daily standard deviation). They roll on different dates, so a roll gap in one shows up as a jump in the *difference*.
- **90 days** exceed a 3% gap (17 of them in April–June 2020, the WTI expiry chaos).
- A clear example: **Brent fell from 118.35 to 101.16 on 2026-04-01 while WTI barely moved** (101.38 → 100.12). That is the signature of a roll into a much cheaper next contract (steep backwardation), not a market move.
- **Robustness:** re-running with both oil returns set to zero on all 90 flagged days (deliberately blunt; it also removes some real divergence). The frozen config gives **IS Sharpe 0.47, OOS −0.49**, and **0 of 9** configs are positive OOS. The numbers move, but the conclusion doesn't.
- **Copper** has no twin series in the dataset, so its rolls are **unverified**. That is a stated limitation. The production fix is to use back-adjusted continuous contracts.

*Where it lives now:* the roll material has moved out of Survivorship. In the report it's the **"Futures rolls" paragraph in §2 Data Pipeline, under "Real-world messiness"**, and §5 Survivorship is now just the short "not applicable" argument, which ends by pointing to §2. In the notebook it has its own **"Data quality: futures rolls"** subsection, straight after Survivorship bias in the pitfall checks.

### 6.4 Data handling: the negative WTI print (Notebook *Pitfall Checks*)
See the callout in 3.2, Step 4. Masked (headline): CAD has no signal for 59 OOS weeks. Bridged: OOS Sharpe −0.17, 0 of 9 positive, same in-sample winner. The conclusion doesn't depend on the choice.

### 6.5 Overfitting / data-snooping (Notebook *Pitfall Checks*)
- **Exactly 9 variations** were tried, all reported, with the holding period fixed rather than searched.
- **0 of 9** were positive out-of-sample.
- **Spearman rank correlation of IS vs OOS Sharpe = 0.23**: the in-sample ranking says almost nothing about the out-of-sample ranking.
- *The argument:* if only the chosen configuration had failed, you could blame bad luck in selection. Because **every** configuration failed, it is a property of the hypothesis, not the tuning (Notebook *Pitfall Checks* markdown after the table).
- **Added to Report §5 "Overfitting/data-snooping":** even *in-sample*, only the three 40-day configurations made money and all six 10/20-day ones lost. So the relationship was already unstable across reasonable parameter choices before the out-of-sample period was opened (see 4.7).

### 6.6 Transaction costs: see 4.5.

### 6.7 Out-of-sample validation
408 IS weeks vs 403 OOS weeks. Sharpe fell from 0.38 to −0.20 (a change of −0.58). That *looks* like a collapse, but neither number is significant (t = 1.12 and −0.48) and the difference is only about one standard error (t = 1.08). Report §5 used to call it "a dramatic collapse, not a modest weakening". It now says the in-sample result was never strong evidence and the out-of-sample period shows no predictability at all. The conclusion is unchanged: the hypothesis does **not** hold at a tradeable level.

**Why 2019 as the split?** It gives two roughly equal halves (408 vs 403 weeks). It wasn't chosen to include stress periods, but the OOS happens to contain COVID and the 2022 dollar surge, which makes it a demanding test (Report §4 "Train/Test Split").

### 6.8 Efficiency touchpoint (Notebook *Efficiency Touchpoint*, Report §6)
- **Most expensive step:** the grid search, **O(G · C · n)** with G = 9 grid points, C = 3 currencies and n ≈ 4,200 days. That is 27 strategy evaluations in **0.162 s** in the saved run (about 6 ms each). Report §6 just says "about 0.16 seconds", which stays true however the timing varies between runs.
- The robustness checks (dollar-adjusted grid, bridged grid, roll-cleaned grid) are *new* computations on different inputs, not repeats of the stored grid. Each is one more O(G·C·n) pass. Report §6 mentions this in one clause.
- Rolling windows are **O(n)**, not O(n × window), because pandas updates the window sum incrementally as it slides instead of re-summing.
- **Fixed:** an earlier version ran the grid 3 times. Now it runs once and the results are stored. This was fixed for **correctness**, not speed: three copies of the same computation can silently drift apart.
- **Left in place:** the z-score is recomputed for each threshold even though it depends only on lookback (a 3× redundancy). It costs milliseconds and is contained in one function.
- *Note:* the "fixed" and "left in place" points above are no longer written up in the notebook or the report. They're answers to have ready if asked.
- **Scaling note (corrected):** walk-forward validation would multiply the cost by R refit windows, O(G·C·n·R). Even with annual refits (R ≈ 8) that's about a second. So **computing cost is not a reason to avoid walk-forward**. The old Report §6 implied it was, and a grader would have spotted that 0.16 s × 8 is trivial. The fixed split is now defended only on methodological grounds (Report §4). Caching would only matter with a much bigger grid or universe, or daily refits.

---

## Part 7: Methodology defense cheat sheet (Report §4)

For each choice: **what, why, what alternative was rejected, and what breaks if the assumption is wrong.** Rubric §4 asks for exactly this. In the report, these are grouped under four subheadings: **Universe**; **Train/Test Split**; **Transaction Cost Assumptions** (costs, and spot returns without carry); and **Signal Construction Reasoning** (per-currency z-score, holding period, sizing, and the one-day lag in "What would break").

| Choice | Why | Alternative rejected | If wrong… |
|---|---|---|---|
| **Universe:** CAD/NOK/AUD, one commodity each | Liquid G10 currencies with clear, well-known commodity links and clean data from 2010 | Wider set (NZD, ZAR, BRL, CLP): brings back illiquidity and data-quality problems | Copper is a weak AUD proxy (iron ore is bigger), so AUD is a noisy test |
| **Per-currency z-score** | "Large" is measured against each commodity's own volatility regime; it tests the hypothesis exactly as stated (a currency underreacts to *its own* commodity) | Cross-sectional ranking (long the best, short the worst): with only 3 assets it becomes an arbitrary relative-value bet | After a volatility regime shift (oil in 2020), the 252-day baseline is miscalibrated for months |
| **Holding period = 1 week, fixed** | The hypothesis says "the following week"; searching it would be data-snooping | Searching 5/10/20 days, as the Plan suggested | Maybe the true diffusion horizon is different. That cost is accepted so the result means something |
| **Search only lookback × threshold (9 combos)** | A small, pre-declared grid keeps the number of variations honest | Larger grids | n/a. The whole grid failed OOS anyway |
| **Fixed split 2011–18 / 2019–26** | Simple, one clearly declared test, easy to report honestly | Walk-forward: more OOS data, but re-opens selection at every refit | A single split is one sample of history; a different split might look different |
| **Costs 2 bp (CAD, AUD), 5 bp (NOK)** | USDCAD and AUDUSD are among the most traded pairs; USDNOK is thinner | A uniform cost would understate NOK | Tested at 0×, 0.5×, 1× and 3×, and the conclusion holds at every level |
| **Fixed 1/3 weight, no leverage** | Simple, and no extra parameter to justify | Inverse-volatility sizing | Concentration comes from correlated signals, not sizing (see Part 5) |
| **1-day lag + weekly shift** | Only past information is ever used | n/a | The closes *are* misaligned by about a day in the FX data. The skipped Friday absorbs it, and extra lag doesn't change the conclusion (6.2) |
| **Mask the negative WTI price** | Don't invent a price | Bridge it (carry the prior price forward) | CAD sits out 59 OOS weeks. Bridged version gives the same conclusion (6.4) |
| **Spot returns, no carry** (new in Report §4) | Avoids a second data source with its own alignment problems; the strategy is long and short about equally often, so carry largely nets out | Add carry from short-term interest rates | If positions had been persistently one-sided, returns would be biased by the rate differential. Buy-and-hold is the most affected |
| **Raw (not back-adjusted) futures** | It's what yfinance provides | Back-adjusted continuous contracts | Roll gaps leak into the signal. Zeroing every suspected oil roll day leaves 0 of 9 configs positive OOS (6.3). Copper is unverified |

---

## Part 8: What was changed, and what is still yours to do

### 8.1 Changes made to the notebook
Originals are backed up outside the project folder. After these changes the notebook re-runs end-to-end from `data/raw/` with no errors, and every original output is identical except the grid timing (0.164 s → 0.156 s).

| Where | Change | Why |
|---|---|---|
| §A download cell | Added `USE_SNAPSHOT = True`: always load the submitted snapshot | Previously, re-running more than 5 days after the snapshot silently re-downloaded all data and changed every number. A grader re-running it would not have matched the report |
| §B log-returns cell + markdown | Prints both masked WTI return dates and documents that CAD has no signal for 59 OOS weeks | This side effect was undocumented |
| §C after rolling-correlation chart | Note explaining that the 2010 "collapse" is a timestamp effect | So nobody reads it as an economic regime change |
| §E after `results_table` | Prints the share of fully-flat weeks and the hit rate over weeks with a position open | Explains the low headline hit rate |
| §G (new) | Falsification criterion 4 test (dollar-adjusted signal) and criterion 3 test (weekly IC both directions) | Both were stated in the pitch but untested |
| §H (new) | Timestamp lead/lag check plus extra-lag robustness | The corruption test can't detect vendor timestamp conventions |
| §H (new) | Masked vs bridged WTI comparison | Shows the CAD gap doesn't drive the result |

`performance_metrics()`, the strategy, the grid and every headline number are **unchanged**.

### 8.2 Changes made to the report
| Section | Change |
|---|---|
| Top | Added the one-line title and a **`[Your name] · Submitted [submission date]` placeholder: fill this in** |
| §2 Universe and sources | Describes the new snapshot policy |
| §2 Cleaning | Corrected missing-value counts (1–8 per series, Brent 32) |
| §2 Real-world messiness | Two masked returns, the 59-week CAD gap, the bridged robustness result, and the two NOK bad ticks (previously referenced in §5 but never described) |
| §2 Feature construction | Rewritten to match what the code actually builds (it previously listed features the notebook never made) |
| §3 | Hit-rate note under the table; "from 2022 onward" → "from mid-2023 onward"; cost sentence now includes gross −0.14; new paragraph with the criterion 3 and 4 results |
| §4 What would break | Replaced the hypothetical close-time concern with the measured offset and why the weekly design survives it |
| §5 Look-ahead bias | Added the timestamp check |
| §6 | Timing 0.164 → 0.156 s; notes that the robustness grids are extra computations |
| §7 | Softened "precisely the falsification scenario" and cited the direct test; added the CAD-gap operational risk |
| §8 Reflection | **Untouched**, as requested |

### 8.3 Second round of changes: methodology review (3 Oct 2026)
A review found three logical problems and several smaller ones. Originals of the notebook, report and this guide from before this round are backed up outside the project folder. The notebook was re-executed end-to-end from `data/raw/` with no errors or warnings. **Every existing headline number is unchanged**; only new outputs were added (and the grid timing, now 0.158 s).

**Notebook (now 95 cells):**

| Where | Change | Why |
|---|---|---|
| §D weekly mechanics markdown | Added "Spot returns only, no carry" and the NaN-transition cost edge case | Both were undocumented assumptions (3.4) |
| §E metrics markdown | States the Sharpe definition (compounded return ÷ vol, no risk-free rate) | It wasn't stated anywhere (1.6) |
| §E (new, after hit rate) | **Significance check:** t-statistics per period and for the difference | The old narrative of "edge then collapse" wasn't supported (4.7) |
| §E (new, after equity curve) | **P&L by year and currency**, oil-crash share, 2020 vs the rest | Shows the episode concentration; replaces numbers that were only in this guide (4.3) |
| §E headline markdown | Reworded to "no reliable evidence in either period" | Follows from the significance check |
| §F benchmark markdown | Buy-and-hold caveats: exposure and carry | Cumulative return and drawdown aren't comparable across different exposures (4.4) |
| §G drawdown cell | **Rewritten:** adds the long/short/flat attribution, P&L on longs vs shorts, active-week hit rate, and the average currency move | The old "dollar dragged all positions" claim was contradicted by the positions (5.3) |
| §G drawdown markdown | Rewritten: signal failure, not dollar exposure; UUP caveats | Same |
| §G criterion 4 result | "Not what sank the hypothesis" → **inconclusive** | You can't test whether non-significant predictability "disappears" (4.6) |
| §H survivorship markdown | Corrected "already roll-adjusted"; explains why the median filter can't see rolls | Factual error, and a check that couldn't detect what it claimed to (6.3) |
| §H (new) | **WTI–Brent roll-gap check** plus a zeroed-days robustness grid | A check that can actually see steps (6.3) |
| §I efficiency markdown | Walk-forward is no longer described as costly | 0.16 s × R is trivial (6.8) |

**Report:**

| Section | Change |
|---|---|
| §3 table | New row: t-statistic 1.12 / −0.48 |
| §3 note under table | Sharpe definition and how to read the t-statistic |
| §3 main paragraph | Rewritten: neither period significant; only the 40-day configs worked in-sample |
| §3 (new paragraph) | "Where the P&L came from": the 57% oil-crash share, 2020 +13.3% vs the rest −20.5%, per-currency splits |
| §3 benchmarks | Buy-and-hold exposure and carry caveats |
| §3 criteria 3 and 4 | Criterion 4 now **inconclusive**; criterion 3 unchanged |
| §4 (new paragraph) | "Why spot returns, without carry" |
| §5 Survivorship | Rewritten: rolls aren't adjusted, the median filter can't see them, the new WTI–Brent check, copper unverified |
| §5 Overfitting | Added the in-sample instability point |
| §5 Out-of-sample validation | "Dramatic collapse" → not significant in either period |
| §6 | Timing 0.158 s (varies by run); walk-forward paragraph no longer cites cost |
| §7 drawdown paragraph | Rewritten as a signal failure, with the long/short evidence and UUP caveats |
| §1, §2, §8 | Untouched |

### 8.4 Third round: simplification (3 Oct 2026)
Two changes. Pre-round copies of the report, notebook, pitch and this guide are backed up outside the project folder.

**1. Report shortened** (about 5,000 → about 3,560 words), mainly in the sections the rubric asks to be brief. §3 and §5 were not shortened, and every number and conclusion is unchanged:

| Section | Before | After | What was cut |
|---|---|---|---|
| §2 Data Pipeline | ~750 words | ~460 | Implementation history (the stale-snapshot bug, Brent gap dates) and detail that is in the notebook |
| §4 Methodology Defense | ~1,075 | ~610 | Same 8 choices, each now *choice → why → alternative rejected*, plus "what would break" |
| §6 Efficiency | ~500 | ~120 | The "removed / left in place" inefficiency story (still in Notebook §I and 6.8 here) |
| §7 Risk | ~350 | ~120 | One paragraph: leverage, CAD–NOK concentration, the drawdown as signal failure, the CAD gap |

What you might be asked about that's **no longer in the report** but still in the notebook: the triple-run grid fix and the z-score caching redundancy (6.8), the exact Brent gap dates (3.2), and the UUP carry caveat (5.3).

**2. Falsification criterion 3 (reverse causality) dropped.**
- Removed the bullet from Report §1 and the "Criterion 3" half of the Report §3 falsification paragraph.
- Deleted the three Notebook §G cells (heading, `weekly_ic()` code, result), leaving 89 cells.
- Renumbered the old criterion 4 (dollar) to **criterion 3** in the report, notebook and this guide.
- The notebook was test-run on a copy and runs end-to-end with no errors. Your saved notebook wasn't re-executed, so its outputs are your own run.

> [!warning] Your separate pitch file
> `capstone_hypothesis_pitch.docx` still lists the reverse-relationship criterion as its third bullet. It had unsaved changes open in Word, so it was **not** edited. Delete that bullet yourself so the pitch matches the report and notebook.

### 8.5 Fourth round: report restructured (4 Oct 2026)
- **New layout:** the report now follows the headings of your draft, `capstone_hypothesis_pitch.docx`. See the layout table at the top of this guide for what sits under each subheading. No content was cut; paragraphs were regrouped, and opening labels that just repeated their new subheading (such as "Why this universe:") were dropped.
- **Added:** the 9-configuration grid table under §3 "Strategy", and your name and submission date from your draft.
- **Rubric-exact headings** in both the report and your draft: "4. Methodology Defense" (not "Defence"), and the pitfall subheadings "Look-ahead bias", "Survivorship bias", "Overfitting/data-snooping", "Transaction costs and slippage" and "Out-of-sample validation".
- **Your draft's results table:** out-of-sample volatility corrected from 5.4% to 5.3%.
- **A lesson from this round:** Word saved its old open copy of the report over a newer version on disk, undoing three rounds of fixes. They were restored from the latest version. When a document is edited outside Word, close it in Word **without saving** before reopening.

### 8.6 Still yours to do
- [x] **Delete the reverse-relationship bullet** ("The reverse relationship (FX predicts commodities) is stronger") from `capstone_hypothesis_pitch.docx`.
- [ ] **Write §8 Reflection.** Material for "what was wrong in my assumptions":
  - the timestamp artefact (6.2), about the data;
  - the in-sample "edge" never being significant (4.7), about the evidence;
  - the drawdown that *looked* like a dollar story but was the signal being wrong in both directions (5.3), about the market;
  - Yahoo futures not being roll-adjusted (6.3), about the data;
  - results driven by the 2014–15 oil crash and 2020 (4.3);
  - "three currencies" really being two bets (5.2);
  - one bad print disabling CAD for 59 weeks (6.4).

  Next steps: walk-forward validation, back-adjusted futures, an iron ore series for AUD, including carry, FX data with properly aligned closes (for example the WM/Reuters 4pm London fix), a longer sample or more currencies for statistical power, and a regime filter on dollar trend.
- [x] **Fill in the name/date placeholder** at the top of the report (now "Tsz Lok Pierson Chu · Submitted October 23", from your draft).
- [ ] **Add a References section** if you cite anything. See 2.5, and only cite what you've read.
- [ ] **Export the report to PDF** (the rubric accepts PDF or Markdown, not .docx). Do this after writing the Reflection.
- [ ] Optional: the section titles in the .docx are bold body text rather than Word "Heading" styles. Converting them gives the PDF a clickable outline, but it's cosmetic.

### 8.7 Limitations worth one sentence each (a finance professor may raise them)
- **Spot returns only, no carry.** Holding a currency in real life also earns or pays the **interest rate difference** between the two countries. The backtest uses spot-price changes only. Over 2022–2026, US rates were often higher than Canadian, Norwegian or Australian rates, so being *long* these currencies would also have cost carry. The strategy was long and short about equally often, so it nets out for the strategy; buy-and-hold would look worse. Stated in Report §4 "Transaction Cost Assumptions".
- **Sharpe without a risk-free rate.** Stated in Report §3 "Results", in the note under the table (1.6).
- **No statistical power.** Eight years per period can't detect a Sharpe of about 0.4 (4.7). A longer history or a wider universe is the only fix.
- **UUP is not the spot dollar.** It's a futures ETF (it includes carry) and mostly euro-weighted. It is fine as a control for the dollar-adjusted signal, but it overstates the dollar's move against CAD/NOK/AUD (5.3).
- **Copper rolls unverified.** The WTI–Brent check only covers oil (6.3).

---

## Part 9: Likely professor questions, with answers

**Q: In one sentence, what did you test?**
Whether a big move in oil or copper predicts the linked currency (CAD, NOK, AUD) moving the same way over the following week, because FX markets are slow to price in commodity news.

**Q: Why should that be true economically?**
A higher export price means more export revenue, better public finances and a better growth outlook, which raise the currency's fair value. Currency traders focus on the dollar and interest rates and want confirmation before acting, so the currency catches up over days rather than instantly.

**Q: Isn't this just momentum?**
Not own-asset momentum. The signal comes from a *different* market (the commodity), and I benchmarked against plain FX momentum using identical code (Notebook *Benchmarks*). The commodity version was only marginally better (−0.20 vs −0.24 Sharpe OOS), which is evidence the commodity link adds little.

**Q: How do you know you didn't use future data?**
Every predictor goes through a single `.shift(1)` in `trailing_return`, then a second shift when daily signals become weekly positions. I verified it with a corruption test (Notebook *Pitfall Checks*): overwrite everything after a decision date with 999 and the signal on that date is unchanged. Because that test can't catch a vendor timestamp convention, I also measured the alignment directly (Notebook *Pitfall Checks*). The FX data turned out to be stamped about a day late, but the weekly design keeps the signal and the traded returns separate, and adding one or two extra days of lag doesn't change the conclusion.

**Q: How many things did you try?**
Nine: 3 lookbacks × 3 thresholds. The holding period was fixed by the hypothesis. All nine are reported, selection used in-sample net Sharpe only, and the winner was frozen before the out-of-sample results were used.

**Q: Why did it work in-sample and fail out-of-sample?**
I'd push back on "work". The in-sample t-statistic is only 1.1, so the in-sample result was never distinguishable from luck, and the drop to out-of-sample is about one standard error. Both periods are dominated by single episodes: 57% of the in-sample profit came from the 2014–15 oil crash, and out-of-sample 2020 made +13% while every other year combined lost 20%. The rank correlation between in-sample and out-of-sample Sharpe across the grid is only 0.23. Everything points to noise rather than an edge that decayed.

**Q: Is the in-sample result statistically significant?**
No. t = 1.12 over 408 weeks. As a rule of thumb t ≈ Sharpe × √years, so a Sharpe of 0.38 would need about 28 years of data to reach t = 2. On top of that, only the 40-day lookback made money in-sample; the 10- and 20-day versions, which are closer to my "priced in over several days" mechanism, lost money even in-sample.

**Q: The worst drawdown happened during a big dollar rally. Isn't that what killed it?**
That's what I first assumed, but the positions say no. During the drawdown the strategy was long about as often as short in every currency, with an average position near zero. It lost money on shorts as well as longs (NOK shorts lost more than NOK longs), and only 42% of active currency-weeks were profitable. So it was the signal being wrong in both directions, not dollar exposure. UUP's +25% also overstates the move: it includes the USD's interest-rate advantage and is mostly euro-weighted, while CAD, NOK and AUD fell about 7% on average. A dollar rally is still a structural risk whenever the signals line up long, but it isn't what happened here.

**Q: Wasn't the in-sample profit just the dollar rally?**
I tested that directly (Notebook *Why It Failed*). The dollar-adjusted signal behaves much like the raw one (in-sample Sharpe 0.38 vs 0.46 over the same weeks), and it fails out-of-sample too. But I can't claim more than that: the criterion asks whether predictability *disappears* after adjustment, and there was no significant predictability to begin with. So the test is inconclusive on that question. What it does show is that stripping out the dollar doesn't rescue the strategy.

**Q: Are your futures series roll-adjusted?**
No, and an earlier draft wrongly said they were. Yahoo's `=F` series are raw front-month splices; the −$37.63 WTI print is the expiring contract's own price. My spike filter can't see roll gaps because they're steps, not spikes, so I added a check that compares WTI with Brent, which roll on different dates. Zeroing all 90 suspected oil roll days leaves 0 of 9 configurations positive out-of-sample. Copper I couldn't verify. A production version should use back-adjusted contracts.

**Q: Maybe FX predicts commodities instead?**
I originally had that as a falsification criterion and tested it with the same z-score recipe in both directions, using the weekly information coefficient (the correlation between this week's signal and next week's return). Every value was within about two standard errors of zero; the largest was 0.09, NOK commodity→FX in-sample. So neither direction led the other at a weekly horizon. I dropped it from the final version because it was the least informative test and nothing else depended on it. The literature does find the reverse direction at other horizons (Chen, Rogoff & Rossi, 2010).

**Q: What happened with the negative oil price?**
I masked it rather than invent a price. That removes two daily returns, and because the rolling windows need complete data, CAD had no signal for 59 out-of-sample weeks. I re-ran with the price bridged instead: out-of-sample Sharpe −0.17 instead of −0.20, and still 0 of 9 configurations positive. So the choice doesn't matter for the conclusion, but it was a useful lesson in how one bad data point can quietly disable a whole leg.

**Q: If a grader re-runs your notebook, will they get the same numbers?**
Yes. `USE_SNAPSHOT = True` makes the notebook load the submitted `data/raw/` files rather than re-download, so every number is reproduced (only the timing figure varies slightly).

**Q: Could transaction costs explain it?**
No. With zero costs the OOS Sharpe is still −0.14.

**Q: Could a different parameter choice have worked?**
Not within the grid: 0 of 9 were positive out-of-sample. It is a failure of the whole family, not an unlucky pick.

**Q: Why these three currencies? Why not more?**
They are liquid G10 currencies with clear single-commodity links and clean data. Emerging-market exporters would add data-quality and liquidity problems. Three is enough to check the result isn't driven by one currency, and indeed all three made money in-sample and all three lost out-of-sample.

**Q: Isn't copper a poor proxy for Australia?**
Yes. Iron ore is the bigger export, but there is no clean, liquid iron ore futures history on yfinance. I say so in the pitch and methodology defense. It makes AUD the noisiest of the three tests.

**Q: How diversified is the strategy?**
Less than it looks. CAD and NOK positions are 0.82 correlated because WTI and Brent move together, so it is effectively two bets. Leverage is capped at 1×.

**Q: What is a z-score doing here?**
It measures how unusual the commodity's recent return is compared with its own last year, in standard deviations. It is like a control chart: trade only when the point is beyond ±1σ, so "large" is judged relative to each commodity's normal volatility.

**Q: Why log returns?**
They add up over time, so a multi-day return is just a rolling sum. That makes the lag logic and the weekly aggregation simple and consistent.

**Q: What is the most expensive computation, and is it a problem?**
The grid search: O(G·C·n), 27 evaluations in about 0.16 s. It isn't a problem at this scale. Walk-forward would multiply it by the number of refits R, but even annual refits make that about a second, so cost wasn't why I used a fixed split.

**Q: What did you learn?**
A real economic relationship (commodities and commodity currencies move together) does not imply a tradeable *lagged* one. Eight years of weekly data can't distinguish a small edge from luck, so a positive in-sample Sharpe means little on its own. The obvious story for a drawdown (the dollar did it) can be wrong, and you have to check positions before telling it. And data plumbing matters as much as the model: a one-day timestamp offset would have looked like strong underreaction in a daily strategy, the futures weren't roll-adjusted, and one negative price silently removed CAD for over a year. *(Write your own version in the Reflection.)*

**Q: What would you do next?**
Use FX data with properly aligned closing times (for example the WM/Reuters 4pm London fix) and back-adjusted futures, use walk-forward validation, get more statistical power from a longer history or more commodity currencies, find an iron ore series for AUD, include carry (the interest rate difference), and try a regime filter that stands aside when the broad dollar is trending strongly.

---

## Appendix: Key numbers at a glance

| Item | Value | Source |
|---|---|---|
| Data span | 2010-01-04 to 2026-09-14, 4,200 aligned days, 0 rows dropped | Notebook *Data Cleaning* |
| FX bad ticks fixed | USDNOK 2020-03-20, 2021-12-27 | Notebook *Data Cleaning* |
| Negative oil price | WTI −37.63 on 2020-04-20, masked; CAD has no signal for 59 OOS weeks | Notebook *Data Cleaning*, *Pitfall Checks* |
| Selected config | 40-day lookback, 1.0σ threshold | Notebook *Backtest and Results* |
| First tradeable week | 2011-03-11 (292-day warm-up) | derived |
| IS / OOS Sharpe | 0.38 / −0.20 | Notebook *Backtest and Results* |
| IS / OOS cum. return | 13.8% / −7.9% | Notebook *Backtest and Results* |
| IS / OOS max drawdown | −9.3% / −20.1% | Notebook *Backtest and Results* |
| OOS gross (no-cost) Sharpe | −0.14 | Notebook *Benchmarks* |
| Buy-and-hold / FX-momentum OOS Sharpe | −0.07 / −0.24 | Notebook *Benchmarks* |
| CAD–NOK position correlation | 0.82 | Notebook *Why It Failed* |
| Worst drawdown window | 2021-08-20 to 2026-04-03; UUP +25.1%; CAD/NOK/AUD average −7.4% | Notebook *Why It Failed* |
| Drawdown attribution | Long ≈ short weeks in every currency, avg position ≈ 0; NOK shorts −7.1% vs longs −2.2%; 42.4% of active currency-weeks profitable | Notebook *Why It Failed* |
| IS / OOS t-statistic | 1.12 / −0.48; difference t = 1.08 | Notebook *Backtest and Results* |
| IS / OOS arithmetic Sharpe | 0.40 / −0.17 | Notebook *Backtest and Results* |
| Grid: positive OOS configs | 0 of 9; IS–OOS rank correlation 0.23 | Notebook *Pitfall Checks* |
| Grid runtime | 0.162 s for 27 evaluations (varies slightly by run) | Notebook *Efficiency Touchpoint* |
| Share of IS profit from Jul 2014–Jun 2015 | 57% | Notebook *Backtest and Results* |
| OOS 2020 vs all other OOS years | +13.3% vs −20.5% | Notebook *Backtest and Results* |
| Per-currency net P&L, IS / OOS | CAD +5.0% / −2.4%, NOK +7.2% / −3.8%, AUD +1.4% / −0.9% | Notebook *Backtest and Results* |
| WTI–Brent roll check | 90 days with gap > 3%; zeroed: IS 0.47 / OOS −0.49, 0 of 9 positive | Notebook *Pitfall Checks* |
| Fully-flat weeks / hit rate when a position is open | 34.1% / 57.2% IS; 41.7% / 51.9% OOS | Notebook *Backtest and Results* |
| Commodity(t−1) vs FX(t) daily correlation, 2011+ | 0.30–0.38 (vs 0.04–0.09 same-day) | Notebook *Pitfall Checks* |
| Dollar index(t−1) vs FX(t), 2011+ | 0.52–0.66 (vs 0.04–0.06 same-day) | Notebook *Pitfall Checks* |
| OOS Sharpe with +1 / +2 days extra lag | −0.27 / −0.11 | Notebook *Pitfall Checks* |
| Bridged-WTI OOS Sharpe | −0.17 (0 of 9 positive) | Notebook *Pitfall Checks* |
| Dollar-adjusted signal IS / OOS Sharpe (vs raw, common weeks) | 0.38 / −0.27 (vs 0.46 / −0.20) | Notebook *Why It Failed* |
