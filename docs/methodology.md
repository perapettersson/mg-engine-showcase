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

Across those 20 anchors the aggregate outcome is a **net loss after cost**, with a pooled hit rate of
**0.438**. Individual anchors vary widely — which is precisely why no single one is quoted anywhere in
this document.

## 2. Walk-forward (in-sample / out-of-sample)

Each anchor's trades are split chronologically into an in-sample (IS) and out-of-sample (OOS)
segment. Comparing IS vs OOS expectancy, profit factor and hit rate exposes overfitting: a strategy
that only works in-sample shows OOS degradation. The cross-anchor distribution of the IS→OOS ratio
is the honest read (per-anchor OOS samples are individually noisy).

The result is the opposite of overfitting. Out-of-sample expectancy came out **higher** than in-sample
(**+0.122 vs +0.067**), hit rate rose from **44.5% to 50.2%**, profit factor from **1.20 to 1.70**, and
half the anchors show positive OOS expectancy. The engine is not fitted to its sample. It simply has no
edge to fit.

## 3. Bootstrap confidence intervals

Trade-level bootstrap resampling (B = 1000) produces 95% confidence intervals on expectancy, profit
factor and payoff ratio — both within each anchor and across anchors. This distinguishes a *reliable*
point estimate (narrow CI excluding the break-even line) from an *uncertain* one (wide CI including
it). It is the single most important guard against reading noise as edge.

The intervals are the honest headline of this project. Across anchors, expectancy per trade comes out at
**[-0.009, +0.212]** — it **spans zero**, so the direction of the edge is not established. Profit factor
lands at **[1.019, 1.528]** and hit rate at **[0.438, 0.484]**: gross, something is measurably there;
net, it is not enough to call an edge. Published as measured, unflattering end included.

## 4. Monte-Carlo trade-order

The realized trade sequence is reshuffled (M = 10000) to build a drawdown distribution and a
capital-efficiency profile (drawdown-to-PnL ratio). This answers two questions a single equity curve
cannot: how lucky was the historical ordering, and how much capital buffer would a live deployment
actually require.

The realized ordering turns out to be unremarkable: its drawdown sits at the **47.7th percentile** of the
reshuffled distribution, so the historical path was neither lucky nor unlucky. Capital efficiency is poor
— drawdown-to-P&L ≈ **1.95**, and the 99th-percentile reshuffled drawdown is **49 index points**, implying
a buffer several times the expected P&L for any live deployment.

## 5. Cost-friction analysis

Execution costs are modelled as a **single per-fill cost constant, scaled by recent volatility**
(higher recent volatility ⇒ wider effective spread). No separate market-impact term is active: at the
position sizes this engine actually trades (order clip ≈ 1 contract) impact is negligible and the
cost should collapse to the half-spread — which is what the model asserts.

That assertion was **checked, not assumed** — and the check was then re-run, and its first answer
superseded. An initial Roll-estimator half-spread over 20 intraday sessions came out at **0.705
points**, which made the 0.75 constant look accurate to within 6%. A later session-restricted
per-day estimate across a full month of tape puts the median at **0.969 points** — so the constant
is roughly **30% too low**, not marginally too high. The earlier figure stays on the record; being
superseded by a better measurement is the point.

This strengthens the verdict rather than weakening it. The engine was charged *less* per fill than
the tape says it would have paid, so the reported net loss is a real cost that is if anything
understated — not an artifact of a pessimistic cost model.

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
