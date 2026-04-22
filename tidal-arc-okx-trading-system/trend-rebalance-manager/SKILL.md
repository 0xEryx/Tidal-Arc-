---
name: trend-rebalance-manager
description: Use this custom agent skill when a trader wants a medium-to-higher timeframe trend-following rebalance strategy on OKX perpetuals, especially when the workflow needs to determine the dominant trend first, then rebalance every 4 hours by adding, reducing, or holding based on market structure, technical indicators, funding, open interest, volume, liquidity quality, and risk controls. Trigger for requests like '做一个趋势追踪滚仓策略', '每4小时按趋势调整仓位', '先判断大级别趋势再决定加减仓', '做一个中高周期 trend following skill', '根据 funding、量能、OI 和流动性动态滚仓', or any request to design or operate a periodic trend-following position-management strategy for OKX swaps. Prefer this skill for trend systems and periodic rebalancing logic rather than short-term scalper management.
version: "0.1.0"
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "📈",
        "requires": { "bins": ["okx"] },
      },
  }
---

# Trend Rebalance Manager

This is an instruction-only strategy skill for a trend-following rollover workflow on OKX perpetuals.

Its purpose is to help an agent:

- determine the dominant higher-timeframe trend
- decide when to add, reduce, hold, or flatten on a 4-hour cadence
- keep dynamic size adjustments tied to objective market and risk inputs
- explain its reasoning in trader-friendly language

This skill is intentionally different from the short-term scalper skills. It is for slower position management with periodic re-evaluation, not fast intraday trigger management.

This version is intentionally **LLM-first and script-free**:

- do not depend on custom local strategy scripts
- do not assume a custom watcher implementation exists
- use existing OKX market / portfolio / trade capabilities plus the instructions in this file
- let the LLM gather the required data, reason over it, and produce a structured rebalance decision

## Use This Skill When

- the user wants a medium-term or higher-timeframe trend-following strategy
- the user wants a 4-hour rebalance cadence
- the strategy needs dynamic add/reduce sizing rather than one-shot entry only
- the user wants to combine:
  - higher-timeframe indicators
  - funding
  - open interest
  - depth / liquidity quality
  - volume expansion or contraction
  - account-level risk controls
- the user says things like:
  - "做一个趋势追踪滚仓策略"
  - "先判断中大级别趋势，再每4小时滚仓"
  - "4小时按趋势加减仓"
  - "根据 funding、量能、流动性和风控来动态调仓"

## Do Not Use This Skill When

- the user only wants a raw market-data answer
- the user only wants a one-off direct order
- the user wants short-term scalper playbooks with minute-level triggers
- the user wants a fully autonomous production-ready bot with unattended stateful automation

## Workflow Position

This skill is a strategy-layer skill. It should sit above raw market, portfolio, and execution skills.

Typical downstream dependencies:

- market data / indicators
- portfolio / max size / balance context
- execution / leverage / conditional protection

This skill does not require custom strategy scripts. It can be used manually, through chat, or through a generic scheduler that simply asks the agent to re-evaluate every 4 hours.

## Before Using This Skill

- Confirm instrument universe and whether the strategy is for one symbol or multiple swaps.
- Confirm profile and account scope (`demo` or `live`).
- Confirm the dominant timeframe stack. A reasonable default is:
  - context trend: `1D`
  - regime confirmation: `4H`
  - execution refinement: `1H` or `4H`
- Confirm whether the strategy is:
  - long-only
  - short-only
  - bi-directional
- Confirm whether add/reduce actions should be:
  - recommendation-only
  - approval-required
  - directly executable once the reasoning is complete

## After This Skill

- Route to execution only after the rebalance decision is clearly stated as a concrete action.
- Route to portfolio checks before any live rebalance if the size logic depends on balance, max size, leverage, or current exposure.
- If the user wants repeated 4-hour operation, reuse this same skill each cycle instead of requiring a custom script layer.

## Core Operating Principle

For this version, the LLM itself should perform the full reasoning loop:

1. gather the required OKX-native data
2. classify the higher-timeframe trend
3. score confirmation, crowding, liquidity, and risk
4. decide `add`, `reduce`, `hold`, `de-risk`, or `flatten`
5. size the change dynamically
6. optionally hand off to execution

Do not claim that a separate custom strategy engine exists if it does not.

## Strategy Design Goals

This skill should guide the agent toward a strategy with five layers:

1. Trend detection
2. Rebalance cadence
3. Dynamic sizing
4. Protection and risk
5. Observability and logs

### 1. Trend Detection

The strategy should determine the dominant trend from medium-to-higher timeframes before considering any add or reduce action.

Recommended first-pass inputs:

- HTF price vs trend EMA
- ADX or trend-strength proxy
- MACD direction or slope
- higher-timeframe market structure
- volume regime

Example directional logic:

- only add in the direction of the confirmed higher-timeframe trend
- reduce or hold when trend remains intact but participation weakens
- reduce aggressively or exit when regime deterioration is confirmed

## Recommended Indicator Stack

Use a restrained indicator stack. Do not try to use every possible signal at once.

### Core Trend Stack

Use these as the default higher-timeframe trend inputs:

- `1D EMA 50 / EMA 200`
  - primary regime filter
  - bullish if price is above both and EMA 50 is above EMA 200
  - bearish if price is below both and EMA 50 is below EMA 200
- `4H EMA 21 / EMA 55`
  - medium-term trend continuation filter
  - strong trend when price holds above the fast/slow pair in the trend direction
- `4H MACD`
  - momentum confirmation
  - use direction and histogram slope rather than crossover alone
- `4H ADX(14)` or trend-strength proxy
  - use to separate trend regime from chop
  - treat low-ADX environments as weaker rollover environments

### Participation / Confirmation Stack

Use these to decide whether trend exposure should be expanded or reduced:

- `4H volume vs 20-bar average volume`
  - continuation is stronger when breakout / continuation candles are accompanied by above-average volume
- `1D and 4H open interest change`
  - supportive when OI expands with price in the trend direction
  - caution when OI expands but price stalls or weakens
- `current funding rate + recent funding context`
  - mild positive funding in an uptrend can be acceptable
  - extreme funding should reduce add size or block adds

### Risk / Exhaustion Stack

Use these to avoid late crowded adds:

- spread and order book depth
- distance from `4H ATR(14)` equilibrium
- funding extreme
- OI spike without follow-through
- strong move on weak volume

### Suggested Default Timeframe Stack

Unless the user specifies otherwise, use:

- context trend: `1D`
- rebalance decision layer: `4H`
- optional refinement layer: `1H`

### Suggested Default Interpretation

- `Trend regime`
  - mainly from `1D EMA 50/200` and `4H EMA 21/55`
- `Momentum quality`
  - mainly from `4H MACD` and `ADX`
- `Participation quality`
  - mainly from volume, OI, and funding
- `Execution safety`
  - mainly from spread, depth, and current risk state

### 2. Rebalance Cadence

The base cadence is every 4 hours.

At each rebalance window, the agent should evaluate:

- trend still valid?
- trend strengthening or weakening?
- funding becoming crowded?
- OI supportive or unstable?
- volume confirming continuation?
- liquidity acceptable for the rebalance size?
- current account risk budget still available?

The default rebalance outputs should be:

- `add`
- `reduce`
- `hold`
- `de-risk`
- `flatten`

If the user is interacting live rather than through automation, the skill should still behave as if it were at a rebalance checkpoint and produce a current-cycle recommendation.

## 4H Rebalance Scoring Framework

Use a more granular scorecard rather than vague prose. The goal is still consistency, but the score should better separate:

- strong trend vs usable trend
- trend strength vs add quality
- healthy continuation vs crowded extension

### Phase 1 Score Buckets

Build **eight** sub-scores on a `0-5` scale.

#### 1. `trend_regime_score`
How clearly the higher-timeframe trend is aligned.

- `0` = opposing regime
- `1` = mostly neutral
- `2` = weak alignment
- `3` = aligned but imperfect
- `4` = clear alignment
- `5` = strong multi-timeframe alignment

#### 2. `momentum_score`
How strong and stable the medium-term continuation looks.

Use mainly:

- `4H MACD` direction and histogram slope
- `4H ADX` or trend-strength proxy
- continuation structure on `4H`

Interpretation:

- `0` = fading / conflicting
- `1` = weak
- `2` = mixed
- `3` = acceptable
- `4` = strong
- `5` = strengthening cleanly

#### 3. `participation_score`
How well volume participation confirms the move.

Use mainly:

- `4H volume vs 20-bar average`
- recent volume expansion / contraction
- whether price continuation is happening on supportive or weak participation

Interpretation:

- `0` = no confirmation
- `1` = poor
- `2` = weak/mixed
- `3` = acceptable
- `4` = supportive
- `5` = strong confirmation

#### 4. `oi_quality_score`
Whether open interest is supporting the trend or increasing fragility.

Use mainly:

- `1D and 4H open interest change`
- price change relative to OI change

Interpretation:

- `0` = dangerous OI behavior
- `1` = clearly unstable
- `2` = questionable
- `3` = neutral / mixed
- `4` = supportive
- `5` = strongly supportive

Quick reading rules:

- price rising with OI rising can support bullish continuation
- price falling with OI rising can support bearish continuation
- OI rising while price stalls or structure weakens should lower this score
- OI falling during extension should reduce confidence in fresh adds

#### 5. `crowding_risk_score`
Whether funding and positioning still look healthy enough for continuation.

Use mainly:

- current funding rate
- recent funding context, not just the latest print
- whether funding is becoming extreme in the current trend direction

Interpretation:

- `0` = extreme crowding / adverse funding
- `1` = heavily crowded
- `2` = elevated risk
- `3` = manageable
- `4` = healthy
- `5` = very healthy

#### 6. `volatility_state_score`
Whether current volatility supports fresh rebalancing or warns of poor add timing.

Use mainly:

- `4H ATR(14)`
- ATR relative to recent history
- distance from `4H` trend mean / equilibrium

Interpretation:

- `0` = volatility state strongly hostile to adds
- `1` = very poor
- `2` = stretched / risky
- `3` = acceptable
- `4` = healthy
- `5` = ideal continuation volatility

Important: do **not** treat higher ATR as automatically bullish or bearish. This score should reward usable volatility, not simply bigger movement.

#### 7. `position_quality_score`
Whether the current location in the move is favorable for adding, holding, or reducing.

Use mainly:

- distance from `4H EMA 21 / 55`
- extension vs recent swing structure
- whether the move looks early, mid-trend, or late / overextended

Interpretation:

- `0` = very poor add location
- `1` = poor
- `2` = stretched
- `3` = acceptable
- `4` = favorable
- `5` = very favorable

This score exists because a trend can be real while the current entry location is still poor.

#### 8. `liquidity_execution_score`
Whether current spread/depth quality is good enough for the intended rebalance.

Use mainly:

- spread
- top-of-book depth
- intended rebalance size relative to visible depth
- observed execution cleanliness

Interpretation:

- `0` = unsafe
- `1` = poor
- `2` = weak
- `3` = acceptable
- `4` = good
- `5` = strong

### Phase 1 Total Score

Calculate:

- `total_score = trend_regime_score + momentum_score + participation_score + oi_quality_score + crowding_risk_score + volatility_state_score + position_quality_score + liquidity_execution_score`

Maximum total score: `40`

### Suggested Action Mapping

Use the score only after checking hard overrides.

- `0-10`
  - `flatten` or `de-risk`
- `11-17`
  - `reduce` or defensive `hold`
- `18-24`
  - `hold`
- `25-31`
  - `add` small or medium
- `32-40`
  - `add` medium or full rebalance unit, subject to risk caps

Do not treat this mapping as mechanical if crowding, execution quality, or margin state is poor.

### Hard Overrides

Regardless of total score, prefer `reduce`, `de-risk`, `flatten`, or blocked adds if any of these are true:

- funding is beyond the user-defined crowding threshold
- depth is too thin for the intended rebalance size
- spread is abnormally wide
- current position already breaches the symbol risk cap
- OI is spiking while price structure weakens
- ATR / extension state implies obvious late-move chasing
- account margin state is unsafe

### Suggested Dynamic Size Mapping

Phase 2 changes the interpretation layer. Do not rely on `total_score` alone.

After the Phase 1 score is computed, also derive these split outputs:

- `direction_score`
  - how strong the directional bias is
- `addability_score`
  - how suitable the current market state is for adding
- `risk_pressure_score`
  - how strongly the system should prefer reducing risk
- `confidence_score`
  - how internally consistent the signals are

#### Split Output Definitions

##### `direction_score` (`0-100`)
Use primarily:

- `trend_regime_score`
- `momentum_score`
- directional interpretation of `oi_quality_score`

Interpretation:

- `0-24` = no usable directional edge
- `25-49` = weak bias
- `50-69` = moderate bias
- `70-84` = strong bias
- `85-100` = very strong bias

##### `addability_score` (`0-100`)
Use primarily:

- `participation_score`
- `oi_quality_score`
- `crowding_risk_score`
- `volatility_state_score`
- `position_quality_score`
- `liquidity_execution_score`

Interpretation:

- `0-24` = do not add
- `25-49` = adds should usually be blocked or tiny
- `50-69` = adds acceptable but restrained
- `70-84` = adds are favorable
- `85-100` = strong add conditions

##### `risk_pressure_score` (`0-100`)
Use primarily:

- crowding deterioration
- ATR/extension stress
- poor position quality
- weakening participation
- weakening momentum
- margin or exposure stress

Interpretation:

- `0-24` = low pressure
- `25-49` = manageable pressure
- `50-69` = moderate pressure, prefer caution
- `70-84` = high pressure, prefer de-risk
- `85-100` = extreme pressure, prefer flatten or hard de-risk

##### `confidence_score` (`0-100`)
This is a consistency score, not a direction score.

Use it to answer:

- are the major signals aligned?
- or are they conflicting in a way that should reduce aggression?

Raise confidence when these align:

- HTF regime
- 4H momentum
- participation
- OI quality
- crowding state

Lower confidence when these conflict, for example:

- trend regime strong but participation weak
- trend regime strong but volatility/extension hostile
- OI rising while price stalls
- funding increasingly crowded while continuation quality weakens

Interpretation:

- `0-24` = highly conflicted
- `25-49` = conflicted
- `50-69` = usable but mixed
- `70-84` = aligned
- `85-100` = highly aligned

#### Confidence Modifier

Apply the confidence score as a behavioral modifier:

- `0-24`
  - block fresh adds unless the user explicitly overrides
- `25-49`
  - downgrade one action tier and one size tier
- `50-69`
  - allow normal interpretation with caution wording
- `70-84`
  - no penalty
- `85-100`
  - no penalty, strongest quality of confirmation

#### Final Decision Logic

Use this order:

1. hard overrides
2. direction score
3. addability score
4. risk pressure score
5. confidence modifier
6. final action and size

Preferred decision mapping:

- strong direction + high addability + low risk pressure
  - `add`
- strong direction + weak addability + medium risk pressure
  - `hold`
- strong direction + weak addability + high risk pressure
  - `de-risk`
- weak direction + high risk pressure
  - `reduce` or `flatten`
- moderate direction + moderate addability + low confidence
  - `hold` over aggressive add

#### Size Intensity Mapping

Once an action still qualifies as `add` after the split-score checks:

- `addability 50-59`
  - `0.25x` rebalance unit
- `addability 60-74`
  - `0.50x` rebalance unit
- `addability 75-89`
  - `0.75x` rebalance unit
- `addability 90-100`
  - `1.00x` rebalance unit

Then apply penalties:

- confidence below `50`: reduce one size tier
- risk pressure above `70`: reduce one or two size tiers
- stretched ATR / poor position quality: reduce one size tier
- high existing exposure: reduce one size tier
- conflicting 1H refinement signal: reduce one size tier or convert to `hold`
- if the raw add size exceeds the capital cap, clip it to the cap and state that the cap overrode the raw size
### 3. Dynamic Sizing

Size adjustments should not be fixed.

The initial design should support dynamic sizing based on:

- indicator conviction score
- volume confirmation
- funding crowding penalty
- OI expansion / contraction
- liquidity quality multiplier
- account risk budget
- max order size / max available size

A practical first-pass sizing model:

- base rebalance unit
- conviction multiplier
- liquidity multiplier
- risk cap

Example:

- strong trend + healthy volume + neutral funding + good depth = add full rebalance unit
- strong trend + overheated funding + weak liquidity = add reduced size or hold
- weakening trend + deteriorating volume + adverse funding = reduce

The LLM should explicitly state which factors increased size and which factors reduced size.

### Hard Capital Constraint

For this skill, every single add action must obey this hard rule:

- each add may use at most `3%` of total available capital

The assistant should interpret this as:

- first inspect available capital / available margin / usable USDT from current account context
- compute `single_add_cap = available_capital × 0.03`
- do not recommend or execute an add larger than that capital allocation cap
- if leverage is used, clearly distinguish:
  - capital committed
  - resulting notional exposure

Example:

- available capital = `10,000 USDT`
- max capital allocation for one add = `300 USDT`
- at `5x` leverage, the resulting notional may be `1,500 USDT`

If the scoring model suggests a larger add, the assistant must clip the add down to the 3% capital cap and explicitly say that the cap overrode the raw sizing suggestion.

### 4. Protection And Risk

This skill should treat risk controls as first-class logic, not afterthoughts.

Recommended controls:

- max account risk per symbol
- max aggregate risk across trend book
- max leverage by symbol quality
- no add if spread / depth quality deteriorates beyond threshold
- no add if funding exceeds threshold
- reduce if crowding grows without price follow-through
- reduce if OI spikes against weakening structure
- kill-switch if account drawdown or margin state breaches threshold

### Total Position Protection Reset

Every time the strategy changes the position size, whether by:

- `add`
- `reduce`
- `de-risk`
- `partial flatten`

the assistant must re-evaluate protection for the **entire resulting position**, not only for the incremental order.

That means every rebalance decision should also restate or update:

- the total-position stop-loss logic
- the total-position take-profit logic
- whether protection should tighten, stay unchanged, or transition toward break-even / profit lock

Do not assume the previous stop-loss / take-profit plan remains valid after the position size changes.

### Default Protection Adjustment Policy

Unless the user explicitly specifies another method, use this default policy:

- after an `add`
  - recalculate protection using current 4H structure and the combined total-position logic
  - do not widen the stop blindly just because the size increased
- after a `reduce`
  - reassess whether the remaining position now justifies a tighter stop
  - consider switching to stronger capital-preservation logic
- after a `de-risk`
  - prioritize preservation over re-expansion

### Suggested TP/SL Anchors

Preferred stop-loss / take-profit anchors, in order:

1. higher-timeframe structure invalidation
2. 4H ATR-based volatility buffer
3. recent swing high / swing low
4. break-even / profit-lock transition after sufficient extension

Avoid arbitrary fixed-percentage TP/SL unless the user explicitly wants that style.

### 5. Observability And Logs

Every 4-hour cycle should produce:

- market snapshot
- trend score
- sizing decision
- risk overrides
- resulting action
- order result or no-op reason

This strategy should be designed so that every rebalance decision is explainable after the fact.

Even without custom scripts, the assistant should always answer with a structured cycle report.

## Data Availability Policy

Use OKX-native data first whenever possible.

### Native From OKX / OKX Trade Kit

Available directly or through current OKX skills / CLI:

- candles / OHLCV
- order book / depth
- funding rate
- open interest
- mark price
- recent trades
- account balance
- positions
- max size / max available size
- leverage controls
- order / algo execution

### Partially Native / Requires Derived Logic

These are not single ready-made OKX outputs and should be derived by the LLM from the fetched OKX-native inputs:

- trend score
- conviction score
- volume regime
- liquidity quality score
- crowding score
- rebalance size recommendation

### External Or Non-Native Inputs

`liquidity map` or liquidation heatmap style signals are not currently exposed as a first-class native OKX market command in the installed OKX market skill set.

For this strategy, treat liquidity-map inputs as one of:

- external provider data
- future adapter module
- locally approximated proxy from:
  - order book depth
  - recent trade aggressiveness
  - OI change
  - funding
  - volatility and volume expansion

Do not claim that a true liquidation map is available from the current OKX-native skill surface unless an external data source is actually wired in.

If the user requires liquidity-map-aware logic but no external liquidity-map source is available, the assistant should say so clearly and fall back to a proxy liquidity score rather than pretending certainty.

## Data Gathering Workflow

When this skill is used, the LLM should gather data in this order:

1. `okx-cex-market`
   - fetch higher-timeframe candles for trend context
   - fetch 4H candles for rebalance context
   - fetch funding rate
   - fetch open interest
   - fetch order book / depth
   - fetch recent trades or volume context if needed
2. `okx-cex-portfolio`
   - fetch current positions
   - fetch balance / available margin
   - fetch max-size or max-avail-size if sizing is needed
3. derive internal scores
   - trend regime score
   - momentum score
   - participation score
   - OI quality score
   - funding crowding score
   - volatility state score
   - position quality score
   - liquidity execution score
   - risk-adjusted size score
4. if the user wants execution
   - hand off to `okx-cex-trade`

## Recommended LLM Reasoning Structure

For each cycle, reason in this order:

1. Trend regime
   - uptrend / downtrend / neutral
2. Trend quality
   - strong / moderate / weak
3. Participation quality
   - strong / average / weak
4. OI quality
   - supportive / mixed / unstable
5. Crowding assessment
   - healthy / crowded / extreme
6. Volatility and location quality
   - usable / stretched / hostile
7. Liquidity assessment
   - good / acceptable / poor
8. Split decision scores
   - direction score
   - addability score
   - risk pressure score
   - confidence score
9. Rebalance action
   - add / reduce / hold / de-risk / flatten
10. Dynamic size
   - small / medium / full rebalance unit
11. Total-position protection update
   - how stop-loss / take-profit should change after the rebalance

The assistant should not skip directly to execution without showing the reasoning summary first.
## Initial Directional Architecture

A practical v1 implementation path for this strategy:

1. `trend snapshot`
   - fetch candles, funding, OI, depth, trades, balance, positions
2. `feature builder`
   - compute trend, volume, crowding, liquidity, and risk features
3. `decision engine`
   - output `add`, `reduce`, `hold`, `flatten`
4. `size engine`
   - convert conviction and risk into notional delta
5. `executor`
   - place / amend / reduce through OKX CLI
6. `reporter`
   - log and summarize the cycle

In this version, these are conceptual stages carried by the LLM and existing OKX skills, not separate custom scripts.

## Suggested First Implementation Scope

To keep the first version realistic, start with:

- one symbol only
- one trend family only
  - e.g. EMA slope + MACD + ADX proxy
- one rebalance cadence only
  - `4H`
- one execution family only
  - market or conservative limit logic
- one protection style only
  - hard reduce thresholds + max leverage cap

Then expand into:

- multi-symbol ranking
- external liquidity-map adapter
- portfolio-level allocator
- richer trend regimes
- laddered trend adds / staged de-risking

## Execution Policy

This skill should default to:

- analyze first
- recommend second
- execute only after the user explicitly asks for execution, unless the conversation already established an automatic mode and the profile is clear

Before any live rebalance execution:

- restate the profile
- restate the symbol
- restate the action
- restate the dynamic size rationale
- restate whether the 3% capital cap constrained the size
- restate the updated total-position stop-loss / take-profit logic

## Ambiguity And Unsupported Requests

- If the user does not specify the instrument universe or trend horizon, ask a concise clarification question.
- If the user requests a full production autonomous service, explain that this version is instruction-only and LLM-driven rather than backed by a dedicated custom service.
- If the user requests true liquidation-map-driven logic, clarify whether an external data source is available.
- Do not pretend that non-native market structure or liquidity-map data already exists if the current system cannot fetch it.
- If the user asks for a repeated 4-hour cycle, it is acceptable to describe how this skill should be invoked every 4 hours, but do not claim that unattended automation already exists unless it actually does.

## Output Expectations

When used for design or evaluation, this skill should aim to return:

- a higher-timeframe trend summary
- a regime-strength summary
- a 4-hour rebalance recommendation
- a dynamic size rationale
- a clear statement of whether the 3% add cap constrained size
- risk warnings
- an execution recommendation or hold/no-op recommendation
- an updated total-position protection plan

Preferred top-level statuses:

- `supported_direction`
- `needs_clarification`
- `unsupported_in_current_stack`

## Preferred User-Facing Output Format

Each cycle should ideally report:

- symbol and profile
- trend regime
- key inputs:
  - trend indicators
  - funding
  - OI
  - volume
  - liquidity quality
  - current risk state
- action:
  - `add`
  - `reduce`
  - `hold`
  - `de-risk`
  - `flatten`
- dynamic size explanation
- capital cap check
- updated total-position stop-loss / take-profit logic
- whether execution is being recommended or actually performed

## Standard Output Template

When possible, return a structured trader-facing report in this shape:

```text
📈 Trend Rollover Manager - <INST_ID> [<PROFILE>]
━━━━━━━━━━━━━━━━━━━━
Trend regime: <uptrend/downtrend/neutral>
Rebalance cadence: 4H
Current position: <long/short/flat + size summary>
━━━━━━━━━━━━━━━━━━━━
Trend:
- 1D EMA regime: <bullish/bearish/neutral>
- 4H EMA continuation: <strong/mixed/weak>
- 4H MACD: <supportive/conflicting>
- 4H ADX / trend strength: <strong/moderate/weak>

Participation:
- Volume regime: <strong/average/weak>
- OI context: <supportive/mixed/risky>
- Funding context: <healthy/crowded/adverse>
- Liquidity quality: <good/acceptable/poor>

Scores:
- Trend regime score: X/5
- Momentum score: X/5
- Participation score: X/5
- OI quality score: X/5
- Crowding risk score: X/5
- Volatility state score: X/5
- Position quality score: X/5
- Liquidity execution score: X/5
- Total score: X/40

Decision:
- Direction score: X/100
- Addability score: X/100
- Risk pressure score: X/100
- Confidence score: X/100
- Action: <add/reduce/hold/de-risk/flatten>
- Size intensity: <0.25x / 0.50x / 0.75x / 1.00x rebalance unit>
- Reason: <short explanation>
- Capital cap: <within 3% cap / clipped to 3% cap>

Protection update:
- Stop-loss logic: <updated logic for the whole position>
- Take-profit logic: <updated logic for the whole position>
- Why changed: <short explanation>

Risk notes:
- <risk warning 1>
- <risk warning 2>

Execution:
- <recommendation only / ready for execution / execution requested>
```

If the user wants a more machine-readable answer, also include:

```json
{
  "status": "supported_direction",
  "instId": "BTC-USDT-SWAP",
  "profile": "demo",
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
  "decision_scores": {
    "direction_score": 82,
    "addability_score": 58,
    "risk_pressure_score": 64,
    "confidence_score": 63
  },
  "decision": {
    "action": "hold",
    "size_intensity": 0.25,
    "unit": "rebalance_unit",
    "capital_cap_pct": 3,
    "capital_cap_applied": false
  },
  "protection_update": {
    "stop_loss_logic": "4H structure invalidation below recent swing low",
    "take_profit_logic": "trail winners while trend remains intact; partial de-risk into extension",
    "updated_for_total_position": true
  },
  "risk_notes": [
    "Funding is elevated but still within tolerance."
  ]
}
```

## User Request Examples

- "帮我设计一个趋势追踪滚仓策略 skill"
- "先判断大级别趋势，再每4小时动态加减仓"
- "做一个 funding + OI + volume + liquidity 的趋势滚仓系统"
- "给我一个中高周期 trend following perpetual strategy skill"
- "先看 BTC-USDT-SWAP 的中大级别趋势，如果趋势继续走强，就给我一个 4 小时滚仓建议"
- "用 demo 账户评估一下 ETH-USDT-SWAP，现在如果按趋势追踪思路，下一次 4H rebalance 应该加仓、减仓还是继续拿着"
- "我想做一个只做多的趋势滚仓逻辑，你先根据 EMA、MACD、量能、funding 和 OI 判断当前是不是值得继续加"
- "按中周期趋势跟随来判断 SOL-USDT-SWAP，这个 4 小时节点我该不该滚仓，加多少更合理"
- "先不要执行，只给我一份 4H 趋势滚仓评分报告，告诉我是 add、reduce 还是 hold"
- "如果按趋势追踪滚仓，现在这笔 BTC 永续该不该减仓？请把 funding、流动性和风险一起算进去"
- "我想做一个每 4 小时检查一次的趋势跟随仓位管理，你先用当前市场状态给我一版 decision"
- "针对 XRP-USDT-SWAP，先给我 trend regime、总分、行动建议和动态仓位理由"
- "每次加仓最多只能用总可用资金的 3%，并且每次调仓后都要重算整体仓位的止盈止损"
- "你在给我 4H rebalance 建议时，记得把 3% 单次加仓上限和总仓位保护位更新一起算进去"

## References

- Design notes: `references/trend-rollover-design-notes.md`
