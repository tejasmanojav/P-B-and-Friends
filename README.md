# P/B and Friends: Can Cheap, Profitable Stocks Beat the Market?

Value investing says to buy companies that are cheap compared with what they own. The simplest way to measure that is the **Price-to-Book (P/B)** ratio. But a stock can be cheap because the business is failing (a "value trap"). So I added profit checks, **Return on Assets (ROA)** and **Return on Equity (ROE)**, and backtested whether cheap + profitable stocks beat the market.

This notebook tests P/B, ROA and ROE filters on US stocks from **March 2021 to September 2025** (18 quarters), in two ways:

- **Part A:** sector by sector, against each sector's ETF (XLK, XLF, XLE, …)
- **Part B:** one portfolio from the 500 biggest US stocks, against **SPY**

**Short answer:** Cheapness alone didn't beat the market. In my tests, *how profitable* a company was mattered more than how cheap it was. The only filters that beat SPY kept the top 25% most profitable companies, and they left only about 7 stocks, so random chance plays a big part. I also learned that data cleaning and validation matter a lot, because bad data can lead to wrong results (more in my [report](REPORT.pdf)).

---

## How it works

Rebalancing is triggered by date at **every calendar quarter end** (March 31, June 30, Sept 30, Dec 31), not on individual earnings release days. On each quarter-end date, for each sector (Part A) or for the top 500 (Part B):

1. **Find the stocks** trading on that quarter-end date, using each company's latest financial report whose publish date was on or before that date.
2. **Filter:** apply that stage's rules — basic checks (known data, positive equity, price ≥ \$1), size floors (market cap, book value), value cutoffs (P/B), and profitability cutoffs (ROA, ROE), measured either across the whole pool or within each sector.
3. **Rank the survivors:** by cheapest P/B (raw or sector-normalized), or in Part A's control runs by biggest market cap or biggest book value.
4. **Buy the top K** (5, 10, 50 or 100) with equal weight and hold until the next quarter end. If a stock stops trading during the quarter, it's sold at its last price.

## Notebook layout

1. Load the data from SimFin + data cleanup
2. Data sanity checks (publish dates, when the backtest can start)
3. How the backtest works (selectors, backtest engine, metrics)
- **Part A:** sector by sector, stages 1–5
- **Part B:** top 500 vs. SPY: the trade-ready table, the filter ladder, the weed-out test, what each version bought
- Overall comparison of all techniques + logistic regression on the 660 Part A runs
- Appendix: test run and metric check vs. QuantStats; **Appendix B:** data audits; spike and CODQL deep dives; Part B grid search; report tables

## Repository

```text
├── PB_index_backtest.ipynb   # The full analysis: code, tables, charts, and notes
├── REPORT.pdf                # The full write-up: the journey and deep dives
└── README.md
```

## How to run

1. Get a free API key from [SimFin](https://www.simfin.com/).
2. Upload `PB_index_backtest.ipynb` to [Google Colab](https://colab.research.google.com/).
3. Add your key as a Colab secret named `SIMFIN_API_KEY` (the key icon in the left sidebar).
4. Choose **Runtime → Run all**.

---

## Data

| | Source | Range |
| --- | --- | --- |
| Quarterly balance sheets and income statements, sector labels | [SimFin](https://www.simfin.com/) free API | last 5 years |
| Daily stock prices (for returns and market cap) | SimFin | Oct 2020 – Oct 2025 |
| Benchmark ETF prices (SPY and 11 sector ETFs) | Yahoo Finance ([yfinance](https://pypi.org/project/yfinance/)) | same period |

- **Backtest window:** the first basket is picked on March 31, 2021, and the last return ends on September 30, 2025 (18 quarterly returns, limited by SimFin's free 5-year data). It starts in 2021 because on December 31, 2020 only 5% of companies had published numbers in the data, vs. 74% by March 31, 2021.
- **No looking into the future:** on each quarter-end rebalance date, a company's report is only used if it was already published on or before that date (companies published a median of 39 days after their fiscal quarter ended).
- **Missing data:** of 6,553 companies, 2,811 (43%) have no financial statements and 665 (10%) have no prices. These can never be picked.

**Two kinds of data traps, and how I handled them:**

| Kind of trap | Issue & example | What I did |
| --- | --- | --- |
| **1. Real market extremes** (valid numbers that break simple ratios or small baskets) | **Penny-stock spikes:** CODQL jumped from \$0.01 to \$0.95 in one quarter (+9,400%) and made a 5-stock basket look like a 41.50%/yr winner<br>**"Zombie" companies:** 643 companies had negative income *and* negative equity, making ROE look positive (`- ÷ - = +`) | **Strategy filters:** require positive equity, a \$1 price floor, and minimum company size (from Stage 2 on) |
| **2. Corrupted vendor numbers** (wrong units or bad prints in the raw files) | **Share counts ~1,000× too high:** Dolby showed up at \$7.56T market cap (really ~\$7.5B)<br>**Balance-sheet outliers:** Ampio's assets were 700–800× too high<br>**Bad daily price prints:** one bad print for Eargo created a fake +2,622% quarter | **Pre-backtest cleanup:** time-series baseline check (removed 592 of 47,537 share counts, 1.2%, and a handful of balance-sheet rows) + 11-day median price-blip filter (removed 633 of 6.16M prices, 0.01%) |

As a final check, 99.8% of 11,819 stock-quarters matched Yahoo Finance within 1%. The full story of how I found and fixed these is in the [report](REPORT.pdf), and all the checks are in Appendix B of the notebook.

---

## Results

**Using CAGR to compare ideas:** every strategy and benchmark is compared by **CAGR** (compound annual growth rate), the average % a year the money grew over the whole test. A strategy "beats" its benchmark if its CAGR is higher over the same 18 quarters. The "gap" is the difference in CAGR, in percentage points a year. Returns are in %, not dollars, and every basket is equal-weighted.

### Part A: sector by sector (5 filter stages × 11 sectors × 4 basket sizes × 3 rankings = 132 runs per stage, 660 runs total)

| Stage | Beat sector ETF (CAGR) | Median CAGR gap vs. ETF | K = 10 median volatility (ETF: 17.23%) | K = 10 median max drawdown (ETF: -21.12%) |
| --- | --- | --- | --- | --- |
| 1. Pure P/B (cheapest stocks, no filters) | 35% | -1.82 pts/yr | 77.02% | -41.86% |
| 2. Clean P/B (market cap ≥ 25th pct, book value ≥ 50th pct, P/B ≤ 50th pct, price ≥ \$1) | 23% | -5.96 pts/yr | 24.84% | -38.06% |
| 3. Clean + ROA ≥ 0.1% | 13% | -5.98 pts/yr | 19.76% | -29.32% |
| 4. Clean + ROE ≥ 0.1% | 12% | -5.59 pts/yr | 20.14% | -34.28% |
| 5. Clean + ROA + ROE | same as stage 3 | | | |

The 3 rankings in each stage are cheapest P/B plus two control rankings (biggest market cap and biggest book value). As the filters tighten across the 5 stages, fewer stocks pass, so the larger basket sizes (K = 50 and K = 100) stop making a difference and end up holding the exact same stocks in most sectors. The average sector ETF's CAGR was 9.62% a year (median 9.09%, median volatility 17.23%, median max drawdown -21.12%). Each filter stage cut volatility and max drawdown relative to Pure P/B (and by Stage 3, 46% of the 132 runs had lower volatility than their sector ETF and 27% had a smaller max drawdown), though a 10-stock sector basket was still usually bumpier than the full sector ETF. Some parameter settings outperformed their sector index (for example in Basic Materials and Energy), but with only 18 quarterly samples the confidence in any single setting was low. Stage 5 is identical to stage 3: with positive equity, ROE is always at least as big as ROA, so the extra check removes nothing.

### Which variables predicted beating the sector ETF? (Logistic regression on the 660 runs)

Fitting a logistic regression across all 660 Part A runs ($R^2 = 54.1\%$) shows that **Sector** explained almost everything ($R^2 = 42.9\%$: Basic Materials, Industrials, and Energy accounted for 73% of all wins), while **Filter Stage** ($R^2 = 5.1\%$), **Basket Size** ($R^2 = 2.0\%$), and **Ranking Metric** ($R^2 = 0.2\%$) barely mattered. The [report](REPORT.pdf) has the full sector-by-sector tables and details.

### Part B: top 500 vs. SPY (the "filter ladder", 50-stock baskets)

Because financial ratios vary a lot across industries (for example, Tech companies naturally have higher P/B and ROA than Utilities), each step in this cross-sector portfolio is tested two ways: **Raw** (one cutoff across all 500 stocks) and **Sector-normalized** (each stock's P/B, ROA and ROE are ranked only against other top-500 stocks in its own sector).

| Step | Filter added (cumulative) | Stocks passing | CAGR (Raw / Norm) | Volatility (Raw / Norm) | Max Drawdown (Raw / Norm) |
| --- | --- | --- | --- | --- | --- |
| *SPY benchmark* | *market-cap weighted index* | *500* | *13.81%* | *15.11%* | *-23.93%* |
| *All 500 (equal weight)* | *no filters* | *500* | *7.55%* | *15.41%* | *-26.16%* |
| 1. Pure P/B (with price ≥ \$1) | positive equity, P/B known, price ≥ \$1 | 469 | 10.62% / 7.86% | **13.30%** / **14.62%** | **-19.15%** / **-20.75%** |
| 2. + P/B ≤ 50th pct | cheaper half by P/B | 219 | 10.62% / 7.86% | **13.30%** / **14.62%** | **-19.15%** / **-20.75%** |
| 3. + ROA ≥ 0.1% | profitable companies | 187 | 9.02% / 5.32% | **13.43%** / **13.81%** | **-20.43%** / **-20.66%** |
| 5. + Size filters (= Part A Stage 3/5) | market cap ≥ 25th pct, book value ≥ 50th pct ("Clean + ROA/ROE") | 122 | 9.81% / 7.17% | **12.68%** / **13.67%** | **-16.79%** / **-19.75%** |
| 6. + ROA & ROE ≥ 75th pct | top 25% most profitable | 7 | **14.20%** / **25.93%** | 20.49% / 20.78% | **-21.65%** / **-22.19%** |
| 7. + P/B ≤ 25th pct (= initial strict) | cheapest 25% by P/B | 2 | **21.07%** / 2.69% | 35.87% / 26.62% | -33.75% / -45.04% |

***Bold** = better than SPY on that metric (higher CAGR, lower volatility, or smaller max drawdown). Step 4 (+ ROE ≥ 0.1%) changes nothing, for the same reason as Part A stage 5.*

### Grid search: which filter matters more? (192 portfolios; average CAGR of 50-stock baskets for each P/B × ROA combination)

| P/B filter \ Profit filter | none | ROA ≥ 0.1% | ROA ≥ 50th pct | ROA ≥ 75th pct | Row average |
| --- | --- | --- | --- | --- | --- |
| **none** | 9.15% | 7.83% | 11.07% | **14.22%** | 10.57% |
| **P/B ≤ 75th pct** | 9.15% | 7.83% | 11.07% | **14.16%** | 10.55% |
| **P/B ≤ 50th pct** | 9.15% | 7.83% | 10.95% | **15.96%** | 10.97% |
| **P/B ≤ 25th pct** | 9.27% | 7.73% | **14.88%** | **20.25%** | 13.03% |
| **Column average** | 9.18% | 7.81% | 11.99% | **16.15%** | 11.28% |

*Each cell averages 4 portfolios (raw vs. sector-normalized cutoffs × size filters on/off). **Bold** = higher than SPY (13.81% a year).*

## Key findings

- **Pure P/B mostly bought penny stocks and failing companies.** Its best results came from one-off spikes in tiny penny stocks (like CODQL's single-quarter +9,400% jump), where a single trade skewed the whole multi-year CAGR by random chance.
- **Removing tiny companies made results realistic, and worse.** The crashes got smaller, but so did the wins that came from random chance.
- **The weeding-out filters (Steps 1–5) did cut downside and volatility against SPY, at the cost of lower returns.** In the 50-stock top-500 baskets, Steps 1–5 all had lower volatility (12.68%–14.62% vs. SPY's 15.11%) and smaller max drawdowns (-16.79% to -20.75% vs. SPY's -23.93% and equal-weight's -26.16%), with Step 5 (+ Size filters) cutting the worst drop to -16.79% raw (-19.75% normalized). In Part A, the filters also cut K = 10 median volatility from 77.02% to 19.76% and median max drawdown from -41.86% to -29.32%.
- **Being *among the most* profitable is what raised returns.** Sector-normalized, the top-25%-quality basket (Step 6) made 25.93% a year vs. SPY's 13.81% (Sharpe 1.15 vs. 0.78, max drawdown -22.19% vs. -23.93%). But it held only about 7 stocks.
- **The profit filter mattered about 3× more than the P/B filter.** Moving across the columns of the grid (from ROA ≥ 0.1% to ROA ≥ 75th pct) raised average CAGR by about 8 points a year (7.81% → 16.15%), while moving down the P/B rows raised it by about 2.5 points (10.55% → 13.03%).
- **My strictest initial filters (Step 7) were too strict.** Because they left only about 2 stocks on average (none in some quarters), concentration drove max drawdown up to -33.75% (raw) and -45.04% (normalized), and their return is mostly random chance.
- **Equal weight lost to SPY by about 6 points a year.** In 2021–2025, a few giant companies drove the market.

**Conclusion:** Weeding out bad stocks (Steps 1–5) reduced max drawdown and volatility below SPY, but didn't beat it on return. Quality did most of the work when returns did beat SPY, and cheapness added little. Because the only return winners were tiny baskets over just 18 quarters, I don't trust them enough to invest in them. For now, the index still wins.

The [report](REPORT.pdf) has the full story: each stage, what I learned from it, and the deep dives behind these numbers.

---

## Limitations

- **Short test period.** Only 18 quarters (2021–2025), because of SimFin's free data limits.
- **Small baskets.** The only strategies that beat SPY held about 7 stocks or fewer, sometimes just 1 or 2.
- **Missing data.** About 43% of companies have no financial statements in the free data. That could make the strategy look safer than it is.
- **No trading costs or taxes.** Trades happen at the closing price for free.
- **Picked from many tries.** The best grid setting was the best of 192 tries, so I use the grid to see patterns, not to pick a winner.
- **Top 500 is a stand-in** for the S&P 500, not the real list.

## Versions

- **v1 (2026):** original analysis and report. See the `v1-original-report` tag.
- **v2 (Sept 2026):** I used an AI assistant (Gemini) to help with debugging and editing. The numbers changed a lot, partly because SimFin's free window moved, but the direction of my v1 lessons held up (watch out for value traps, penny stocks and over-filtering). Changes:
  - on each quarter-end rebalance date, each company's report is used only if its publish date was on or before that date (instead of a fixed 3-month delay)
  - all the size and P/B filters are now really applied, and the notebook counts what each one removes
  - stocks that stop trading are sold at their last price, instead of sitting in the data with a frozen price
  - CAGR counts every quarter
  - a data-cleanup step for bad share counts, other numbers and price blips
  - Part B rebuilt as a filter ladder, plus a weed-out test and a grid search
