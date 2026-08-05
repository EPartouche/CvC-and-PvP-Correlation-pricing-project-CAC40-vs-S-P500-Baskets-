# CvC-PvP-PRICING-PORTFOLIO-PROJECT----CAC40-vs-S-P500-BASKETS
Python project pricing CvC/PvP correlation options (outperformance/exchange options) on CAC40 vs S&amp;P500 equity baskets via Monte Carlo. Includes basket-by-basket correlation analysis, portfolio VaR, sector benchmark comparison (MSCI proxy), and ~23 auto-generated charts.


# CvC / PvP Correlation Options — Pricing & Portfolio Analysis: CAC40 vs S&P500 Sector Baskets

## Origin of the project

My first exposure to pure markets — after an earlier stint in asset management — came during an internship at **BGC Partners / Sunrise Brokers in London**, on the **equity exotic derivatives desk**. It was there that I first encountered a family of structures I had never seen in any textbook: **CvC ("Call vs Call")**, **PvP ("Put vs Put")**, **Quanto synthetic forwards**, and a handful of other correlation-driven products that brokers were pricing and flowing between desks every day.

At the time, I understood these products from the trading-floor side: who was asking for a quote, roughly which direction the flow was going, how brokers talked about them. But I didn't yet have a first-principles understanding of *why* these structures behaved the way they did — what was actually driving their price, and why a book of them could move violently even when the underlying markets themselves were calm. Of everything I saw on that desk, **CvC and PvP were by far the most "mainstream"** — the ones that came up again and again in the flow, as opposed to the more exotic one-off Quanto or barrier structures.

That gap stayed with me, and this project is my own investigation to close it: I wanted to build, from scratch, the full pricing and risk machinery behind a CvC/PvP book — not just to reproduce a formula, but to actually **see** the correlation risk that makes these products what they are, through simulation, sensitivity analysis, and portfolio-level risk metrics.

## What this project is

A complete Python pricing and portfolio-analysis engine for **CvC / PvP correlation options** — exchange options that pay out on the *relative* performance of one equity basket versus another — applied to three sector-matched pairs of CAC40 and S&P500 baskets: **Tech, Energy, and Luxury/Media**. The project prices each trade via Monte Carlo simulation, decomposes the correlation structure that drives its value, aggregates the trades into a portfolio with a 95% Value-at-Risk, and benchmarks every basket against a sector reference index (MSCI World proxy).

**The following elements represent only an extract from the completed project.** For the full, reproducible analysis, run `cvc_pvp_project.py` — every chart below is generated automatically and saved to `output/figures/`.

---

## Task 1 — Basket Construction & Market Data

The first task was to define six equity baskets, split into three sector-matched pairs designed to isolate a relative-performance ("correlation") view between the French and US markets:

| Trade | CAC40 basket | S&P500 basket |
|---|---|---|
| 1 | **CAC Tech** — Capgemini, Dassault Systèmes, STMicroelectronics | **S&P Tech** — Nvidia, Microsoft, Alphabet |
| 2 | **CAC Energy** — TotalEnergies, Engie | **S&P Energy** — ExxonMobil, Chevron |
| 3 | **CAC Luxury** — LVMH, Hermès, Kering, L'Oréal | **S&P Media** — Disney, Comcast, Netflix, Warner Bros Discovery |

Market data (5 years of daily prices) is sourced from Yahoo Finance where available, with an automatic fallback to a calibrated 3-factor market simulation (local market + common sector + idiosyncratic noise) when no data feed is available — ensuring the pipeline always runs end-to-end regardless of network access.

**Annualized volatility by basket:**

| Basket | Annualized vol |
|---|---|
| CAC Tech | ~29.4% |
| CAC Energy | ~28.8% |
| CAC Luxury | ~29.5% |
| S&P Tech | ~35.3% |
| S&P Energy | ~30.5% |
| S&P Media | ~29.8% |

The gap is immediately informative: **S&P Tech is the most volatile basket of the six by a clear margin (~35%)**, almost entirely driven by Nvidia's idiosyncratic volatility — a reminder that even a "sector-matched" pair can carry very different risk profiles on each side of the Atlantic before any correlation effect is even considered.

![CAC40 sector baskets — cumulative performance](output/figures/01_cac40_baskets.png)
*Figure 1 — Cumulative performance (base 100) of the three CAC40 sector baskets over the 5-year window.*

![S&P500 sector baskets — cumulative performance](output/figures/02_sp500_baskets.png)
*Figure 2 — Cumulative performance (base 100) of the three S&P500 sector baskets over the same window.*

![Sector reference indices](output/figures/03_sector_benchmarks.png)
*Figure 3 — The four MSCI World proxy sector indices (IT, Energy, Luxury/Consumer Discretionary, Communication Services) used later as reference benchmarks in Task 6.*

---

## Task 2 — Correlation Analysis, Basket by Basket

Correlation is the single risk factor that determines the value of a CvC/PvP trade, so the second task was a full correlation analysis across all six baskets and four sector benchmarks — both static (full-period) and 60-day rolling.

**Full correlation matrix — key readings:**

| Pair | Correlation |
|---|---|
| CAC Tech vs S&P Tech | **0.39** |
| CAC Energy vs S&P Energy | **0.44** |
| CAC Luxury vs S&P Media | **0.51** |
| Every basket vs its own MSCI sector benchmark | 0.51 – 0.73 |
| MSCI benchmarks vs each other | 0.83 – 0.86 |

Three things stand out:

1. **CAC Tech vs S&P Tech has the lowest correlation of the three pairs (0.39)** — meaning the Tech trade carries the "purest" correlation exposure of the book: the two baskets genuinely diverge more than the other pairs, which is exactly the environment in which a CvC/PvP structure earns its value.
2. **Every basket correlates more strongly with its own MSCI sector benchmark than with its transatlantic sector counterpart.** This confirms the existence of a real global sector factor (tech is tech, energy is energy, wherever you are), but the gap between "basket-to-benchmark" correlation (0.51–0.73) and "basket-to-basket" correlation (0.39–0.51) is exactly the geography-specific spread that a CvC/PvP trade is designed to monetize.
3. **The four MSCI benchmarks are almost interchangeable with each other (0.83–0.86 correlation)** — a useful sanity check confirming that, at the broad index level, sector distinctions matter far less than the common global equity factor.

![Global correlation heatmap](output/figures/04_global_correlation_heatmap.png)
*Figure 4 — Full 10×10 correlation matrix across all 6 baskets and 4 sector benchmarks. This is the single most information-dense chart of the project: read row by row, it shows exactly how much diversification (or lack thereof) exists between every pair of instruments used downstream.*

![Rolling correlation of trading pairs](output/figures/05_rolling_correlation_trades.png)
*Figure 5 — 60-day rolling correlation for each of the 3 trading pairs. Unlike the static matrix above, this shows the correlation *regime* is not stable through time — a CvC/PvP position is therefore exposed not just to a correlation level, but to correlation drifting up or down as market conditions change, which is the real-world risk a trading desk has to manage day to day.*

---

## Task 3 — Monte Carlo Pricing of CvC / PvP Options

The third task priced each of the 3 trades as an **exchange option**. Formally, letting R_A and R_B denote the terminal performance of baskets A and B:

- **CvC ("Call vs Call")** pays `N × max(R_A − R_B, 0)` — a long position on basket A's *relative* outperformance of basket B.
- **PvP ("Put vs Put")** pays `N × max(R_B − R_A, 0)` — the symmetric hedge, protecting against basket A's relative underperformance.

Since a basket is a weighted sum of lognormal assets, it is **not itself lognormal** — so no closed-form solution exists for the multi-asset case (the Margrabe formula is included in the code purely as a validation reference for the simplified 2-asset case). Pricing therefore uses a **correlated multi-asset Geometric Brownian Motion**, simulated via Cholesky decomposition of the empirical correlation matrix, with **60,000 paths** and **antithetic variates** for variance reduction.

**Pricing results:**

| Trade | Notional | Maturity | CvC price | 95% CI | CvC as % notional |
|---|---|---|---|---|---|
| CAC Tech vs SP Tech | €10,000,000 | 1.0y | **€1,305,041** | [€1,289,633 ; €1,320,450] | 13.05% |
| CAC Energy vs SP Energy | €8,000,000 | 1.0y | **€946,550** | [€934,966 ; €958,134] | 11.83% |
| CAC Luxury vs SP Media | €9,000,000 | 1.5y | **€1,265,978** | [€1,249,956 ; €1,282,000] | 14.07% |

Two observations worth flagging:

- **The Tech trade prices the richest in percentage terms among the two 1-year trades (13.05% vs 11.83%)** — directly consistent with Task 1's volatility numbers and Task 2's correlation numbers: S&P Tech's higher standalone volatility, combined with the lowest inter-basket correlation of the book, mechanically widens the distribution of R_A − R_B and therefore the option's value.
- **CvC and PvP prices are nearly identical for every pair** (e.g. €1,305,041 vs €1,302,946 for Tech, a 0.16% difference) — exactly what put/call correlation parity predicts when the underlying relative-performance distribution is close to symmetric around zero, and a useful internal consistency check on the pricing engine itself.

![Payoff diagram — CAC Tech vs SP Tech](output/figures/06_01_payoff_CAC_Tech_vs_SP_Tech.png)
*Figure 6a — Left: scatter of the 60,000 simulated (basket A, basket B) terminal performances, with the in-the-money region highlighted. Right: the resulting CvC payoff profile as a function of R_A − R_B.*

![Payoff diagram — CAC Energy vs SP Energy](output/figures/06_02_payoff_CAC_Energy_vs_SP_Ene.png)
*Figure 6b — Same analysis for the Energy pair. Note the tighter scatter cloud relative to Figure 6a, consistent with Energy's slightly higher inter-basket correlation (0.44 vs 0.39).*

![Payoff diagram — CAC Luxury vs SP Media](output/figures/06_03_payoff_CAC_Luxury_vs_SP_Med.png)
*Figure 6c — Same analysis for the Luxury/Media pair, over its longer 1.5-year maturity.*

![Monte Carlo convergence — CAC Tech vs SP Tech](output/figures/07_01_convergence_CAC_Tech_vs_SP_Tech.png)
*Figure 7a — Running mean of the discounted payoff (with 95% confidence band) as the number of simulated paths grows, for the Tech trade. The estimate stabilizes well before 60,000 paths, confirming the simulation is adequately converged.*

![Monte Carlo convergence — CAC Energy vs SP Energy](output/figures/07_02_convergence_CAC_Energy_vs_SP_Ene.png)
*Figure 7b — Same convergence diagnostic for the Energy trade.*

![Monte Carlo convergence — CAC Luxury vs SP Media](output/figures/07_03_convergence_CAC_Luxury_vs_SP_Med.png)
*Figure 7c — Same convergence diagnostic for the Luxury/Media trade.*

---

## Task 4 — Correlation Sensitivity ("Correlation Vega")

This is the task that most directly answers the question I originally walked away from Sunrise Brokers with: **what actually drives the P&L of a CvC/PvP book?** Each trade was re-priced under a grid of *imposed* correlation levels (from −0.30 to +0.95), holding each basket's individual volatility fixed, to isolate correlation as the sole variable.

| Trade | Price at ρ = 0 | Price at ρ = 0.9 | Estimated historical ρ | Price collapse (ρ 0 → 0.9) |
|---|---|---|---|---|
| CAC Tech vs SP Tech | €1,755,953 | €276,144 | 0.39 | **−84.3%** |
| CAC Energy vs SP Energy | €1,318,170 | €88,315 | 0.44 | **−93.3%** |
| CAC Luxury vs SP Media | €1,844,868 | €182,430 | 0.51 | **−90.1%** |

The pattern is unambiguous and consistent across all three trades: **the option's value collapses as correlation rises**, losing 84–93% of its zero-correlation value by the time correlation reaches 0.9. This is the mechanical consequence of the payoff structure itself — a higher correlation compresses the dispersion of relative performance (R_A − R_B) that the option depends on, so there is simply less room for one basket to meaningfully outrun the other.

This is also, in plain terms, exactly the risk a broker on that Sunrise Brokers desk was managing without necessarily framing it this way out loud: **selling a CvC or PvP structure is fundamentally a short-correlation position.** The seller profits if the two markets stay tightly linked (their book decays favorably as time passes with correlation stable or rising), and loses sharply if a correlation breakdown lets one basket run away from the other — which is precisely the kind of dislocation that tends to happen around idiosyncratic shocks (an earnings surprise, a regulatory shift, a single-name blow-up) rather than broad market moves.

**Energy shows the steepest relative collapse (−93.3%)** despite starting from the second-lowest correlation — a reminder that the shape of this curve, not just its starting point, matters for how a book should be hedged.

![Correlation sensitivity — CAC Tech vs SP Tech](output/figures/08_01_correlation_sensitivity_CAC_Tech_vs_SP_Tech.png)
*Figure 8a — Price of the CAC Tech vs SP Tech CvC option as a function of imposed inter-basket correlation, with the current estimated historical correlation (0.39) marked as a vertical reference line. This is the "correlation vega" curve — the defining risk profile of the entire product family.*

![Correlation sensitivity — CAC Energy vs SP Energy](output/figures/08_02_correlation_sensitivity_CAC_Energy_vs_SP_Ene.png)
*Figure 8b — Same analysis for the Energy pair, showing the steepest proportional price decay of the three trades.*

![Correlation sensitivity — CAC Luxury vs SP Media](output/figures/08_03_correlation_sensitivity_CAC_Luxury_vs_SP_Med.png)
*Figure 8c — Same analysis for the Luxury/Media pair, starting from the highest historical correlation (0.51) of the book.*

---

## Task 5 — Portfolio Aggregation & Value-at-Risk

The fifth task aggregated the three CvC trades into a single book, sized at a combined notional of **€27,000,000**, and computed a 95% Value-at-Risk via resampling of the underlying Monte Carlo payoff distributions.

| Metric | Value |
|---|---|
| Total portfolio value (mark-to-model) | **€3,517,570** |
| Combined notional | €27,000,000 |
| Portfolio value as % of notional | 13.03% |
| **VaR 95% (1-year horizon)** | **€3,517,570** |

The VaR figure is worth explaining rather than just reporting: it lands essentially equal to the full mark-to-model value of the portfolio. This is not a modeling error — it is the expected behavior of a **book of long option positions**. Since every trade payoff is floored at zero (`max(·, 0)`), there exists a plausible simulated scenario in the 5% tail where all three baskets underperform their pair in a way that leaves every option worthless at maturity. For a long-option book, the maximum realistic loss is therefore the full premium paid — which is exactly what the VaR captures here. Framed differently: **this portfolio's downside is fully known and bounded at inception (the price paid), while its upside is asymmetric and driven by exactly the correlation-breakdown scenarios discussed in Task 4.**

![Portfolio composition](output/figures/09_portfolio_composition.png)
*Figure 9 — Left: mark-to-model value of each of the three trades. Right: notional allocation across the book (37% Tech, 30% Energy, 33% Luxury/Media by notional).*

![Portfolio P&L distribution](output/figures/10_portfolio_pnl_distribution.png)
*Figure 10 — Full simulated distribution of portfolio value at maturity (60,000+ resampled scenarios), with the current mark-to-model value and the 95% VaR threshold overlaid. The distribution's right skew — a long tail of large gains against a hard floor near zero — is the visual signature of a long-option book.*

---

## Task 6 — Basket vs Sector Benchmark (MSCI World Proxy)

The final task compared each of the six baskets individually to its corresponding MSCI World sector proxy, to test which side of the Atlantic tracks its "pure" sector factor more closely — a question directly relevant to how well a CAC40 basket can be expected to hedge, or fail to hedge, its US counterpart via a common sector exposure.

| Basket | Benchmark | Correlation | Beta |
|---|---|---|---|
| CAC Tech | MSCI World IT | 0.523 | 0.574 |
| **S&P Tech** | MSCI World IT | **0.643** | **0.847** |
| CAC Energy | MSCI World Energy | 0.513 | 0.575 |
| **S&P Energy** | MSCI World Energy | **0.727** | **0.864** |
| **CAC Luxury** | MSCI World Luxury/ConsDiscr | **0.665** | 0.752 |
| S&P Media | MSCI World Communication Svc | 0.670 | 0.747 |

In both the Tech and Energy pairs, **the S&P500 basket tracks its sector benchmark noticeably more closely than the CAC40 counterpart** — both in correlation (0.643 vs 0.523 for Tech; 0.727 vs 0.513 for Energy) and in beta (0.847 vs 0.574; 0.864 vs 0.575). This is intuitive once you consider index construction: MSCI World sector indices are dominated by large-cap US constituents almost by definition, so a basket built from Nvidia, Microsoft and Alphabet is always going to look more like "the sector" than one built from Capgemini, Dassault Systèmes and STMicroelectronics — however well-chosen the latter names are within the French market.

The **Luxury/Media pair breaks this pattern**: CAC Luxury actually posts the *higher* correlation of the two (0.665 vs 0.670 is essentially a statistical tie, both leading the book alongside S&P Media) — a reminder that European Luxury is, unusually among sectors, a segment where French names are the global benchmark-setters rather than the followers.

![Basket vs sector correlation ranking](output/figures/11_basket_vs_sector_correlation.png)
*Figure 11 — All six baskets ranked by their correlation to their respective sector benchmark. S&P Energy leads the book; CAC Energy and CAC Tech bring up the rear.*

![CAC Tech vs sector](output/figures/12_01_CAC_Tech_vs_sector.png)
*Figure 12a — CAC Tech cumulative performance against MSCI World IT (left), and the underlying daily-return scatter with fitted beta (right).*

![S&P Tech vs sector](output/figures/12_02_S&P_Tech_vs_sector.png)
*Figure 12b — Same analysis for S&P Tech — note the visibly tighter return scatter versus Figure 12a, consistent with its higher correlation and beta.*

![CAC Energy vs sector](output/figures/12_03_CAC_Energy_vs_sector.png)
*Figure 12c — CAC Energy against MSCI World Energy.*

![S&P Energy vs sector](output/figures/12_04_S&P_Energy_vs_sector.png)
*Figure 12d — S&P Energy against MSCI World Energy — the tightest-tracking pair in the entire dataset (ρ = 0.727).*

![CAC Luxury vs sector](output/figures/12_05_CAC_Luxury_vs_sector.png)
*Figure 12e — CAC Luxury against MSCI World Luxury/Consumer Discretionary.*

![S&P Media vs sector](output/figures/12_06_S&P_Media_vs_sector.png)
*Figure 12f — S&P Media against MSCI World Communication Services.*

![Rolling correlation, baskets vs sector](output/figures/13_rolling_correlation_sectors.png)
*Figure 13 — 60-day rolling correlation of every basket against its sector benchmark. This is the closing chart of the project: it shows that even the strongest static relationships above (S&P Energy, CAC Luxury) are not constant through time, reinforcing Task 2 and Task 4's central message that correlation is a *regime*, not a fixed parameter — and that any CvC/PvP book has to be risk-managed with that instability explicitly in mind.*

---

## Key Takeaways

1. **Correlation, not direction, is the core risk of a CvC/PvP book.** All three trades exhibit the same qualitative correlation-vega profile: value falls by 84–93% as correlation rises from 0 to 0.9, regardless of sector.
2. **US sector baskets track their MSCI benchmark more tightly than their CAC40 counterparts** in Tech and Energy — a structural feature of MSCI World sector index construction (US large-cap dominated) rather than a market-specific anomaly. Luxury is the exception, where CAC leads.
3. **The CAC Luxury vs S&P Media pair, while the least "sector-matched" pairing on paper, carries the highest historical correlation (0.51) of the three trades** — showing that intuitive sector pairings don't always predict the strongest empirical correlation link, and that this kind of analysis should precede, not follow, trade structuring.
4. **A long CvC/PvP book has a VaR close to its full mark-to-model value**, consistent with the floored, option-like payoff structure of these products — the downside is bounded and known at inception, unlike the asymmetric upside that a correlation breakdown can unlock.
5. **Correlation is a regime, not a constant** — every rolling-correlation chart in this project (Figures 5 and 13) shows meaningful drift over the 5-year window, which is the real risk-management challenge behind the elegant static formulas.

This project closes the loop I opened on the Sunrise Brokers desk: I can now explain, from first principles and with my own simulation engine, exactly why a CvC or PvP quote moves the way it does — and, more usefully, exactly what a desk running a book of them needs to watch.

## How to Reproduce

```bash
pip install numpy pandas matplotlib seaborn scipy yfinance tabulate
python cvc_pvp_project.py
```

All 24 charts are regenerated in `output/figures/` and a full numerical report in `output/reports/report.md`.
