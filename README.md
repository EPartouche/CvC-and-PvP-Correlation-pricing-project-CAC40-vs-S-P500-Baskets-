# CvC/PvP-Correlation pricing project: CAC40 vs S&P500 Baskets
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

**CAC 40 STOCKS**

**Markets forcasts**

***Tech***
***Capgemini***

<img width="957" height="283" alt="image" src="https://github.com/user-attachments/assets/07ed3a84-83fe-4ce1-bbd1-0739f3272a33" />

***Dassault Systemes***

<img width="958" height="272" alt="image" src="https://github.com/user-attachments/assets/8d2e7171-84d2-44f7-848b-74dc7849e5fa" />

***STMicroelectronics***

<img width="946" height="272" alt="image" src="https://github.com/user-attachments/assets/8648bff1-ee6f-4e5c-b915-b09a9343ae21" />

***Energy***
***Total Energies***

<img width="958" height="287" alt="image" src="https://github.com/user-attachments/assets/e9467f04-225d-43e9-855d-0443490e9ba6" />

***Engie***

<img width="948" height="271" alt="image" src="https://github.com/user-attachments/assets/d0fdb4a1-c7d9-4ecd-953d-3144bc43af67" />

***Luxury***
***LVMH***

<img width="952" height="273" alt="image" src="https://github.com/user-attachments/assets/427fe35b-5f91-4701-895f-f982b41caa2b" />

***Hermes***

<img width="949" height="268" alt="image" src="https://github.com/user-attachments/assets/f6ff4777-9e78-44ae-9b63-f13869e32f66" />

***Kering***

<img width="947" height="261" alt="image" src="https://github.com/user-attachments/assets/e2940670-16f4-40f8-9919-55287a262aba" />

***L'Oréal***

<img width="950" height="265" alt="image" src="https://github.com/user-attachments/assets/bd953206-b28c-4a94-944c-ea869c5593fc" />


Before analyzing correlation or pricing any option, we first look at how each CAC40 sector basket actually performed on a standalone basis over the past 5 years. This chart sets the baseline: it shows CAC Tech's clear outperformance against the tighter, shared trajectory of CAC Energy and CAC Luxury, which we'll revisit once we get into correlation.


<img width="1917" height="1016" alt="01_cac40_baskets" src="https://github.com/user-attachments/assets/2e7fd2a2-c722-4c6e-98e9-ef093f6bd03a" />

*Figure 1 — Cumulative performance (base 100) of the three CAC40 sector baskets over the 5-year window.*


**S&P 500**
**Markets forcasts**

***Tech***
***Nvidia***

<img width="953" height="275" alt="image" src="https://github.com/user-attachments/assets/559d5ff7-f7c8-45af-bee8-98272e61e7e2" />

***Microsoft***

<img width="961" height="275" alt="image" src="https://github.com/user-attachments/assets/6718b7e8-aca7-4fa5-a157-55cbc9dd88cb" />

***Alphabet***

<img width="941" height="271" alt="image" src="https://github.com/user-attachments/assets/8d1e65a6-743c-4c07-bdfa-a5ef117bc36c" />

***Energy***
***ExxonMobil***

<img width="957" height="271" alt="image" src="https://github.com/user-attachments/assets/2f5d4f8a-f6d4-421f-b9d1-25b10ac19696" />

***Chevron***

<img width="958" height="275" alt="image" src="https://github.com/user-attachments/assets/8439a170-7ac2-4d86-b589-738499535f71" />

***Medias***
***Walt Disney Corp.***

<img width="946" height="273" alt="image" src="https://github.com/user-attachments/assets/709be6e4-f839-47ad-88b6-f4754ba0df43" />

***Comcast***

<img width="954" height="269" alt="image" src="https://github.com/user-attachments/assets/89df5c8e-43bb-4d28-a6e1-5d7260daf247" />

***Netflix***

<img width="953" height="269" alt="image" src="https://github.com/user-attachments/assets/d52ad3ae-ef8f-42c4-b21e-ac62df9ab5d6" />

***Warner Bros Discovery Inc.***

<img width="953" height="269" alt="image" src="https://github.com/user-attachments/assets/fd1d145a-1c11-48ab-a9c1-37f87fd2b8d9" />


<img width="1917" height="1016" alt="02_sp500_baskets" src="https://github.com/user-attachments/assets/053f72e6-0a26-430e-991c-2dfd9c30a90d" />

*Figure 2 — Cumulative performance (base 100) of the three S&P500 sector baskets over the same window.*

<img width="1917" height="1016" alt="03_sector_benchmarks" src="https://github.com/user-attachments/assets/2c4eab25-a1f8-4d38-a442-f12cef81e7d4" />

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

<img width="1898" height="1572" alt="04_global_correlation_heatmap" src="https://github.com/user-attachments/assets/0471030e-a6b3-431e-9170-c2c957011614" />

*Figure 4 — Full 10×10 correlation matrix across all 6 baskets and 4 sector benchmarks. This is the single most information-dense chart of the project: read row by row, it shows exactly how much diversification (or lack thereof) exists between every pair of instruments used downstream.*

<img width="2097" height="1016" alt="05_rolling_correlation_trades" src="https://github.com/user-attachments/assets/deef125a-964b-4c67-881f-28c7c53fb40a" />

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

<img width="2456" height="965" alt="06_01_payoff_CAC_Tech_vs_SP_Tech" src="https://github.com/user-attachments/assets/8e270ff3-8e87-4721-9622-ea97b347613d" />

*Figure 6a — Left: scatter of the 60,000 simulated (basket A, basket B) terminal performances, with the in-the-money region highlighted. Right: the resulting CvC payoff profile as a function of R_A − R_B.*

<img width="2457" height="965" alt="06_02_payoff_CAC_Energy_vs_SP_Ene" src="https://github.com/user-attachments/assets/19f6fcc2-21ec-4c54-a8f7-ee32f9b93042" />

*Figure 6b — Same analysis for the Energy pair. Note the tighter scatter cloud relative to Figure 6a, consistent with Energy's slightly higher inter-basket correlation (0.44 vs 0.39).*

<img width="2455" height="965" alt="06_03_payoff_CAC_Luxury_vs_SP_Med" src="https://github.com/user-attachments/assets/62cd9769-1ce2-4341-b122-014496be5b3c" />

*Figure 6c — Same analysis for the Luxury/Media pair, over its longer 1.5-year maturity.*

<img width="1917" height="1013" alt="07_01_convergence_CAC_Tech_vs_SP_Tech" src="https://github.com/user-attachments/assets/18e5dc8f-c99d-4f14-9aa0-37dc48f15923" />

*Figure 7a — Running mean of the discounted payoff (with 95% confidence band) as the number of simulated paths grows, for the Tech trade. The estimate stabilizes well before 60,000 paths, confirming the simulation is adequately converged.*

<img width="1917" height="1013" alt="07_02_convergence_CAC_Energy_vs_SP_Ene" src="https://github.com/user-attachments/assets/2d7732ba-9b9c-4a52-8a88-aa272d50a2b9" />

*Figure 7b — Same convergence diagnostic for the Energy trade.*

<img width="1917" height="1013" alt="07_03_convergence_CAC_Luxury_vs_SP_Med" src="https://github.com/user-attachments/assets/0b253158-81f6-4e9a-8629-48482df84d03" />

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

<img width="1736" height="1013" alt="08_01_correlation_sensitivity_CAC_Tech_vs_SP_Tech" src="https://github.com/user-attachments/assets/b25a26a2-a0d2-40f1-9541-176c3aec7b78" />

*Figure 8a — Price of the CAC Tech vs SP Tech CvC option as a function of imposed inter-basket correlation, with the current estimated historical correlation (0.39) marked as a vertical reference line. This is the "correlation vega" curve — the defining risk profile of the entire product family.*

<img width="1736" height="1013" alt="08_02_correlation_sensitivity_CAC_Energy_vs_SP_Ene" src="https://github.com/user-attachments/assets/bf12a3f6-c59c-4bc4-bf8e-01d45bd7591d" />

*Figure 8b — Same analysis for the Energy pair, showing the steepest proportional price decay of the three trades.*

<img width="1736" height="1013" alt="08_03_correlation_sensitivity_CAC_Luxury_vs_SP_Med" src="https://github.com/user-attachments/assets/2f6e2ecc-84cb-4634-88a7-e0ddb9e88c77" />

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

<img width="2389" height="1016" alt="09_portfolio_composition" src="https://github.com/user-attachments/assets/c2b2ed66-93f8-4aff-8f5f-9daa567abb6a" />

*Figure 9 — Left: mark-to-model value of each of the three trades. Right: notional allocation across the book (37% Tech, 30% Energy, 33% Luxury/Media by notional).*

<img width="1917" height="1015" alt="10_portfolio_pnl_distribution" src="https://github.com/user-attachments/assets/eba62092-9fbd-4a49-8af1-1dcc7c3ad2a3" />

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

<img width="1737" height="1016" alt="11_basket_vs_sector_correlation" src="https://github.com/user-attachments/assets/7f921355-d140-45e0-84b3-fec1de6fb256" />

*Figure 11 — All six baskets ranked by their correlation to their respective sector benchmark. S&P Energy leads the book; CAC Energy and CAC Tech bring up the rear.*

<img width="2459" height="1043" alt="12_01_CAC_Tech_vs_sector" src="https://github.com/user-attachments/assets/16081c38-6b9c-4518-8566-353c522a5cf5" />

*Figure 12a — CAC Tech cumulative performance against MSCI World IT (left), and the underlying daily-return scatter with fitted beta (right).*

<img width="2459" height="1043" alt="12_02_S P_Tech_vs_sector" src="https://github.com/user-attachments/assets/c393f653-f073-4e3e-966c-64201334d882" />

*Figure 12b — Same analysis for S&P Tech — note the visibly tighter return scatter versus Figure 12a, consistent with its higher correlation and beta.*

<img width="2459" height="1043" alt="12_03_CAC_Energy_vs_sector" src="https://github.com/user-attachments/assets/40343a84-2821-4098-be97-1826b545bb50" />

*Figure 12c — CAC Energy against MSCI World Energy.*

<img width="2459" height="1043" alt="12_04_S P_Energy_vs_sector" src="https://github.com/user-attachments/assets/0624e2df-a127-49e5-b568-32364feb6078" />

*Figure 12d — S&P Energy against MSCI World Energy — the tightest-tracking pair in the entire dataset (ρ = 0.727).*

<img width="2459" height="1043" alt="12_05_CAC_Luxury_vs_sector" src="https://github.com/user-attachments/assets/fdc773b3-75ae-4ff8-9663-238f35ea3d99" />

*Figure 12e — CAC Luxury against MSCI World Luxury/Consumer Discretionary.*

<img width="2459" height="1043" alt="12_06_S P_Media_vs_sector" src="https://github.com/user-attachments/assets/550f3e17-578f-4fe7-bf56-d2162f718b05" />

*Figure 12f — S&P Media against MSCI World Communication Services.*

<img width="2097" height="1016" alt="13_rolling_correlation_sectors" src="https://github.com/user-attachments/assets/395f684d-a28f-4b2c-82a7-fcc2b381eddc" />

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
