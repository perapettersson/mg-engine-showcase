# Validation methodology

A single backtest proves nothing. The engine is evaluated the way a research desk evaluates a
candidate strategy — across regimes, out-of-sample, with confidence intervals and order-randomized
drawdowns, and with an explicit cost model. The guiding rule throughout: **measure honestly, and
falsify your own hypotheses before the market does.**

## 1. Cross-anchor study

The engine is replayed against **20 anchors** spanning distinct 2024–2026 market regimes — trending
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

Execution costs are vol-scaled (an Almgren–Chriss-style model: higher recent volatility ⇒ wider
effective spread). Per-anchor break-even hit rate is computed from each anchor's own payoff ratio,
and the margin above/below break-even is reported. This separates a *cost-trapped* failure mode
(gross edge eaten by friction) from a *signal-trapped* one (no gross edge at all).

## 6. Falsification discipline

The project's defining habit is killing its own ideas. A multi-layer regime state-machine (an
8-state signal classifier, autocorrelation-adaptive payoff switching, regime-conditional direction
decisions) was designed, implemented behind audit-only flags, verified across the anchor set, and
**falsified** when it produced no net lift. It was reverted to audit-only rather than tuned to look
good on the sample. The same fate met a queue of signal-extraction ideas, each documented as tested
and dropped.

The conclusion the evidence supports — that the binding constraint is the **information content of
OHLC-only inputs**, not the modeling method — is itself a falsifiable claim, and the path to test it
(adding volume / order-flow / cross-asset information) is explicit.

---

**Why this matters.** The headline number is not "it makes money." The headline is a reproducible
research process that produces trustworthy conclusions — including negative ones — and never confuses
a good-looking sample with a real edge.
