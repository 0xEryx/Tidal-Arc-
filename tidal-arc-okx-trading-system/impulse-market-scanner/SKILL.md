---
name: impulse-market-scanner
description: Use this custom OpenClaw skill when a trader wants to scan OKX perpetual contracts for healthy liquidity, elevated volatility, strong participation, and short-term tradability before handing symbols to long-impulse-manager or short-impulse-manager. This skill is for building a shortlist of intraday-ready swaps using OKX market MCP data such as tickers, instruments, order books, candles, open interest, funding, recent trades, and technical indicators. Trigger for requests like '帮我筛适合短线的合约', '找流动性健康且波动大的永续', '做一个给 impulse manager 配套的选币 skill', '先筛标的再决定做多做空', or any request to rank OKX swaps by liquidity health, volatility, and execution quality.
version: "0.1.0"
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "🧭",
        "requires": { "bins": ["okx"] },
      },
  }
---

# Impulse Market Scanner

This is an instruction-only market selection skill for finding OKX perpetual contracts that are suitable for short-term impulse trading.

Its purpose is to help an agent:

- scan the OKX swap universe
- find contracts with healthy liquidity and meaningful short-term volatility
- avoid thin, distorted, or overcrowded names
- produce a shortlist that can be handed off to `long-impulse-manager` or `short-impulse-manager`

This skill is not a direct execution skill and not a directional entry engine. It is the market-selection layer that should run before the impulse managers.

## Use This Skill When

- the user wants to find short-term tradable perpetual contracts
- the user wants a shortlist of high-liquidity, high-volatility swaps
- the user wants to rank symbols before using an impulse manager
- the user wants to avoid low-depth, wide-spread, dead-book contracts
- the user says things like:
  - "帮我筛适合短线的合约"
  - "找几个流动性健康、波动大的永续"
  - "做一个给 impulse manager 配套的选标的 skill"
  - "先筛标的，再决定用 long 还是 short impulse manager"

## Do Not Use This Skill When

- the user already knows the target `instId` and only wants a directional trade decision
- the user wants account, position, or order management
- the user wants a broad medium-term trend strategy instead of intraday contract selection

## Workflow Position

This skill sits before:

- `long-impulse-manager`
- `short-impulse-manager`

Typical flow:

1. run `impulse-market-scanner`
2. identify the best 3-10 candidate swaps
3. hand the chosen symbol to `long-impulse-manager` or `short-impulse-manager`
4. only then consider execution

## Core Operating Principle

The scanner should prefer contracts that are:

- liquid enough to trade cleanly
- volatile enough to matter
- active enough to continue moving
- not obviously broken by extreme spread, empty depth, or dysfunctional participation

The scanner should not confuse:

- raw volume with real tradability
- raw volatility with usable volatility
- crowded perp activity with healthy opportunity

## Data Sources

Use OKX market MCP tools first.

Preferred inputs:

- `market_get_instruments`
- `market_get_tickers`
- `market_get_orderbook`
- `market_get_candles`
- `market_get_open_interest`
- `market_get_funding_rate`
- `market_get_trades`
- `market_get_indicator`

Optional supporting inputs:

- `market_get_mark_price`
- `market_get_price_limit`

Do not require account credentials for this skill.

## Universe Defaults

Unless the user overrides them, use these defaults:

- instrument type: `SWAP`
- settlement preference: `USDT` linear swaps
- instrument state: `live`
- scan style: short-term intraday
- profile mode: `balanced`
- target shortlist size: `5`

If the universe is too large, do a two-stage pass:

1. broad prefilter using instruments and tickers
2. deep scoring on the top candidate subset

## Profile Modes

This skill supports three default scan profiles.

### `balanced`

Use when the goal is to find cleaner, more execution-friendly contracts.

Preference:

- tighter spread
- stronger visible depth
- more stable tape activity
- meaningful but not chaotic volatility
- healthier funding / OI context

Use this as the default mode unless the user explicitly asks for a more aggressive scanner.

### `aggressive`

Use when the goal is to find earlier, faster, or more explosive contracts even if execution quality is slightly less clean.

Preference:

- higher realized short-term volatility
- stronger expansion potential
- faster OI / participation changes
- willingness to tolerate somewhat noisier microstructure

In this mode:

- allow looser liquidity thresholds than `balanced`
- rank volatility expansion and regime-shift potential more heavily
- still reject obviously broken contracts

### `outlier`

Use when the goal is to find abnormal movers, unusual participation shifts, and higher-beta contracts instead of defaulting to the cleanest majors.

Preference:

- outsized `24h` price change
- fast `15m` or `1H` open-interest change
- stronger short-term volatility expansion
- unusual funding or positioning skew
- enough liquidity to trade, but not necessarily major-coin cleanliness

In this mode:

- down-rank ultra-mature majors unless they also show true abnormal expansion
- prioritize anomaly and expansion signals over pure execution cleanliness
- allow more event-driven or higher-beta contracts into the shortlist
- still reject obviously untradeable names

## Operational Threshold Template

Use peer-relative thresholds by default when scanning a mixed swap universe.

This is more stable than hard-coding one absolute number across every contract.

### `balanced` Threshold Template

Use a cleaner, more execution-sensitive filter:

- 24h activity: keep roughly the top `20-30%` of the scanned universe by activity
- spread quality: prefer names in the better half of the universe
- depth quality: require visible book support on both sides and reject obvious thin-book names
- volatility: require at least mid-to-upper-half short-term volatility
- participation: require trade cadence and volume expansion that are clearly alive
- perp activity: prefer mid-to-upper-half OI / perp engagement
- funding distortion: down-rank contracts with obviously stretched funding
- regime-shift watch: only flag if compression / expansion evidence is reasonably clean

### `aggressive` Threshold Template

Use a faster, opportunity-seeking filter:

- 24h activity: keep roughly the top `35-50%` of the scanned universe by activity
- spread quality: allow somewhat noisier names if the book is still usable
- depth quality: allow thinner but still non-broken books
- volatility: prefer upper-half to upper-quartile short-term volatility or rising expansion state
- participation: allow earlier pickup in trades / volume even if not fully mature
- perp activity: allow fast-rising OI / activity even if absolute quality is less clean than `balanced`
- funding distortion: tolerate more crowding than `balanced`, but still reject obvious dysfunction
- regime-shift watch: actively surface earlier compression-break or exhaustion candidates

### `outlier` Threshold Template

Use an abnormality-seeking filter:

- 24h activity: require enough volume to avoid dead contracts, but do not force major-coin dominance
- spread quality: allow materially wider spreads than `balanced`, as long as the market is still tradeable
- depth quality: require minimum usable depth, not major-coin depth
- volatility: strongly prefer upper-quartile short-term volatility, expansion, or abrupt range change
- participation: strongly reward unusual trade cadence and volume burst
- perp activity: strongly reward large positive or negative `OI delta %`
- funding distortion: treat unusual funding as informative, not automatically disqualifying
- anomaly bias: reward names showing joint abnormality across `price`, `OI`, `volume`, `funding`, or `volatility`

### Practical Interpretation Rule

If the user says:

- "更稳一点" / "更干净一点" / "更适合执行"
  - use `balanced`
- "更激进" / "更早抓机会" / "想找快要启动的"
  - use `aggressive`
- "找异动" / "找妖一点的" / "不要 BTC ETH，要像 RAVE 这种"
  - use `outlier`

If the user does not specify, use `balanced`.

## Regime-Shift Watch

This scanner may also produce a separate **regime-shift watchlist**.

This is for contracts that may be approaching a short-term breakout, breakdown, or momentum handoff.

This is not a guarantee of reversal or continuation. It is a shortlist of contracts where structure and participation suggest that a directional move may be near.

The scanner should treat regime-shift candidates as one of these:

- `compression-ready`
- `breakout-watch`
- `breakdown-watch`
- `trend-fatigue-watch`
- `active-but-unclear`

## Scanner Architecture

Use a three-layer scanner:

### Layer 1. Universe Eligibility

Keep only contracts that are structurally tradable:

- instrument state is live
- the contract is a swap
- tick size and lot size are not obviously hostile to clean execution
- 24h activity is not negligible

Use mainly:

- `market_get_instruments`
- `market_get_tickers`

### Layer 2. Liquidity Health

Check whether the contract can actually be traded cleanly:

- best-bid / best-ask spread is not abnormally wide
- top-of-book depth is not too thin
- visible ladder is not empty or gapped
- recent trades suggest an active market instead of a stale book

Use mainly:

- `market_get_orderbook`
- `market_get_trades`
- `market_get_tickers`

### Layer 3. Impulse Suitability

Check whether the contract has enough short-term movement and participation to justify an impulse-style handoff:

- 24h range is meaningful
- `15m` and `5m` volatility are alive
- volume is active enough to support continuation
- open interest is meaningful
- funding is informative but not so extreme that the contract is obviously distorted
- compression / expansion state is informative enough to flag possible regime shifts

Use mainly:

- `market_get_candles`
- `market_get_open_interest`
- `market_get_funding_rate`
- `market_get_indicator`

## Timeframe Stack

Use this default stack:

- `1H` for broad context and major market state
- `15m` for tradability and expansion quality
- `5m` for microstructure and noise check

This scanner is not meant to give a full directional setup. It is meant to answer:

- is the contract active enough?
- is it liquid enough?
- is the movement meaningful enough?
- is it healthy enough to hand off to an impulse manager?
- is it compressing or expanding in a way that suggests a potential regime shift?

## Default Decision Framework

Build **eight** sub-scores on a `0-5` scale.

### 1. `universe_quality_score`

How suitable the contract is at a structural level.

Use mainly:

- instrument state
- swap availability
- tick size
- lot size
- basic market accessibility

### 2. `spread_efficiency_score`

How clean the spread is relative to the contract's price and style of trading.

Prefer:

- tighter spread
- stable best bid / ask
- no obvious gap risk near the top of book

### 3. `depth_resilience_score`

How usable the visible order book is for short-term execution.

Use mainly:

- top `5-20` levels
- bid / ask depth symmetry
- whether depth disappears too quickly away from mid

### 4. `volatility_score`

How alive the contract is.

Use mainly:

- 24h range percentage
- `15m` ATR or `15m` historical volatility
- `15m` Bollinger width or equivalent expansion proxy

### 5. `participation_score`

Whether the movement is supported by actual trading activity.

Use mainly:

- 24h volume ranking
- recent trade cadence
- `15m` volume expansion versus local average

### 6. `perp_activity_score`

Whether the perpetual contract itself is active enough to matter.

Use mainly:

- open interest size
- current funding context
- whether perp positioning looks alive but not dysfunctional

### 7. `execution_cleanliness_score`

Whether the contract looks stable enough for a short-term trader to interact with.

Penalize:

- erratic spread
- visibly thin book
- dead trade tape
- unstable microstructure

### 8. `impulse_fit_score`

Whether the contract belongs in an impulse-style workflow specifically.

Prefer:

- active but not chaotic behavior
- enough movement for intraday follow-through
- enough depth to avoid sloppy execution
- enough participation to justify handing off to a directional manager

## Outlier Evaluation

When the profile mode is `outlier`, also derive a separate `anomaly_score` on a `0-100` scale.

Its purpose is to surface contracts that look unusually active relative to their normal state or relative to the current swap universe.

### Preferred Outlier Inputs

Use these heavily:

- absolute `24h` price change ranking
- `15m` or `1H` `OI delta %`
- short-term `ATR`, `HV`, and `BBWidth`
- volume burst versus recent local baseline
- funding-rate deviation
- recent trade cadence

### Outlier Heuristics

Flag a contract as an outlier candidate when multiple abnormality clusters align:

#### 1. Price + OI Expansion

Look for:

- outsized price movement
- rising `OI delta %`
- active trade tape

This often indicates a live speculative move rather than a stale squeeze artifact.

#### 2. Volatility Shock

Look for:

- `ATR`, `HV`, or `BBWidth` jumping relative to the local baseline
- range expansion on `15m`
- clear movement away from prior equilibrium

#### 3. Participation Shock

Look for:

- sharp increase in recent trade cadence
- volume burst
- top-of-book activity that no longer looks dormant

#### 4. Positioning Distortion

Look for:

- unusual positive or negative funding
- rapid OI build or unwind
- signs of crowding, squeeze, or forced repositioning

This is useful for finding high-risk, high-reward names, but it should not override minimum tradability checks.

### Outlier Labels

The scanner may label outlier candidates as:

- `outlier-long-watch`
- `outlier-short-watch`
- `crowded-long-watch`
- `crowded-short-watch`
- `event-beta-watch`
- `high-risk-two-way`

## Regime-Shift Evaluation

In addition to the main score, the scanner may derive a separate `regime_shift_score` on a `0-100` scale.

Its purpose is to identify contracts that may be near an inflection, breakout, breakdown, or momentum transition.

### What The Agent Can Read

The agent does not need to "see" a chart image to reason about this.

It can use:

- raw OHLCV candles from `market_get_candles`
- technical indicators from `market_get_indicator`
- candlestick-pattern indicators that OKX already supports, such as:
  - `doji`
  - `bull-engulf`
  - `bear-engulf`
  - `bull-harami`
  - `bear-harami`
  - `shooting-star`
  - `hanging-man`
  - `three-soldiers`
  - `three-crows`

So the agent can read both:

- bare K-line structure from OHLCV
- precomputed candlestick / momentum / volatility indicators

### Preferred Regime-Shift Inputs

Use a compact but practical stack:

- `bbwidth` on `15m` to detect compression
- `atr` or `hv` on `15m` to measure volatility state
- `adx` on `15m` to detect trend-strength expansion
- `ema` on `1H` and `15m` to detect regime alignment or transition
- `vwap` on `15m` for intraday equilibrium behavior
- `macd` on `15m`
- `rsi` or `stoch-rsi` on `15m` and `5m`
- candlestick-pattern indicators on `15m` or `5m`
- `open interest`
- `funding`
- recent trade activity

### Regime-Shift Heuristics

The scanner may flag a contract as a potential regime-shift candidate when one or more of these clusters appear:

#### 1. Compression -> Expansion Setup

Look for:

- low or contracting `bbwidth`
- muted or compressed `atr` / `hv`
- then improving participation
- then OI or trade activity starts to wake up

This often belongs in:

- `compression-ready`
- `breakout-watch`
- `breakdown-watch`

#### 2. Trend Expansion Setup

Look for:

- `ADX` rising from a softer base
- `15m` range expansion
- volume expansion
- OI increasing with price movement

This often means the contract is becoming more suitable for an impulse manager even if the final direction still needs downstream confirmation.

#### 3. Reversal / Fatigue Setup

Look for:

- trend extension plus weaker continuation quality
- extreme funding or crowded perp positioning
- candlestick warning patterns
- loss of clean follow-through

This often belongs in:

- `trend-fatigue-watch`

#### 4. Directional Handoff Setup

Look for:

- `1H` context shifting
- `15m` momentum inflecting
- `5m` microstructure no longer behaving like the prior regime

This can justify handing the symbol to either `long-impulse-manager` or `short-impulse-manager` for a more specific directional decision.

## Total Score

Calculate:

- `total_score = universe_quality_score + spread_efficiency_score + depth_resilience_score + volatility_score + participation_score + perp_activity_score + execution_cleanliness_score + impulse_fit_score`

Maximum total score: `40`

### Suggested Interpretation

- `0-12` -> reject
- `13-20` -> weak candidate
- `21-28` -> usable candidate
- `29-34` -> strong candidate
- `35-40` -> top impulse candidate

## Hard Filters

Regardless of total score, strongly down-rank or reject a contract if:

- the instrument is not live
- spread is abnormally wide for intraday execution
- visible depth is too thin
- recent trades are sparse or stale
- volatility exists but liquidity is poor
- funding is so extreme that the market looks dysfunctional

## Preferred Indicator Set

Keep the indicator stack compact and practical.

Recommended:

- `atr` on `15m`
- `hv` on `15m`
- `bbwidth` on `15m`
- `adx` on `15m`
- `ema` on `1H` and `15m`
- `vwap` on `15m`
- `macd` on `15m`
- `rsi` or `stoch-rsi` on `15m`
- candlestick pattern indicators on `15m` or `5m`

Use indicators to support the scanner, not to produce a trade signal.

## Scanner Workflow

### Step 1. Build the eligible swap universe

Use:

- `market_get_instruments`
- `market_get_tickers`

Keep only:

- live swaps
- user-requested quote / settlement preference if provided
- the upper slice by 24h activity

### Step 2. Check liquidity health

For the candidate subset, use:

- `market_get_orderbook`
- `market_get_trades`

Check:

- spread
- visible depth
- tape activity
- whether the book looks tradable rather than decorative

### Step 3. Check volatility and participation

For the remaining candidates, use:

- `market_get_candles`
- `market_get_indicator`
- `market_get_open_interest`
- `market_get_funding_rate`

Check:

- 24h range
- `15m` ATR / HV / BBWidth
- recent participation quality
- OI magnitude
- funding health
- possible compression / expansion state
- possible regime-shift signatures from candles and indicators
- possible abnormality signatures from price / OI / volume / funding interaction

### Step 4. Rank and shortlist

Return:

- top candidates
- rejection reasons for obvious non-candidates
- why each shortlisted contract is suitable for impulse trading
- optional regime-shift watchlist if the setup suggests upcoming market transition

### Step 5. Hand off

If the user wants the next step, hand the selected `instId` to:

- `long-impulse-manager` if the user wants long-side evaluation
- `short-impulse-manager` if the user wants short-side evaluation

## Handoff Guidance

This skill should not make the final long / short decision.

It may optionally label a candidate as:

- `bullish-context`
- `bearish-context`
- `two-way-active`
- `active-but-messy`
- `compression-ready`
- `breakout-watch`
- `breakdown-watch`
- `trend-fatigue-watch`
- `outlier-long-watch`
- `outlier-short-watch`
- `event-beta-watch`
- `high-risk-two-way`

But the real directional decision should be made downstream by an impulse manager.

## Recommended Output Expectations

Return:

- scan universe description
- profile mode used
- filters used
- top candidates ranked by total score
- liquidity summary
- volatility summary
- perp-activity summary
- anomaly / outlier summary if relevant
- regime-shift watchlist if relevant
- key reject reasons
- which candidates are cleanest for `long-impulse-manager`
- which candidates are cleanest for `short-impulse-manager`

## Standard Output Template

```text
🧭 Impulse Market Scanner
━━━━━━━━━━━━━━━━━━━━
Universe: <SWAP / USDT-linear / live>
Scan style: <intraday impulse>
Profile: <balanced / aggressive / outlier>
Shortlist size: <N>
━━━━━━━━━━━━━━━━━━━━
Top candidates:
1. <INST_ID> | Score: X/40
   - Liquidity: <excellent / good / usable / poor>
   - Volatility: <high / moderate / low>
   - Participation: <strong / average / weak>
   - Perp activity: <healthy / crowded / soft>
   - Handoff: <long impulse / short impulse / two-way / avoid>
   - Reason: <short explanation>
   - Outlier note: <optional abnormality note>

2. <INST_ID> | Score: X/40
   - Liquidity: <excellent / good / usable / poor>
   - Volatility: <high / moderate / low>
   - Participation: <strong / average / weak>
   - Perp activity: <healthy / crowded / soft>
   - Handoff: <long impulse / short impulse / two-way / avoid>
   - Reason: <short explanation>

Rejected / down-ranked:
- <INST_ID>: <too wide spread / too thin / dead tape / distorted funding / poor impulse fit>

Regime-shift watchlist:
- <INST_ID> | <compression-ready / breakout-watch / breakdown-watch / trend-fatigue-watch>
  - Trigger basis: <short explanation>
  - Best handoff: <long impulse / short impulse / wait>

Outlier watchlist:
- <INST_ID> | <outlier-long-watch / outlier-short-watch / event-beta-watch / high-risk-two-way>
  - Basis: <price / OI / funding / volatility abnormality summary>
  - Risk note: <high beta / crowded / thin but tradable / event-driven>
```

## User Request Examples

- "帮我找几个适合短线 impulse 的合约"
- "筛一下 OKX 永续里流动性好、波动大的标的"
- "先做 market scanner，再把最好的标的交给 long impulse manager"
- "给我几个适合做 short impulse 的合约"
- "先做候选池，不要直接执行"
- "用 aggressive 模式帮我找几个可能要变盘的合约"
- "除了流动性和波动，还帮我抓一下可能快要 breakout 的标的"
- "不要 BTC ETH，给我找像 RAVE 这种有异动的"
- "用 outlier 模式帮我找几个高风险高收益的永续"
