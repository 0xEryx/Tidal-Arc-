# Tidal Arc

`Tidal Arc` is a four-skill OKX perpetual trading framework designed for agent-driven market selection, directional impulse decision-making, and higher-timeframe trend maintenance.

The system is designed for OpenClaw, but the underlying logic is strategy-first rather than tool-first. In other words, the point of the pack is not just that it contains four prompts. The point is that each skill has a clear role, a clear data surface, a clear decision boundary, and a clear handoff to the next layer.

## What Tidal Arc Is Trying To Solve

Most "AI trading skills" fail in one of two ways:

- they are too broad and try to do scanning, direction, sizing, and management inside one fuzzy prompt
- they are too narrow and can only answer one local question without understanding market regime, execution quality, or protection logic

Tidal Arc is built to avoid both problems.

It breaks the workflow into four separate but connected layers:

1. `impulse-market-scanner`
   The market-selection layer.
2. `long-impulse-manager`
   The long-side intraday directional decision layer.
3. `short-impulse-manager`
   The short-side intraday directional decision layer.
4. `trend-rebalance-manager`
   The slower trend and position-maintenance layer.

This separation is the main reason the pack feels more mature than a single monolithic agent skill.

## Why The Name Fits

The name `Tidal Arc` reflects the design:

- `Tidal` refers to flow, participation, expansion, pressure, and regime movement
- `Arc` refers to the curved path from market screening, to directional impulse action, to slower trend maintenance

The pack is not purely a scalper stack and not purely a trend-following stack. It moves across both.

## Why This Is A More Mature Agent Trading Skills Pack

This pack is comparatively mature for five reasons.

### 1. It separates market discovery from trade direction

The scanner does not pretend to know the final trade direction.

Its job is:
- to find tradable contracts
- to identify abnormal contracts
- to determine whether a contract is healthy enough, active enough, or distorted enough to deserve attention

The final direction is delegated downstream.

### 2. It separates long logic from short logic

Long and short are not mirror images in real markets.

Shorts require different handling for:
- squeeze risk
- negative funding crowding
- failed breakdowns
- bounce-failure structure

By splitting long and short into separate skills, the pack avoids the common failure mode where one generic "directional manager" becomes too vague.

### 3. It uses multi-layer reasoning instead of one-indicator triggers

The pack is not based on a single RSI threshold or a simple MA cross.

It combines:
- structure
- volatility
- participation
- open interest
- funding
- liquidity
- execution cleanliness
- position state

This makes the agent less likely to overreact to one noisy signal.

### 4. It explicitly includes protection logic

Many AI trading packs stop at "buy here" or "sell there."

Tidal Arc includes a system-level protection standard:
- first opens require explicit invalidation or stop-loss logic
- add-on actions require a whole-position protection review

That is a big part of why the pack behaves more like an actual trading framework and less like a signal generator.

### 5. It is structured around MCP-available OKX data

This pack was written around data the OKX stack can actually expose:
- tickers
- order books
- candles
- open interest
- funding
- recent trades
- technical indicators
- candlestick-pattern indicators

So the logic is grounded in a real tool surface instead of imaginary private data.

## System Architecture

At a high level, Tidal Arc works like this:

1. scan the market
2. find clean or abnormal candidates
3. pass a chosen contract to the appropriate directional manager
4. if the trade turns into a slower maintenance problem, switch to rebalance logic

### Practical Handoff Path

```text
Market universe
  -> impulse-market-scanner
  -> shortlist / watchlist
  -> long-impulse-manager OR short-impulse-manager
  -> trade lifecycle evolves
  -> trend-rebalance-manager
```

This handoff path is deliberate. Each skill only answers the questions it is supposed to answer.

## Skill 1: `impulse-market-scanner`

### What It Does

This is the front-end market-selection layer for the entire pack.

Its role is to scan the OKX perpetual universe and answer:

- which contracts are liquid enough to trade?
- which contracts are volatile enough to matter?
- which contracts are active enough to support an intraday impulse trade?
- which contracts are showing abnormal expansion, participation, or positioning?

It is not supposed to place a trade or produce the final direction.

### What Data It Uses

The scanner is built around OKX market MCP data:

- instruments
- tickers
- order books
- candles
- open interest
- funding rate
- recent trades
- indicators

### How It Functions

It works in layers:

#### Layer 1. Universe eligibility

This removes contracts that are structurally bad fits:
- not live
- not the desired swap type
- too inactive
- too awkward in size granularity or market accessibility

#### Layer 2. Liquidity health

This checks whether a contract is actually tradable:
- spread quality
- visible depth
- order-book resilience
- whether the tape is alive or stale

#### Layer 3. Impulse suitability

This checks whether the contract has enough movement and participation to justify passing it to an impulse manager:
- 24h range
- short-term volatility
- volume participation
- OI activity
- funding context

#### Layer 4. Regime-shift and outlier watch

This is what makes the scanner more than a simple liquidity screener.

It can also identify:
- compression-ready names
- breakout-watch names
- breakdown-watch names
- trend-fatigue names
- outlier names with abnormal price / OI / volume / funding behavior

### What Makes It Special

The scanner does not only look for "good contracts."

It can search in three styles:

- `balanced`
  Cleaner, more execution-friendly names
- `aggressive`
  Faster, noisier, earlier-moving names
- `outlier`
  Abnormal, higher-beta, risk-and-reward names like the kind of contract you described as "RAVE-style"

This is one of the strongest parts of the pack, because it gives the agent different search personalities without changing the whole system design.

### What Methods It Uses To Judge

The scanner uses:

- structural filters
- liquidity filters
- peer-relative thresholds
- volatility expansion
- OI / funding context
- candlestick-pattern indicators
- regime-shift heuristics

Important point:
it does not need to literally "see" a chart image. It can read raw OHLCV and indicator outputs, which is enough to reason about bare-K structure.

### What It Outputs

It returns:

- a shortlist of candidates
- ranking and reason summaries
- optional regime-shift watchlist
- optional outlier watchlist
- suggested handoff to long or short impulse logic

## Skill 2: `long-impulse-manager`

### What It Does

This is the aggressive long-only intraday decision layer.

It is responsible for deciding whether a selected contract should be:
- opened
- added to
- held
- reduced
- de-risked
- flattened

### What Data It Uses

It uses:

- `1H` directional bias
- `15m` continuation structure
- `5m` trigger quality
- volume participation
- OI quality
- funding crowding
- liquidity and spread
- current exposure and position state

### How It Functions

It follows a three-timeframe structure:

- `1H` asks whether long bias is even justified
- `15m` asks whether continuation is strong enough to act
- `5m` asks whether the current location is good enough to open or add

Then it scores the setup across multiple dimensions and maps that into one of the allowed actions.

### What Makes It Special

Its special feature is that it is not just an "entry signal" skill.

It explicitly handles:
- first entry
- add-on entry
- hold logic
- reduction logic
- de-risk logic
- full flatten logic

That makes it closer to an actual position-management layer than a generic trading prompt.

### What Methods It Uses To Judge

It combines:

- EMA structure
- VWAP behavior
- MACD
- RSI / Stoch RSI
- ADX
- volume expansion
- OI support or fragility
- funding crowding
- liquidity quality
- current risk budget

It does not rely on any single indicator.

### Protection Logic

This is one of the reasons it feels more mature:

- if the action is `open_long`, a default stop-loss / invalidation plan must be stated
- if the action is `add_long`, protection must be re-evaluated for the whole resulting position

This is a much stronger design than simply saying "go long here."

## Skill 3: `short-impulse-manager`

### What It Does

This is the aggressive short-only intraday decision layer.

It answers the same action question set as the long manager, but for short exposure.

### Why It Exists Separately

It exists separately because short logic is not just long logic with the signs flipped.

Short-side trading has distinct failure modes:
- squeeze events
- bounce failures
- trapped-breakdown reversals
- negative funding crowding
- late-stage short pileups

Those deserve their own logic rather than hidden exceptions inside a generic directional manager.

### What Data It Uses

It uses the same general stack as the long manager:

- `1H` bias
- `15m` continuation
- `5m` trigger
- participation
- OI
- funding
- spread and depth
- current position state

But it interprets them differently.

### What Methods It Uses To Judge

It looks for:

- bearish context
- downside continuation
- weak-bounce rejection
- breakdown continuation
- squeeze risk
- overcrowded short conditions
- directional fragility after fast downside moves

### Protection Logic

Like the long manager:

- first `open_short` requires explicit invalidation / stop-loss logic
- `add_short` requires a whole-position protection review

That symmetry is intentional at the risk-framework level, even though the market logic is different.

## Skill 4: `trend-rebalance-manager`

### What It Does

This is the higher-timeframe maintenance layer.

It exists for situations where the market is no longer best treated as a fast intraday impulse problem.

Instead of asking "should I hit this now," it asks:

- what is the dominant trend?
- how should this position be maintained?
- should size be increased, reduced, held, or flattened on a slower cadence?

### What Data It Uses

It leans on:

- `1D` context
- `4H` regime confirmation
- `1H` / `4H` refinement
- broader structure
- funding
- OI
- volume
- liquidity
- account-level risk controls

### What Makes It Special

This skill stops the whole pack from being trapped in short-term thinking.

Without it, the system would be strong at finding intraday action but weak at managing evolving positions over time.

### What Methods It Uses To Judge

It uses:

- trend classification
- periodic rebalance scoring
- structure confirmation
- crowding and liquidity checks
- dynamic size adjustment
- risk maintenance

It is intentionally slower and more tolerant of noise than the impulse layers.

## How The Four Skills Work Together In Practice

### Workflow A: Clean intraday trade

1. scanner finds a tradable contract
2. long or short impulse manager evaluates direction
3. execution happens if approved
4. later updates continue through the same impulse manager

### Workflow B: High-beta outlier trade

1. scanner runs in `outlier` mode
2. an abnormal contract is surfaced
3. the relevant impulse manager evaluates whether the move is actionable or already too distorted
4. if the trade survives and matures, rebalance logic may take over later

### Workflow C: Trend maintenance

1. a position already exists
2. the question is no longer about a fresh impulse entry
3. rebalance logic becomes the correct layer

## Shared Judgement Methods Across The Pack

Across the system, the agent reasons with a repeatable toolkit:

- multi-timeframe structure
- market microstructure
- liquidity quality
- volatility state
- participation quality
- OI and funding
- score-based action selection
- explicit protection updates

This is important because it means the pack is not just four unrelated skills. It is one decision language expressed in four specialized surfaces.

## Why This Pack Is Better Than A Simple Prompt Bundle

This pack is more mature than a simple collection of prompts because:

- each skill has a bounded role
- each role uses a clear subset of OKX data
- each skill has a defined output shape
- the handoff between skills is intentional
- long and short are separated
- market screening and trade decision are separated
- first entry and add-on protection are separated
- short-term action and slower maintenance are separated

That separation of concerns is exactly what makes real trading systems more stable.

## Advanced Topic: Extending Tidal Arc Into A Multi-Agent System

Although Tidal Arc is fully usable as a single-agent OpenClaw skills pack, its structure also makes it a strong candidate for multi-agent extension.

This is possible because the pack already separates the workflow into role-like layers:

- market discovery
- long-side directional evaluation
- short-side directional evaluation
- slower trend maintenance

In other words, the skills already behave like modular agent roles even before a full multi-agent orchestration layer is added.

### Why Tidal Arc Is Multi-Agent Friendly

Most single-agent trading prompts are hard to scale because one agent is expected to:

- scan the market
- rank symbols
- choose direction
- manage risk
- maintain open positions

Tidal Arc is more extensible because those responsibilities are already split into specialized surfaces.

### Suggested Multi-Agent Role Split

#### 1. Scout Agent

Uses:
- `impulse-market-scanner`

Responsibilities:
- scan the OKX swap universe
- build a shortlist
- build regime-shift and outlier watchlists
- forward high-quality candidates downstream

This agent should not make the final long/short decision.

#### 2. Long Agent

Uses:
- `long-impulse-manager`

Responsibilities:
- evaluate long-side setups
- decide whether to open, add, hold, reduce, de-risk, or flatten
- produce long-side protection logic

#### 3. Short Agent

Uses:
- `short-impulse-manager`

Responsibilities:
- evaluate short-side setups
- handle short-specific risks such as squeeze conditions and crowded shorts
- produce short-side protection logic

#### 4. Trend Agent

Uses:
- `trend-rebalance-manager`

Responsibilities:
- maintain positions that have moved beyond short-term impulse logic
- manage slower trend-following adds, reductions, holds, and flatten decisions
- periodically re-evaluate higher-timeframe structure

#### 5. Coordinator Or Risk Agent

Optional, but strongly recommended.

Responsibilities:
- route symbols to the correct downstream agent
- compare long and short outputs
- enforce risk constraints
- prevent contradictory actions across agents
- decide whether execution is allowed

### Suggested Multi-Agent Workflow

A practical multi-agent flow can look like this:

1. `Scout Agent` scans the market and returns:
   - shortlist
   - regime-shift watchlist
   - outlier watchlist
2. `Coordinator Agent` selects the highest-priority candidates.
3. Each candidate is routed to:
   - `Long Agent`, or
   - `Short Agent`
4. If a position evolves beyond pure intraday impulse logic, hand it to `Trend Agent`.
5. If execution is enabled, route the final action through a separate execution or risk gate.

### Shared State Design

If Tidal Arc is adapted into a multi-agent system, the agents should share structured state rather than only free-form chat context.

Recommended shared objects:

- `candidate_list`
- `watchlists`
- `position_state`
- `risk_state`
- `execution_permissions`
- `handoff_reason`
- `protection_plan`

A simple implementation can use JSON, Redis, a database, or any orchestration layer that preserves structured handoff.

### Recommended Upgrade Path

For teams with enough engineering depth, a staged upgrade path is safer than trying to build a fully autonomous multi-agent system immediately.

Suggested path:

1. Start with the current single-agent pack.
2. Split out `impulse-market-scanner` as a dedicated scout agent.
3. Split long and short into separate directional agents.
4. Add a coordinator / risk gate.
5. Only then consider deeper execution automation and slower maintenance delegation.

The key point is that Tidal Arc does not need to be rewritten to support this evolution. Its current architecture already points in that direction.


## Limits And Honesty

Tidal Arc is still an agent skills pack, not a complete unattended HFT engine.

It does not claim to provide:

- proprietary order-flow analytics
- liquidation heatmaps
- guaranteed directional forecasts
- fully autonomous execution safety

Its strength is not omniscience. Its strength is structured judgement, explicit handoff, and realistic use of the data OKX can actually provide.

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
