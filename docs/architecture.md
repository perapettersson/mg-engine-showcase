# Architecture

The MG Engine is a bar-by-bar trading engine. Each bar is processed through a decision chain at
time `t` and a consequence chain at `t+1`. The architecture is built on three non-negotiable
principles: **single-writer state**, **strict t+1 execution timing**, and **deterministic replay**.

## The per-bar pipeline

```
┌──────────────────────────── DECISION CHAIN (bar t) ────────────────────────────┐
│                                                                                │
│  05 MG  →  06 EDGE  →  07 RISK  →  10 GOV  →  11 TGT  →  12 INT                 │
│  direction? clean?    how big?    allowed?   per-freq    one global             │
│                                              intent      action (latch)         │
└────────────────────────────────────────────────────────────────┬───────────────┘
                                                       pending latched for t+1
                                                                   │
┌──────────────────────────── CONSEQUENCE CHAIN (t+1) ─────────────▼───────────────┐
│                                                                                  │
│  13 EXEC  →  fill @ open(t+1)  →  08 ACCOUNTING  →  09 REPORTING                  │
│  the only layer that mutates realized position state                             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**One-word summary per layer**

| Layer | Question | Core tools |
|---|---|---|
| 05 MG | Is there an exploitable directional bias? | latent agents, H, σ², Δ(real−fake), score gap |
| 06 EDGE | Is the signal clean enough to trade? | threshold gates, side gates, edge labels |
| 07 RISK | How big, given validated edge? | sizing chain, min-hold, adoption cap |
| 10 GOV | Are we allowed to trade now? | sessions, daily caps, situational awareness, news blackout |
| 11 TGT | What does each frequency want? | per-freq enter/exit/hold/flip intent, owner-bridge |
| 12 INT | Which one action next bar? | global priority, pending latch |
| 13 EXEC | Execute the pending from t−1 | t+1 open fill, quantization, cost model |

## Core timing law (binding)

```
state(t)    = consequence(t-1)        # state already includes the previous fill
decision(t) → consequence(t+1)        # intent at t executes at t+1
```

No same-bar round-trips. Execution can never make a trading decision — it only realizes the
pending latched on the previous bar. This removes an entire class of look-ahead bugs by
construction.

## A / B / C phase split

Every layer is implemented as three functions with strict roles:

- **A — projection** (read-only compute; never mutates committed state)
- **B — selection** (decision logic; mutates only a local working copy)
- **C — commit** (the single writer; writes the layer's committed surface)

This makes data flow auditable: a field's value can always be traced to exactly one committing
module.

## Single-writer state

Every state surface has exactly one owning module. Downstream layers read committed surfaces;
they never reach around to mutate another layer's state. Examples:

- realized position is written **only** by the execution layer (13C)
- edge validity is written **only** by the edge committer (06C)
- governance flags are written **only** by the governance committer (10C)

A copy-the-whole-struct commit means any new field added in the selection phase automatically
flows downstream without a separate wiring step.

## Multi-frequency consensus

Independent MG agent populations run on **prime-number timeframes** (a 1-minute execution carrier
plus 7m / 13m / 19m voters). Primes are chosen so the higher timeframes share almost no bar
boundaries with each other or with trader-common grids (5/15/30/60) — applying the MG minority
principle on the *time axis* and producing more independent samples of the underlying process.

The collapse function at the signal commit either (a) **selects** the single best-quality frequency
or (b) requires **consensus** among the higher timeframes before taking a position. Execution always
resolves to the 1-minute carrier.

## Precedence ≠ pipeline order

Pipeline order (05 → 13) is not the same as decision authority. A later layer can outrank an
earlier one: governance force-flatten and overnight-flatten override risk sizing; a min-hold
lifecycle guard outranks a fresh signal. Authority is an explicit, documented ordering enforced by
assertions — not an accident of execution order.

## Determinism & replay safety

Same configuration + same data ⇒ identical output, bar for bar. The only randomness is seeded agent
initialization, routed through a deterministic seeded runner. This makes every result exactly
reproducible and every regression bisectable.
