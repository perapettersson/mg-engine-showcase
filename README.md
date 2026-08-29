# MG Engine — Minority-Game Systematic Trading Research Platform

> A deterministic, replay-safe, bar-by-bar trading engine for equity-index futures,
> grounded in Minority Game theory (Challet · Marsili · Zhang · Coolen).
> ~60,000 lines of R in a strict single-writer architecture.

**Status:** research-grade · fully reproducible · honestly falsified
**Stack:** R · paradigm ports cleanly to Python · ~149 per-bar attribution surfaces

---

## What it is

A from-scratch systematic-trading research platform that turns Minority-Game (MG) statistical
mechanics into a complete per-bar decision pipeline. Each bar flows through seven layers —
**signal → edge validation → position sizing → governance → per-frequency intent → execution
latch → fill** — under a binding `decision(t) → execution(t+1)` timing law. Every state field has
**exactly one writer**, and the same configuration on the same data reproduces the same output,
bar for bar.

```
            DECISION CHAIN (bar t)                              CONSEQUENCE (t+1)

  05 MG → 06 EDGE → 07 RISK → 10 GOV → 11 TGT → 12 INT   ──►     13 EXEC
  signal   validate   size    allowed?  intent   latch          fill @ open(t+1)
```

## Architecture highlights

- **Single-writer discipline** — every surface has one owning module; state flows via committed
  surfaces, never side-stepped. A (projection) / B (selection) / C (commit) phase split per layer.
- **Strict t+1 execution timing** — intent decided at bar `t` executes at the open of bar `t+1`;
  no same-bar round-trips, no look-ahead.
- **Multi-frequency consensus** — independent MG agent populations run on prime-number timeframes
  (1m carrier + 7m/13m/19m voters) to avoid trader-cluster boundaries; execution is always on the
  1-minute carrier for fast-to-market entry.
- **Full attribution** — ~149 CSV surfaces capture why every bar did what it did, from raw MG
  signal through edge, risk, governance, intent and fill.
- **Determinism** — same config + same data = identical output; no RNG outside seeded agent init.

See [`docs/architecture.md`](docs/architecture.md).

## Theoretical basis

The signal layer emulates the speculative MG framework: N latent agents per timeframe each hold
S strategies mapping a market-history index `μ` to an action, with virtual-score learning and
grand-canonical abstention. Core order parameters are computed per bar:

- **H** — predictability of the price-sign history `(1/P) Σ⟨A⟩²_μ`
- **σ²** — aggregate volatility `Var(A)`
- **Δ(real − fake)** — payoff of the adaptive strategy vs a random baseline (the main edge detector)
- **score gap**, crowd concentration, frozen-fraction, susceptibility proxies

Implemented mechanics span Challet/Marsili/Zhang, Coolen (cavity / generating-functional analysis),
De Martino (mix-game), Marsili (producer–speculator), Sasidevan (stochastic strategies),
Kalinowski (multi-memory) and Ferreira–Marsili (autocorrelation-adaptive payoff).

## Validation & rigor

Strategy quality is measured the way a research desk would, not by a single backtest:

- **Cross-anchor study** — 20 anchors spanning 2022–2024 regimes (trend-up/down, FOMC, stress).
- **Walk-forward** in-sample / out-of-sample split per anchor.
- **Bootstrap confidence intervals** (B = 1000) on expectancy, profit factor, payoff ratio.
- **Monte-Carlo trade-order** shuffles (M = 10000) for drawdown distribution and capital efficiency.
- **Cost-friction analysis** with vol-scaled execution costs (Almgren–Chriss style).

See [`docs/methodology.md`](docs/methodology.md).

## Honest results

Across the cross-anchor study the engine is **research-grade**: a measurable gross signal exists,
but the *net* edge is marginal and **information-limited under OHLC inputs** — consistent with
the Glosten–Milgrom information ceiling. A **volume (OHLCV)** signal extension was wired in and
cross-anchor-tested (N=20) — it produced **no net lift** (the ceiling held under OHLCV too) and was
reverted, evidence the binding constraint is information *content*, not the OHLC/OHLCV distinction.
Several architectural hypotheses (an 8-state regime
classifier, dynamic payoff-mode switching) were implemented, tested, and **falsified**; each was
reverted rather than curve-fit to the sample. The deliverable is the **methodology and engineering
integrity** — deterministic replay, full attribution, and honest negative results — not an inflated
P&L claim.

## Companion study — why the verdict is about information, not method

A single falsified engine is a weak result: on its own it cannot separate *"my model was wrong"*
from *"there was nothing there to find."* The verdict above is stronger because a **second,
independent study** attacked the same question with a different model, a different market, and
strictly richer inputs — and returned the same null.

Both studies are placed on a shared coordinate system that separates **levers** (axes that change
what is knowable) from **labels** (axes that only change what it is called):

| Axis | Type | MG engine | Companion study |
|---|---|---|---|
| **info** | lever | OHLC(V) | OHLCV **+ 1D order flow** |
| **target** | lever | direction | direction |
| **horizon** | lever | 7–19 min | 1 h / 4 h |
| **construction** | lever | single-asset directional | single-asset directional |
| venue | gate | index future | spot |
| model | *label* | Minority Games (R) | 1D CNN (PyTorch) |
| asset | *label* | equity index | crypto |
| **Outcome** | | **falsified** | **falsified** |

Only the two **labels** differ. Every **lever** is identical or co-falsified — including `info`,
which the companion moved *upward* by adding order flow rather than sideways. It was built on a
self-maintained derivatives store (~30 pairs of 1-minute OHLCV plus funding, open-interest,
volatility, long/short-ratio and liquidation panels) and held to the same pre-committed
falsification protocol.

**Two model families, two asset classes, richer information in the second — the same null.**
That is the difference between *"my model did not work"* and a **measured information ceiling**:
the binding constraint sits in what price and order-flow data *contain* about this target at this
horizon, not in the machinery reading them. Swapping the model — MG → CNN → anything newer — moves
sideways along a label. It cannot lift a ceiling that is a property of the data.

### What the negative decided

The coordinate system is not a filing scheme; it is a decision rule. With `model` and `asset`
established as exhausted, the follow-on work moved a **lever** instead: same market, same data
store, a different **information channel** — from price and order flow to *positioning* (what the
crowd is holding, rather than the wake it leaves in the tape), which is also the regime Minority
Game theory actually describes. That study is open, not concluded.

Recording the negative precisely is what made the next question obvious.

## Engineering principles

- *"The logic carries the bot"* — no curve-fit thresholds, no sample-tuned weights, no side-disable
  filters. Every parameter is justifiable independently of the current sample.
- Single-writer state, deterministic replay, explicit precedence ordering, and architectural
  invariants enforced by assertions.

## Repository map

| Path | Contents |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | Pipeline, A/B/C phases, timing law, single-writer model |
| [`docs/methodology.md`](docs/methodology.md) | Cross-anchor / walk-forward / bootstrap / Monte-Carlo validation |
| [`src/`](src/) | Module taxonomy and design notes |

## About

A self-directed research project by **Per Pettersson**. This repository documents its architecture
and research methodology at a high level and contains no source code. A technical walk-through is
available on request — per.privat.pettersson@gmail.com.
