# Tidal Arc

`Tidal Arc` is a four-skill OKX perpetual trading framework built for agent-driven market selection, directional impulse trading, and higher-timeframe trend rebalancing.

The name reflects the structure of the system:

- `Tidal` represents flow, participation, expansion, and market pressure
- `Arc` represents transition, handoff, and the curved path from screening to impulse action to trend maintenance

This framework is designed for OpenClaw, but its structure is intentionally strategy-first rather than tool-first.

## Core Idea

Tidal Arc is not a single "do everything" skill.

It is a layered workflow:

1. scan the market for tradable or abnormal contracts
2. hand a selected symbol to a directional impulse manager
3. use a separate rebalance layer when the market context calls for slower trend maintenance

This keeps the system interpretable and avoids mixing:

- market screening
- intraday trigger logic
- higher-timeframe position management

## Included Skills

### 1. `impulse-market-scanner`

Role:
- find intraday-ready OKX swaps
- detect healthy liquidity, usable volatility, and active participation
- optionally produce regime-shift and outlier watchlists

Key modes:
- `balanced`
- `aggressive`
- `outlier`

Best use:
- before calling either impulse manager

### 2. `long-impulse-manager`

Role:
- aggressive long-only intraday decision engine
- supports first open, add, hold, reduce, de-risk, and flatten

Time structure:
- `1H` bias
- `15m` decision
- `5m` trigger

### 3. `short-impulse-manager`

Role:
- aggressive short-only intraday decision engine
- supports first open, add, hold, reduce, de-risk, and flatten

Time structure:
- `1H` bias
- `15m` decision
- `5m` trigger

### 4. `trend-rebalance-manager`

Role:
- slower higher-timeframe trend and position maintenance layer
- periodic add / reduce / hold / flatten management

Time structure:
- `1D` context
- `4H` regime
- `1H` / `4H` refinement

## System Logic

Tidal Arc separates four problems:

### Market discovery

Handled by `impulse-market-scanner`.

This layer answers:
- what is liquid enough?
- what is volatile enough?
- what is active enough?
- what is abnormal enough to deserve attention?

### Long-side impulse execution logic

Handled by `long-impulse-manager`.

This layer answers:
- is long direction justified?
- is this a first open, add, hold, reduce, de-risk, or flatten?

### Short-side impulse execution logic

Handled by `short-impulse-manager`.

This layer answers the same questions for short exposure.

### Higher-timeframe maintenance

Handled by `trend-rebalance-manager`.

This layer exists so the whole system does not collapse into a pure intraday-only mindset.

## Risk Style

Tidal Arc is not conservative.

Its impulse components are designed for medium-to-high-risk intraday trading, especially when the user wants:

- active directional participation
- continuation trading
- add-on management
- event-driven or outlier hunting

The rebalance component is steadier, but still active rather than passive.

## Protection Standard

Across the system, one principle remains stable:

- first opens should carry explicit invalidation or stop-loss logic
- add-on actions should trigger a whole-position protection review

This is important because it keeps the system from becoming a pure signal engine with no protection discipline.

## Suggested Workflow

Use Tidal Arc in this order:

1. Run `impulse-market-scanner`
2. Select the best candidate or watchlist name
3. Route to `long-impulse-manager` or `short-impulse-manager`
4. If the trade evolves into slower position maintenance, hand off to `trend-rebalance-manager`

## Why This Name Fits

`Tidal Arc` fits because the system is built around:

- flow
- expansion
- participation
- regime transition
- directional follow-through

It is not purely a scalper stack and not purely a trend-following stack. It moves across both.

## Folder Structure

```text
tidal-arc-okx-trading-system/
├── README.md
├── README.en.md
├── README.zh-CN.md
├── impulse-market-scanner/
│   └── SKILL.md
├── long-impulse-manager/
│   └── SKILL.md
├── short-impulse-manager/
│   └── SKILL.md
└── trend-rebalance-manager/
    └── SKILL.md
```
