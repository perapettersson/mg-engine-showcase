# Validation methodology

A single backtest proves nothing. The engine is evaluated the way a research desk evaluates a
candidate strategy — across regimes, out-of-sample, with confidence intervals and order-randomized
drawdowns, and with an explicit cost model. The guiding rule throughout: **measure honestly, and
falsify your own hypotheses before the market does.**

## 1. Cross-anchor study

The engine is replayed against **20 anchors** spanning distinct 2022–2024 market regimes — trending
up, trending down, central-bank (FOMC/ECB) windows, and stress events. Each anchor is an independent
full run in its own isolated worktree, dispatched in parallel. Per-anchor and aggregate KPIs are
collected: expectancy per trade, profit factor, payoff ratio, and hit rate (hit rate is treated as a
*diagnostic*, never the headline metric).

## 2. Walk-forward (in-sample / out-of-sample)

Each anchor's trades are split chronologically into an in-sample (IS) and out-of-sample (OOS)
segment. Comparing IS vs OOS expectancy, profit factor and hit rate exposes overfitting: a strategy
that only works in-sample shows OOS degradation. The cross-anchor distribution of the IS→OOS ratio
is the honest read (per-anchor OOS samples are individually noisy).

## 3. Bootstrap confidence intervals

Trade-level bootstrap resampling (B = 1000) produces 95% confidence intervals on expectancy, profit
factor and payoff ratio — both within each anchor and across anchors. This distinguishes a *reliable*
point estimate (narrow CI excluding the break-even line) from an *uncertain* one (wide CI including
it). It is the single most important guard against reading noise as edge.

## 4. Monte-Carlo trade-order

The realized trade sequence is reshuffled (M = 10000) to build a drawdown distribution and a
capital-efficiency profile (drawdown-to-PnL ratio). This answers two questions a single equity curve
cannot: how lucky was the historical ordering, and how much capital buffer would a live deployment
actually require.

## 5. Cost-friction analysis

Execution costs are modelled as a **single per-fill cost constant, scaled by recent volatility**
(higher recent volatility ⇒ wider effective spread). No separate market-impact term is active: at the
position sizes this engine actually trades (order clip ≈ 1 contract) impact is negligible and the
cost should collapse to the half-spread — which is what the model asserts.

That assertion was **checked, not assumed**. The constant (0.75 index points per fill) was validated
after the fact against exchange tape for the traded contract: a Roll-estimator half-spread measured
over 20 intraday sessions came out at **0.705 points** — the guess is right to within 6%. This is
load-bearing for the verdict: it means the reported net loss is a **real** cost, not an artifact of a
pessimistic cost model.

Per-anchor break-even hit rate is computed from each anchor's own payoff ratio, and the margin
above/below break-even is reported. This separates a *cost-trapped* failure mode (gross edge eaten by
friction) from a *signal-trapped* one (no gross edge at all).

## 6. Falsification discipline

The project's defining habit is killing its own ideas. A multi-layer regime state-machine (an
8-state signal classifier, autocorrelation-adaptive payoff switching, regime-conditional direction
decisions) was designed, implemented behind audit-only flags and verified across the anchor set —
then **falsified**. Activating its consumers did not merely fail to help: it cost **−155 net P&L**
against the baseline across the anchor set. It was reverted to audit-only rather than tuned until
the regression went away. The same fate met a queue of signal-extraction ideas, each documented as
tested and dropped.

## 7. Testing the ceiling claim itself

The conclusion the evidence supports — that the binding constraint is the **information content of
the inputs**, not the modeling method — is itself a falsifiable claim. And the uncomfortable
objection to it is obvious: *maybe the engine filtered out its own edge.* Three tests, in increasing
strength.

**Add information.** A **volume (OHLCV)** extension was wired in and cross-anchor-evaluated (N=20):
**no net lift**, profit-factor robustness regressed, reverted rather than kept. The ceiling held
under OHLCV too.

**Measure the information directly.** Rather than inferring a ceiling from P&L, mutual information
between candidate features and the forward outcome is estimated against a **circular-shift null**,
which preserves autocorrelation where a naive permutation would not. The best real feature clears
that null by roughly 0.003–0.009 bits; a planted positive control clears it by 0.017–0.025. The
instrument works — there is simply very little to measure.

**Test the gate, not the data — the positive control.** The sharpest objection is that a
conservative entry gate could be manufacturing the null by abstaining on real opportunities. So
inject a *known*, synthetic directional edge into the feed and watch what the engine's own phase
gate does with it. The gate is never told where the edge is; it has to find it. Result: the gate's
pass rate rose from **14.4% to 64.4%**, and trade count from **7 to 58** — it lights up roughly
4.5× when there is something to find. The gate is therefore a calibrated classifier rather than an
over-filter, and the abstention observed on real data is a **true negative**: correct silence on a
feed carrying no signal, not a missed one.

That distinction is the whole reason it was worth measuring, because the two diagnoses imply
opposite next moves. An over-filter would mean recalibrate the gate. A true negative means the gate
is fine and only richer *information* can move the result — so the follow-on work moved information,
not the model. Order-flow was the first channel tried and was itself probed to a null on real
exchange data; the open lever is crowd **positioning**.

---

**Why this matters.** The headline number is not "it makes money." The headline is a reproducible
research process that produces trustworthy conclusions — including negative ones — and never confuses
a good-looking sample with a real edge.
