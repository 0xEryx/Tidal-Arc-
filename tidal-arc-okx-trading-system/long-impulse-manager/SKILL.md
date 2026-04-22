---
name: long-impulse-manager
description: Use this custom OpenClaw skill when a trader wants an aggressive long-only impulse workflow on OKX perpetuals. This skill is for medium-fast intraday long bias decisions that combine a higher-timeframe trend filter with 15m decision logic and 5m trigger logic, then decide whether to open a fresh long, add to an existing long, hold, reduce, de-risk, or flatten. Trigger for requests like '做一个只做多的加仓减仓 skill', '先判断趋势再决定开多或追多', '15分钟结合5分钟做多头追踪', '看一下现在能不能开多', '如果已经有多单就判断要不要加仓', '给我一个 aggressive 的 long impulse manager', or any request to use multi-factor OKX market data plus position context to manage or initiate long exposure.
version: "0.1.0"
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "🚀",
        "requires": { "bins": ["okx"] },
      },
  }
---

# Long Impulse Manager

This is an instruction-only, long-only OpenClaw strategy skill for aggressive directional scaling on OKX perpetuals.

Its purpose is to help an agent:

- determine whether long bias is justified
- decide whether to `open_long`, `add_long`, `hold_long`, `reduce_long`, `de-risk_long`, or `flatten_long`
- use a short-to-medium intraday structure built around `1H -> 15m -> 5m`
- combine technical, positioning, liquidity, and risk inputs before acting
- explain the decision in trader-friendly language

This skill is intentionally more aggressive than a slow trend rebalance skill. It is designed for medium-fast directional trading, not passive swing holding and not minute-by-minute scalper playbooks.

This version is intentionally **LLM-first and script-free**:

- do not depend on custom local strategy scripts
- do not assume a custom watcher or autonomous bot service exists
- use existing OKX market / portfolio / trade capabilities plus the instructions in this file
- let the LLM gather the required data, reason over it, and produce a structured long-side action

## Use This Skill When

- the user wants a long-only directional manager
- the user wants to evaluate a fresh long entry or an add-on long
- the user wants a `15m` decision layer with a `5m` trigger layer
- the user wants a more aggressive, medium-to-high-risk style
- the user wants the assistant to combine:
  - higher-timeframe trend filter
  - 15m structure and continuation quality
  - 5m trigger quality
  - open interest
  - funding
  - volume participation
  - spread / depth / liquidity quality
  - account exposure and risk controls
- the user says things like:
  - "做一个 long impulse manager"
  - "只做多，判断开仓和追仓"
  - "15分钟结合5分钟给我做多建议"
  - "如果趋势偏多，现在能不能先开多"
  - "已有多单，帮我判断要不要继续加"

## Do Not Use This Skill When

- the user wants short-side logic
- the user only wants raw market data or a plain indicator answer
- the user only wants a one-off direct execution with no analysis
- the user wants a minute-level scalper playbook with dense trigger choreography
- the user wants a fully autonomous production bot with unattended stateful automation

## Workflow Position

This skill is a strategy-layer skill. It should sit above:

- market data / indicators
- portfolio / current exposure / max size
- execution / leverage / protection logic

Use it as the decision engine for long-side action. Hand off to execution only after the action is concrete.

## Before Using This Skill

- Confirm the target `instId`.
- Confirm profile and account scope: `demo` or `live`.
- Confirm whether execution mode is:
  - recommendation-only
  - approval-required
  - directly executable after reasoning
- Confirm whether there is already a live long position.
- Confirm whether the user wants:
  - fresh long entry allowed
  - add-only management for existing long
  - both fresh entry and add/reduce behavior

Reasonable defaults if the user does not specify:

- context bias: `1H`
- primary decision layer: `15m`
- trigger refinement: `5m`
- style: aggressive
- direction: long-only
- execution mode: approval-required

## After This Skill

- Route to `okx-cex-portfolio` before any live action if size depends on balance, max size, or current exposure.
- Route to `okx-cex-trade` only after the final action is clearly stated.
- Reuse this same skill each cycle if the user wants repeated evaluation.

## Core Operating Principle

For this version, the LLM itself should perform the full loop:

1. gather the required OKX-native data
2. classify the higher-timeframe bias
3. test 15m long continuation quality
4. test 5m trigger quality
5. score participation, crowding, liquidity, and risk
6. decide `open_long`, `add_long`, `hold_long`, `reduce_long`, `de-risk_long`, or `flatten_long`
7. size the change dynamically
8. if appropriate, hand off to execution

Do not claim that a separate custom strategy engine exists if it does not.

## Timeframe Architecture

Use this stack unless the user explicitly overrides it:

- `1H` = bias filter
- `15m` = main decision layer
- `5m` = trigger / timing layer

Interpretation:

- `1H` answers: should we even be thinking about longs?
- `15m` answers: does continuation look strong enough to act?
- `5m` answers: is the immediate location good enough to open or add now?

## Recommended Indicator And Market Stack

Use a restrained, trader-oriented stack rather than too many overlapping signals.

### 1H Bias Filter

- `1H EMA 20 / 50 / 200`
- optional `1H SuperTrend`
- optional `1H MACD` direction

Default long bias interpretation:

- stronger long bias when price is above `EMA 20` and `EMA 50`
- stronger still when `EMA 20 > EMA 50 > EMA 200`
- weaker if price is above short EMAs but below `EMA 200`
- no fresh long if `1H` regime is clearly bearish unless the user explicitly wants counter-trend behavior

### 15m Decision Stack

- `15m EMA 20 / 50`
- `15m VWAP`
- `15m MACD`
- `15m RSI`
- `15m ADX` or trend-strength proxy
- `15m volume vs 20-bar average`
- `15m open interest context`
- recent funding context

### 5m Trigger Stack

- `5m EMA 20`
- `5m RSI` or `5m Stoch RSI`
- `5m volume burst`
- local pullback / reclaim or breakout continuation behavior
- spread / top-of-book depth / recent trades

### Risk / Crowding Inputs

- funding rate
- OI expansion / contraction
- spread
- visible depth
- distance from short-term equilibrium
- ATR-based stretch
- current long exposure

## Actions

This skill should only output one of these actions:

- `open_long`
- `add_long`
- `hold_long`
- `reduce_long`
- `de-risk_long`
- `flatten_long`

## Default Decision Framework

Build **eight** sub-scores on a `0-5` scale.

### 1. `bias_regime_score`
How supportive the `1H` trend filter is for longs.

### 2. `continuation_score`
How strong the `15m` continuation structure looks.

Use mainly:

- `15m EMA 20 / 50`
- `15m MACD`
- `15m ADX`
- structure of higher lows / continuation

### 3. `trigger_quality_score`
How good the `5m` long trigger location is right now.

Use mainly:

- reclaim after pullback
- breakout with support from volume
- 5m momentum recovery
- whether current location is early / acceptable / overextended

### 4. `participation_score`
How well volume confirms the long continuation.

Use mainly:

- `15m volume vs 20-bar average`
- `5m` burst quality
- whether expansion is occurring with real participation

### 5. `oi_quality_score`
Whether OI is supporting the move or increasing fragility.

Quick reading rules:

- price up + OI up can support continuation
- price up + OI flat can still be usable but less convincing
- OI up while price stalls should reduce confidence
- sharp OI expansion without follow-through should lower add appetite

### 6. `crowding_risk_score`
Whether funding and positioning still look healthy enough for a long add.

Interpretation should punish:

- excessively hot positive funding
- overly crowded long conditions
- obvious late extension with euphoric positioning

### 7. `liquidity_execution_score`
Whether spread and depth are good enough for the intended long action.

### 8. `position_risk_score`
Whether the current account and position state still allow aggressive long behavior.

Use mainly:

- current exposure
- balance / available margin
- max size / max available size
- symbol concentration

## Total Score

Calculate:

- `total_score = bias_regime_score + continuation_score + trigger_quality_score + participation_score + oi_quality_score + crowding_risk_score + liquidity_execution_score + position_risk_score`

Maximum total score: `40`

### Suggested High-Level Mapping

- `0-10` -> `flatten_long` or `de-risk_long`
- `11-17` -> `reduce_long`
- `18-24` -> `hold_long`
- `25-31` -> `open_long` or `add_long` small / medium
- `32-40` -> `open_long` or `add_long` aggressively, subject to risk caps and overrides

## Split Decision Scores

After the 0-40 total score is built, also derive:

- `direction_score` (`0-100`)
- `entry_addability_score` (`0-100`)
- `risk_pressure_score` (`0-100`)
- `confidence_score` (`0-100`)

### `direction_score`
How strong the bullish directional edge is.

### `entry_addability_score`
How suitable the current state is for opening or adding right now.

### `risk_pressure_score`
How strongly the system should prefer reducing or avoiding additional risk.

### `confidence_score`
How aligned the major signals are.

Lower confidence when, for example:

- `1H` bias is bullish but `15m` continuation is weak
- `15m` looks good but `5m` trigger is poor
- OI expands while price stalls
- funding gets crowded while participation weakens

## Hard Overrides

Regardless of score, block fresh longs or reduce aggression if any of these are true:

- spread is abnormally wide
- visible depth is too thin for the intended action
- funding is beyond the allowed crowding threshold
- current long exposure is already too large
- price is obviously extended relative to short-term equilibrium
- account margin state is unsafe
- `1H` bias is no longer supportive for longs

## Sizing Style

This skill is intentionally medium-to-high risk. It is more aggressive than a conservative rebalance manager.

Use sizing bands, not fixed size:

- `tiny`
- `small`
- `medium`
- `aggressive`

Suggested mapping after all overrides:

- low addability -> `tiny`
- moderate addability -> `small`
- good addability -> `medium`
- exceptional alignment and healthy liquidity -> `aggressive`

Then apply penalties:

- low confidence -> reduce one size tier
- high risk pressure -> reduce one or two size tiers
- poor 5m trigger -> reduce one tier or convert to `hold_long`
- poor liquidity -> reduce one tier or block

## First-Open Stop-Loss Policy

This is a required rule for this skill.

When the action is `open_long`, the assistant must include a **default stop-loss plan** to guard against immediate directional error.

The default interpretation is:

- every first long entry should come with an initial invalidation stop
- the stop should be based on structure first, not arbitrary percentage first
- preferred anchors:
  1. recent `5m` / `15m` swing low invalidation
  2. volatility buffer using short-term ATR
  3. market structure failure below the reclaim / breakout support zone

Do not open a fresh long without clearly stating the default stop-loss logic unless the user explicitly overrides that behavior.

## Add-On Protection Policy

For later `add_long` actions:

- do not blindly require a brand-new separate stop for the incremental add
- do re-evaluate the protection logic for the **entire resulting long position**
- state whether the total-position stop should:
  - stay unchanged
  - tighten
  - move closer to break-even
  - transition into profit protection

This means:

- `open_long` -> default stop-loss is mandatory
- `add_long` -> total-position protection review is mandatory

## Profit-Taking Policy

This skill does not require a rigid default take-profit on every cycle.

Preferred behavior:

- for a fresh open, the assistant may suggest a staged profit-taking idea, but the mandatory protection element is the stop-loss
- for adds and ongoing management, the assistant should focus on:
  - whether to hold
  - whether to reduce
  - whether to de-risk
  - whether to trail protection

If the user explicitly wants TP levels, provide them. Otherwise, do not force artificial TP precision.

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
- order execution

### Derived By The LLM

These should be derived from fetched data, not invented:

- long bias quality
- continuation quality
- trigger quality
- crowding assessment
- liquidity quality score
- dynamic size recommendation

### Non-Native / Unsupported

Do not pretend a true liquidation heatmap or full proprietary order-flow analytics already exist if they are not actually wired in.

If such data is unavailable, fall back to proxy logic from:

- depth
- recent trades
- OI behavior
- funding
- volatility / participation

## Data Gathering Workflow

When this skill is used, gather data in this order:

1. `okx-cex-market`
   - `1H` candles for bias
   - `15m` candles for continuation
   - `5m` candles for trigger timing
   - funding
   - OI
   - order book / depth
   - recent trades if needed
   - relevant indicators
2. `okx-cex-portfolio`
   - current positions
   - balance / available margin
   - max-size or max-avail-size if sizing is needed
3. derive internal scores
4. if execution is requested
   - hand off to `okx-cex-trade`

## Recommended LLM Reasoning Structure

For each cycle, reason in this order:

1. `1H` bias
2. `15m` continuation quality
3. `5m` trigger quality
4. participation
5. OI quality
6. crowding / funding
7. liquidity
8. current exposure / risk
9. decision scores
10. action
11. size
12. protection update

Do not skip directly to execution without showing the reasoning summary first.

## Execution Policy

This skill should default to:

- analyze first
- recommend second
- execute only after the user explicitly asks for execution, unless the conversation already established automatic behavior and the profile is clear

Before any live execution:

- restate profile
- restate symbol
- restate whether this is a fresh long or an add-on long
- restate size rationale
- restate default stop-loss logic if it is a first open
- restate total-position protection update if it is an add

For `open_long` execution requests, do not present a naked entry plan.

At minimum, the execution-ready recommendation must include:

- the invalidation basis
- the stop-loss logic
- whether the stop is structure-based, ATR-buffered, or both

## Ambiguity And Unsupported Requests

- If `instId` is missing, ask for it.
- If the user wants short-side logic, route elsewhere.
- If the user wants a fully unattended production service, explain that this version is instruction-only and LLM-driven.
- If the request lacks enough context to distinguish fresh open from add-on management, ask a concise clarification question.

## Output Expectations

When used for evaluation or live decision support, this skill should aim to return:

- symbol and profile
- `1H` long bias summary
- `15m` continuation summary
- `5m` trigger summary
- participation / OI / funding / liquidity summary
- action recommendation
- size rationale
- whether this is a first open or add-on action
- stop-loss plan if first open
- total-position protection update if add-on
- execution recommendation or no-op reason

Preferred top-level statuses:

- `supported_direction`
- `needs_clarification`
- `unsupported_in_current_stack`

## Standard Output Template

```text
🚀 Long Impulse Manager - <INST_ID> [<PROFILE>]
━━━━━━━━━━━━━━━━━━━━
Bias stack: 1H -> 15m -> 5m
Current position: <flat / long + size summary>
━━━━━━━━━━━━━━━━━━━━
Bias:
- 1H regime: <bullish / mixed / not supportive>
- 15m continuation: <strong / usable / weak>
- 5m trigger: <ready / acceptable / poor>

Market quality:
- Volume: <strong / average / weak>
- OI: <supportive / mixed / unstable>
- Funding: <healthy / crowded / extreme>
- Liquidity: <good / acceptable / poor>

Scores:
- Bias regime score: X/5
- Continuation score: X/5
- Trigger quality score: X/5
- Participation score: X/5
- OI quality score: X/5
- Crowding risk score: X/5
- Liquidity execution score: X/5
- Position risk score: X/5
- Total score: X/40

Decision:
- Direction score: X/100
- Entry/addability score: X/100
- Risk pressure score: X/100
- Confidence score: X/100
- Action: <open_long / add_long / hold_long / reduce_long / de-risk_long / flatten_long>
- Size: <tiny / small / medium / aggressive>
- Reason: <short explanation>

Protection:
- Fresh open stop-loss: <required if open_long>
- Stop basis: <5m/15m invalidation, ATR buffer, or structure-failure reference>
- Total-position protection update: <required if add_long>

Execution:
- <recommendation only / ready for execution / execution requested>
```

## User Request Examples

- "做一个 long impulse manager"
- "BTC-USDT-SWAP 现在适合先开多吗"
- "如果已经有多单，现在要不要继续追"
- "给我一版 15m 结合 5m 的多头加减仓判断"
- "先别执行，只给我做多评分报告"
- "如果这次是首次开多，记得把默认止损一起给出来"
- "如果我已经有仓位，就帮我判断加仓还是减仓，并更新整体保护逻辑"
