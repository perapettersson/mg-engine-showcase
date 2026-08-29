# MG Engine — Minority-Game Systematic Trading Research Platform

> A deterministic, replay-safe, bar-by-bar trading engine for equity-index futures,
> grounded in Minority Game theory (Challet · Marsili · Zhang · Coolen).
> ~60,000 lines of R in a strict single-writer architecture.

**Status:** research-grade · fully reproducible · honestly falsified
**Stack:** R reference · four verified ports (Python · C++ · C · Rust) · 163 attribution surfaces

---

## What it is

A from-scratch systematic-trading research platform that turns Minority-Game (MG) statistical
mechanics into a complete per-bar decision pipeline. Each bar flows through seven layers —
**signal → edge validation → position sizing → governance → per-frequency intent → execution
latch → fill** — under a binding `decision(t) → execution(t+1)` timing law. Every state field has
**exactly one writer**, and the same configuration on the same data reproduces the same output,
bar for bar.

Signal and execution instrument are deliberately separate: the engine reads a 1-minute cash-index
feed and is designed to trade the corresponding exchange-listed future — the standard
index-as-signal / future-as-trade split, and the reason the cost model is calibrated against
futures tape rather than against the index.

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
- **Multi-frequency agent populations** — independent MG populations run on prime-number timeframes
  (1m carrier + 7m/13m/19m deciders) to avoid trader-cluster boundaries, collapsed to one decision by
  either of two selectable rules: argmax single-winner, or k-of-N consensus among the higher
  timeframes. Execution always resolves to the 1-minute carrier — which never votes on direction.
- **Full attribution** — 163 CSV surfaces (52 exported state surfaces + 111 derived diagnostics)
  capture why every bar did what it did, from raw MG signal through edge, risk, governance, intent
  and fill.
- **Determinism** — same config + same data = identical output; no RNG outside seeded agent init.
- **Four parallel ports** — the engine is re-implemented in Python, C++, C and Rust at full 47/47
  file parity with the R reference, each producing the same 163-surface output. R is the single
  source of truth; ports are verified against it by static differential audit and output-surface
  parity, **not** by bit-identity — each seeds its own RNG, so cross-engine metric comparison is
  deliberately treated as uninterpretable rather than quietly reported as agreement.

See [`docs/architecture.md`](docs/architecture.md).

## Theoretical basis

The signal layer emulates the speculative MG framework: N latent agents per timeframe each hold
S strategies mapping a market-history index `μ` to an action, with virtual-score learning and a
grand-canonical-style abstention threshold (a static participation cutoff — not the full
opportunity-cost benchmark of the canonical formulation). Core order parameters are computed per bar:

- **H** — predictability of the price-sign history `(1/P) Σ⟨A⟩²_μ`
- **σ²** — aggregate volatility `Var(A)`
- **Δ(real − fake)** — the adaptive strategy's payoff against a random baseline. The canonical form
  is computed inside the latent agent ecology; the field the entry gate actually consumes is a
  cheaper composite proxy, labelled as such in the source rather than passed off as the canonical
  quantity
- **score gap**, crowd concentration, frozen-fraction, susceptibility proxies

Implemented mechanics are listed by **what actually runs**, not by what was coded — the distinction
is the point:

- **Live** — Challet/Marsili/Zhang (the canonical kernel: virtual-score learning, minority payoff,
  crowd/anti-crowd strategy pairing); De Martino (mix-game population split; continuous payoff form);
  Sasidevan (the stochastic *representation* only — not the paper's co-action equilibrium).
- **Audit-only** — Coolen (cavity phase-health, generating-functional history recurrence): computed
  every bar, gated off, zero decision impact. Ferreira–Marsili (autocorrelation-adaptive payoff): the
  metric runs; the regime switch it was built to drive was falsified and is dormant.
- **Implemented, then falsified and disabled** — producer–speculator injection (CMZ): hit rate fell
  63% → 41% cross-anchor, reverted, config-disabled. Mitman-Choe-Johnson heterogeneous memory depth:
  faithfully implemented, then configured inert once a memory-depth probe found no directional
  information at *any* depth (mutual information at or below the shuffled null, m = 1…8).

A per-reference provenance map — paper → mechanism → code site → honest status — is maintained
separately, specifically so that "implemented" is never allowed to blur into "load-bearing".

## Validation & rigor

Strategy quality is measured the way a research desk would, not by a single backtest:

- **Cross-anchor study** — 20 anchors spanning 2022–2024 regimes (trend-up/down, FOMC, stress).
- **Walk-forward** in-sample / out-of-sample split per anchor.
- **Bootstrap confidence intervals** (B = 1000) on expectancy, profit factor, payoff ratio.
- **Monte-Carlo trade-order** shuffles (M = 10000) for drawdown distribution and capital efficiency.
- **Cost-friction analysis** — a single vol-scaled per-fill cost constant, validated after the fact
  against exchange tape (0.75 points assumed vs a 0.705-point half-spread measured over 20 days).

See [`docs/methodology.md`](docs/methodology.md).

## Research protocol

The validation above is the *measurement*. The protocol is what decides whether a measurement is
allowed to count — and it is the part that took longest to build.

**Nothing is built until a probe survives.** A coordinate — a point in a typed axis-space of
information, target, horizon, construction and venue — is falsified *cheaply*, before any engine
exists. The sequence is fixed:

```
GATE -1  point-in-time      is the data legitimate to measure on? (knowledge-time, not event-time)
GATE  0  coordinate         does ≥1 causal axis actually move, or is this a relabel?
GATE  1  pre-commitment     kill-lines + trial count written down BEFORE any data is fetched
GATE  2  cheap measurement  rank-IC on data already held — no pipeline, no engine
GATE  3  multiple testing   deflated Sharpe · PBO via CSCV · FWER/FDR · minimum track record
GATE  4  robustness         purged + embargoed CV · realistic cost · decay
GATE  5  verdict            survive all → only now build. Any kill → record it, move a lever.
GATE  6  survivor → paper   promotion AND demotion lines pre-registered before paper trading starts
```

**Pre-registration is enforced by the tooling, not by good intentions.** The statistics harness
refuses to emit a verdict unless a pre-registration file already exists on disk. Result-shopping is
impossible by construction rather than discouraged by policy. It returns exactly three answers —
`PASS`, `KILL:<reason>`, `UNDERPOWERED` — and ships with 19 deterministic self-tests, including
ground-truth controls in both directions: a known-null feature must be killed, a known-real one must
survive.

**Multiple-testing correction runs at programme scope, not probe scope.** Deflating a Sharpe ratio by
the trials inside one probe is not enough — whatever you eventually deploy is the maximum over *every*
coordinate the programme has ever searched. Per the False Strategy Theorem the expected maximum Sharpe
across 1,000 null trials is **3.26**, with a true Sharpe of zero. So the deflation is applied against
the whole search history, which grows every time a verdict is reached quickly.

**Verdicts are typed, and never merged.** `FALSIFIED` (the coordinate is dead) is a different event
from `KILL` (a pre-committed line tripped), from `GATE-FAIL` (never entered testing), from
`cost-trapped` (statistically real, uneconomic), from `UNDERPOWERED` (*cannot conclude* — explicitly
not a kill, and not permitted to be reported as one). Collapsing these would be the cheapest way to
launder a weak result into a strong-sounding one.

**The gates are themselves audited for over-strictness.** A protocol tuned to avoid false positives
can quietly kill edges that are real but marginal — and the resulting pile of negative results then
*looks* like a genuine information ceiling when it is an artifact of the process. So the check runs in
the other direction: take a **documented public anomaly**, answer known and fixed in advance, and put
it through the entire gauntlet. The gates must let it through.

Momentum was used. The result was **mixed, and is recorded as mixed**: the gauntlet detected it
(p = 0.001, bootstrap CI excluding zero, out-of-sample intact, adequately powered) — but the
programme-scope Sharpe deflation *rejected* it. One of the best-documented anomalies in finance,
failed by a single over-eager gate. The conclusion was written back into the protocol rather than
explained away: deflation is one layer of evidence, never a sole guillotine.

**There is a failure log, and it is not empty.** It records occasions when the process itself was
violated — a result sentence drafted from expectation before the script had printed, caught on review
against the verbatim output, corrected. Those entries stay in the file, with the lesson attached. The
pull to pre-narrate is strongest exactly when the result would be exciting, which is the moment the
discipline is for.

## Honest results

Across the cross-anchor study the outcome is **a rigorous negative**: a measurable gross signal exists,
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
| **info** | lever | OHLC | OHLCV **+ 1D order flow** |
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
established as exhausted, the follow-on work moved **two levers** instead — same market, same data
store. First `info`: from price and order flow to *positioning* (what the crowd is holding, rather
than the wake it leaves in the tape), which is the regime Minority Game theory actually describes.
Then `construction`: from an unconditional directional bet to fading the crowd **only at its
extremes** — because positioning content is necessary but not sufficient. The crowd also has to be
the *fadeable* one; fed a well-informed crowd, a minority-game model faithfully does the wrong
thing. That study is open, not concluded.

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
