# Source — module taxonomy

This page documents the engine's module structure (~60,000 lines of R across ~45 files). **No source
code is included in this repository** — the engineering is visible from the structure alone.

## Layer modules (decision chain → consequence chain)

| Module | Layer | Role | Phase split |
|---|---|---|---|
| `05_mg_*` | MG signal | Latent agent ecology, order parameters, consensus carrier | A / B / C |
| `06_edge_*` | Edge | Threshold + side-gate validation, edge labels | A / B / C |
| `07_position_risk_*` | Risk | Sizing chain, min-hold, adoption cap, owner snapshot | A / B / C |
| `10_governance_*` | Governance | Sessions, daily caps, situational awareness, news blackout | A / B / C |
| `11_position_target_*` | Target | Per-frequency intent, owner-bridge | A / B / C |
| `12_execution_intent_*` | Intent | Global action priority, pending latch | A / B / C |
| `13_execution_realized_*` | Execution | t+1 fill, quantization, cost model — the only writer of position | A / B / C |
| `08_trade_accounting_*` | Accounting | WAC, realized P&L, trade events | A / B / C |
| `09_execpos_reporting_*` | Reporting | Derived reporting surfaces | A / B / C |

## Infrastructure

| Module | Role |
|---|---|
| `00_paths` / `00_system_bootstrap` | Path SOT and the entire static configuration tree |
| `02_market_data` | No-look-ahead bar feed + higher-timeframe aggregation |
| `03_system_orchestrator` | Engine entry, the bar loop, regime-mode branch |
| `14_postrun_export` | ~49 canonical CSV surfaces, written from committed state |
| `15_diagnostic` | ~100 derived attribution surfaces for full signal→fill tracing |

## What is intentionally not here

- Licensed market data
- Broker / market-access configuration
- The alpha-bearing signal internals

These remain private. The architecture (`../docs/architecture.md`) and validation methodology
(`../docs/methodology.md`) are the public showcase.
