# Trend Rollover Manager Design Notes

This note captures the current knowledge base for building a trend-following rollover strategy skill on top of the existing OKX stack.

## 1. What The Current OKX Skill Surface Already Gives Us

Based on the installed OKX skills and official OKX docs, the current stack already exposes or supports:

- order book / market depth
- candles / OHLCV
- funding rate
- open interest
- mark price
- recent trades
- technical indicator queries
- account balance
- positions
- max order size / max available size
- leverage changes
- swap order placement
- conditional stop / algo management

Useful local references:

- `../../../../.openclaw/workspace/.agents/skills/okx-cex-market/SKILL.md`
- `../../../../.openclaw/workspace/.agents/skills/okx-cex-trade/SKILL.md`
- `../../../../.openclaw/workspace/.agents/skills/okx-cex-portfolio/SKILL.md`

## 2. What We Can Reliably Build From OKX-Native Inputs

The following are realistic v1 features using only current OKX-native inputs:

- higher-timeframe trend filters from candles
- rebalance every 4 hours
- volume confirmation from candles / trades
- funding crowding filters
- OI confirmation or warning signals
- liquidity quality proxy from spread + depth
- account-risk-aware sizing from balance / positions / max-size

These are enough for a solid first version of a trend-following rollover strategy.

## 3. What Is Not Native Yet

The requested `liquidity map` / liquidation heatmap style input is the biggest gap.

Current conclusion:

- we do have depth, funding, trades, OI, and price
- we do not currently have a first-class installed OKX-native liquidation-map skill or command in the local OKX skill surface

So for now, a liquidity-map-dependent design should explicitly choose one of three paths:

1. external provider integration
2. internal proxy score
3. feature deferred until adapter exists

## 4. Recommended Decision Model

A good v1 decision model is not "predict price", but "rebalance in the direction of a validated trend."

Phase 1 upgrade: move from a coarse `0-2` five-bucket score into a more granular `0-5` eight-dimension score.

Recommended decision layers:

1. Regime layer
   - trend up / trend down / neutral
2. Strength layer
   - trend strengthening / weakening
3. Participation layer
   - volume participation confirming or fading
4. OI quality layer
   - OI supporting trend or increasing fragility
5. Crowding layer
   - funding healthy / stretched / extreme
6. Volatility layer
   - ATR and extension usable vs hostile
7. Position quality layer
   - early/mid/late trend location
8. Liquidity layer
   - safe to adjust size or not
9. Risk layer
   - allowed size delta after caps

## 5. Suggested 4H Rebalance Contract

Each cycle should produce a structured record like:

```json
{
  "cycle_ts": 0,
  "instId": "BTC-USDT-SWAP",
  "trend_regime": "uptrend",
  "scores": {
    "trend_regime_score": 4,
    "momentum_score": 4,
    "participation_score": 3,
    "oi_quality_score": 3,
    "crowding_risk_score": 3,
    "volatility_state_score": 2,
    "position_quality_score": 2,
    "liquidity_execution_score": 4,
    "total_score": 25
  },
  "risk_cap_ok": true,
  "decision": "add",
  "target_delta_notional": 500,
  "reasoning": [
    "higher timeframe trend remains intact",
    "volume confirms continuation",
    "funding is manageable rather than ideal",
    "ATR and extension state argue for smaller sizing",
    "depth is sufficient for one rebalance unit"
  ]
}
```

Phase 1 interpretation notes:

- `trend_regime_score` and `momentum_score` answer whether the trend is real
- `participation_score` and `oi_quality_score` answer whether the move has support
- `crowding_risk_score` answers whether leverage is getting too crowded
- `volatility_state_score` answers whether the move is too extended or chaotic to add cleanly
- `position_quality_score` answers whether current location is favorable for add/hold/reduce
- `liquidity_execution_score` answers whether the size can be executed safely

## 6. Suggested Future Skill Split

If this strategy grows beyond one file, a clean split would be:

- `trend-rollover-manager`
  - top-level strategy orchestrator
- `trend-rollover-features`
  - compute trend, volume, OI, funding, liquidity scores
- `trend-rollover-executor`
  - translate rebalance delta into OKX orders
- `trend-rollover-scheduler`
  - run every 4 hours and emit cycle reports

## 7. Official Source Pointers

Official OKX docs and references used for this direction:

- OKX V5 API docs (public market + trade/account): [OKX API docs](https://www.okx.com/docs-v5/en)
- OKX Agent Trade Kit / agent docs: [OKX agent docs](https://www.okx.com/docs-v5/agent_en/)

Examples of relevant official capability areas:

- order book: `/api/v5/market/books`
- candlesticks: `/api/v5/market/candles`
- funding rate: `/api/v5/public/funding-rate`
- open interest: `/api/v5/public/open-interest`
- balance: account balance endpoints
- maximum order quantity: account max-size endpoints

## 8. Practical Recommendation

Build this skill in layered phases.

### Phase 1

- one symbol
- 4H cadence
- OKX-native inputs only
- no true external liquidity map
- dynamic add/reduce sizing
- detailed logs
- eight dimensions on a `0-5` scale
- total score on a `0-40` scale

### Phase 2

Add a second decision layer so the engine stops treating `total_score` as the only decision variable.

New split outputs:

- `direction_score`
- `addability_score`
- `risk_pressure_score`
- `confidence_score`

Recommended interpretation:

- `direction_score`
  - answers whether the directional trend edge is strong enough to care
- `addability_score`
  - answers whether this specific moment is actually suitable for adding
- `risk_pressure_score`
  - answers whether the system should prefer reducing risk instead
- `confidence_score`
  - answers whether the major inputs agree or conflict

Why this matters:

- a market can have a strong directional regime while still being a poor add location
- a trend can remain valid while risk pressure rises enough to justify hold or de-risk
- confidence should downgrade aggression when signals are internally conflicted

### Phase 3

- external liquidity-map adapter
- multi-symbol ranking
- portfolio-level allocator
- richer trend regimes and de-risk logic
- more adaptive thresholding by symbol and regime
