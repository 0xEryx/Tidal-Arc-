# Tidal Arc / 潮弧

`Tidal Arc` 是一套围绕 OKX 永续合约构建的四技能 agent 交易框架，中文可以叫 `潮弧`。

## 这套体系到底在解决什么问题

很多所谓的 AI 交易 skill，通常会死在两个问题上：

- 要么太宽，想把选标的、方向、开仓、加仓、仓位管理全塞进一个模糊 prompt 里
- 要么太窄，只能回答一个局部问题，完全不知道市场状态、执行质量和保护逻辑

`潮弧` 是为了避开这两个坑才这样设计的。

它把交易流程拆成四层：

1. `impulse-market-scanner`
   负责市场选择
2. `long-impulse-manager`
   负责做多方向的日内决策
3. `short-impulse-manager`
   负责做空方向的日内决策
4. `trend-rebalance-manager`
   负责更慢节奏的趋势维护和仓位滚动

这也是为什么它会比一个“大一统万能技能”更成熟。

## 为什么叫 Tidal Arc / 潮弧

这个名字不是随便取的。

- `Tidal` 代表流动、参与、扩张、压力和 market flow
- `Arc` 代表从选标的，到方向脉冲，再到趋势维护之间的弧线式切换

它不是纯 scalper 栈，也不是纯 trend-following 栈，而是在两者之间形成一条连续路径。

## 为什么说它是一个比较成熟的 agent 交易 skills pack

这套包之所以比较成熟，核心有五点。

### 1. 它把市场发现和方向决策分开了

scanner 不负责最终下结论做多还是做空。

它只负责：
- 找出哪些合约可交易
- 找出哪些合约有异动
- 找出哪些合约值得交给后面的方向决策层

最终方向由 downstream 的 impulse manager 决定。

### 2. 它把 long 和 short 分开了

真实市场里，做多和做空并不是镜像关系。

空头逻辑天然会遇到不同问题：
- squeeze risk
- 负 funding 过拥挤
- 假跌破
- 弱反抽失败

如果把多空硬塞在同一个 skill 里，最后一定会变得模糊。

### 3. 它不是靠单一指标拍板

这套体系不是“RSI 到了就开”或者“均线交叉就上”。

它看的是一个组合：

- 结构
- 波动
- 参与度
- OI
- funding
- 流动性
- 执行质量
- 仓位风险状态

所以它更像一套“结构化判断框架”，而不是单点触发器。

### 4. 它明确包含保护逻辑

很多 AI 交易 skill 只会说“这里可以买”“这里可以卖”。

`潮弧` 这套东西把保护逻辑直接写进了体系标准：

- 首次开仓必须有 invalidation / stop-loss 逻辑
- 后续加仓必须重评整仓保护

这点非常重要，因为它让这套技能更接近真实交易框架，而不是一个只会发信号的玩具。

### 5. 它是围绕 OKX 实际可提供的数据面写的

这套体系不是假设 agent 有什么神秘私有数据。

它是围绕 OKX 这套能力能真实拿到的东西来写的：

- tickers
- order books
- candles
- OI
- funding
- recent trades
- 技术指标
- candlestick pattern indicators

所以它不是幻想型设计，而是落在真实 MCP / CLI 能力上的。

## 系统架构

高层看，`潮弧` 是这样工作的：

1. 先扫描市场
2. 找出干净的或者异动的候选合约
3. 把候选标的交给对应方向的 impulse manager
4. 如果交易逐渐演化成更慢的仓位维护问题，再切换到 rebalance 层

### 实际 handoff 路径

```text
市场全体
  -> impulse-market-scanner
  -> shortlist / watchlist
  -> long-impulse-manager 或 short-impulse-manager
  -> 交易周期拉长
  -> trend-rebalance-manager
```

这个 handoff 路径是刻意设计的。每个 skill 只回答自己该回答的问题。

## Skill 1: `impulse-market-scanner`

### 它是干什么的

这是整套体系的前置市场筛选层。

它负责回答：

- 哪些永续合约流动性够健康？
- 哪些永续合约波动够大？
- 哪些永续合约参与度够强？
- 哪些永续合约的异动足够明显，值得交给 impulse manager？

它不负责直接下单，也不负责给最终方向。

### 它用什么数据

它围绕 OKX 市场面数据工作：

- instruments
- tickers
- order books
- candles
- open interest
- funding
- recent trades
- indicators

### 它怎么 function

它分层工作：

#### 第一层：Universe eligibility

先把结构上就不适合的合约剔掉：

- 不是 live
- 不是目标 swap 类型
- 太死、太小、太不活跃
- granularity 或 market accessibility 太差

#### 第二层：Liquidity health

然后判断这个市场到底能不能碰：

- spread 宽不宽
- depth 够不够
- 盘口有没有断层
- tape 是活的还是死的

#### 第三层：Impulse suitability

再判断它有没有资格交给 impulse manager：

- 24h range 是否有意义
- 短周期波动是否活跃
- volume participation 是否存在
- OI 是否够活
- funding 是否只是极端扭曲

#### 第四层：Regime-shift / Outlier watch

这也是它比普通 screener 更成熟的地方。

它不只是筛“干净合约”，还能额外产出：

- compression-ready
- breakout-watch
- breakdown-watch
- trend-fatigue-watch
- outlier watchlist

### 它的特别之处

这个 scanner 不只是“找好合约”，它还能按不同风格搜索。

它有 3 档模式：

- `balanced`
  偏干净、偏执行友好
- `aggressive`
  偏早、偏快、容忍更多噪音
- `outlier`
  偏异动、偏高 beta、偏高风险高收益

这就是为什么它不只会把 `BTC/ETH` 顶上来，还能专门去找 `RAVE` 这类异常活跃标的。

### 它用了什么方法进行判断

它用的是组合判断，不是单点判断：

- 结构性过滤
- 流动性过滤
- peer-relative threshold
- 波动扩张
- OI / funding 上下文
- candlestick-pattern indicators
- regime-shift heuristics

关键点是：
它不需要真的“看图像版 K 线”，只要能读取 OHLCV 和指标输出，就足以做裸 K 结构推理。

### 它输出什么

它会输出：

- 候选合约 shortlist
- 排名和原因
- regime-shift watchlist
- outlier watchlist
- 建议交给 long 还是 short impulse manager

## Skill 2: `long-impulse-manager`

### 它是干什么的

这是做多方向的日内脉冲决策层。

它要回答的是：

- 这标的现在能不能做多？
- 是首次开多、继续加多、持有、减多、去风险，还是直接平掉？

### 它用什么数据

它用的是多时间框架结构：

- `1H` 背景偏向
- `15m` 延续质量
- `5m` 触发质量
- volume participation
- OI quality
- funding crowding
- liquidity / spread
- 当前仓位状态

### 它怎么 function

它按三层时间框架工作：

- `1H` 决定是否值得考虑多头
- `15m` 决定是否有延续条件
- `5m` 决定当前点位是否适合开或加

然后再通过多项评分把结论映射到允许动作。

### 它的特别之处

它不是单纯的“进场信号 skill”。

它显式支持：

- 首次开仓
- 加仓
- 持有
- 减仓
- 去风险
- 平仓

所以它更像一个 position-management layer，而不是一句“可以做多”。

### 它用了什么方法进行判断

它结合的是：

- EMA 结构
- VWAP
- MACD
- RSI / Stoch RSI
- ADX
- volume expansion
- OI 支撑或脆弱性
- funding crowding
- liquidity quality
- current risk budget

也就是说，它不是靠单一指标拍板。

### 它为什么更成熟

因为它在保护逻辑上不是空的：

- `open_long` 必须有默认 stop-loss / invalidation 逻辑
- `add_long` 必须重评整仓保护

这比单纯说“这里可以多”成熟得多。

## Skill 3: `short-impulse-manager`

### 它是干什么的

这是做空方向的日内脉冲决策层。

它回答和 long manager 同类的问题，但对象是空头仓位。

### 为什么它必须单独存在

因为空头逻辑不是多头把符号反过来就完事。

空头天然会面对：

- squeeze
- 弱反抽失败
- 假跌破反抽
- 负 funding 过度拥挤
- 快速下跌后的方向脆弱性

这些都值得拥有自己的 skill。

### 它用什么数据

它和 long manager 用的是同一类数据：

- `1H` 偏向
- `15m` 延续
- `5m` 触发
- participation
- OI
- funding
- spread / depth
- 当前仓位状态

但解释方式不同。

### 它用了什么方法进行判断

它重点看：

- bearish context
- downside continuation
- weak-bounce rejection
- breakdown continuation
- squeeze risk
- crowded short conditions
- 快跌之后的脆弱性

### 它为什么更成熟

因为它不是 generic directional manager 的附属选项，而是独立处理 short-side 特有风险。

在保护逻辑上，它和 long manager 保持一致：

- 首次 `open_short` 必须有 invalidation / stop-loss
- `add_short` 必须重评整仓保护

## Skill 4: `trend-rebalance-manager`

### 它是干什么的

这是中高时间级别的趋势维护层。

当问题不再是“现在能不能打一笔冲动单”，而变成：

- 当前大趋势是什么？
- 现有仓位怎么维护？
- 应该加、减、持有还是平掉？

就该交给这一层。

### 它用什么数据

它更偏向：

- `1D` 背景
- `4H` regime 确认
- `1H / 4H` 细化
- 大级别结构
- funding
- OI
- volume
- liquidity
- account-level risk controls

### 它的特别之处

它让整套系统不至于永远卡在短线思维里。

没有这层，系统会很会抓 intraday action，但不会处理持仓如何演化。

### 它用了什么方法进行判断

它用的是：

- trend classification
- periodic rebalance scoring
- 结构确认
- crowding / liquidity check
- dynamic sizing
- risk maintenance

它比 impulse 层慢，也更容忍噪音。

## 四个 Skill 在实际中如何协同

### 工作流 A：标准日内交易

1. scanner 找出可交易标的
2. long 或 short impulse manager 判断方向
3. 需要的话执行
4. 后续继续由同一 impulse manager 管理

### 工作流 B：异动型高 beta 交易

1. scanner 跑 `outlier` 模式
2. 把异常活跃合约筛出来
3. 对应 impulse manager 判断这个异动现在还能不能做，还是已经太拥挤
4. 如果交易存活并演化，再考虑交给 rebalance

### 工作流 C：趋势维护

1. 仓位已经存在
2. 问题已经不是“能不能新打一笔”
3. 那就交给 rebalance 层

## 这四个 Skill 共同使用了什么判断语言

从系统层面，它们共享的是一套统一判断语言：

- 多时间框架结构
- 市场微观结构
- 流动性质量
- 波动状态
- 参与度质量
- OI 和 funding
- 评分式动作选择
- 明确保护更新

这点很重要，因为这意味着它们不是四份毫无关系的 prompt，而是一套统一决策语言下的四个专业界面。

## 为什么这套 pack 比普通 prompt bundle 更成熟

因为它不是简单地把四份 prompt 放在一起，而是：

- 每个 skill 都有清晰边界
- 每个 skill 都用清晰的数据面
- 每个 skill 都有明确定义的输出
- 每个 handoff 都是刻意设计的
- 多空分离
- 市场筛选和交易决策分离
- 首次开仓和后续保护分离
- 短线动作和中高周期维护分离

这种 separation of concerns，本质上就是成熟交易系统更稳定的原因。

## Advanced Topic：如何把 Tidal Arc 扩展成多智能体系统

虽然 `潮弧` 现在作为单个 OpenClaw skills pack 已经可以独立工作，但它本身的结构也非常适合继续扩展成 multi-agent system。

原因很简单：这套包已经把交易流程拆成了近似“角色层”的结构：

- 市场发现
- 做多方向评估
- 做空方向评估
- 更慢节奏的趋势维护

也就是说，它在单智能体阶段就已经具备了多智能体拆分的基础。

### 为什么它天然适合做多智能体

很多单智能体交易 prompt 很难扩展，因为一个 agent 同时要负责：

- 扫市场
- 排名候选
- 做方向判断
- 管理风险
- 维护仓位

`潮弧` 更容易扩展，是因为这些责任已经被拆分成了不同 skill。

### 一个实用的多智能体角色拆法

#### 1. Scout Agent

使用：
- `impulse-market-scanner`

职责：
- 扫描 OKX 永续市场
- 生成 shortlist
- 生成 regime-shift 和 outlier watchlist
- 把高质量候选标的交给下游

这个 agent 不负责最终决定做多还是做空。

#### 2. Long Agent

使用：
- `long-impulse-manager`

职责：
- 评估做多方向 setup
- 决定开、多、持有、减仓、去风险、平仓
- 生成多头保护逻辑

#### 3. Short Agent

使用：
- `short-impulse-manager`

职责：
- 评估做空方向 setup
- 处理空头特有风险，比如 squeeze 和 crowded short
- 生成空头保护逻辑

#### 4. Trend Agent

使用：
- `trend-rebalance-manager`

职责：
- 接管已经不再适合纯日内 impulse 逻辑的仓位
- 做更慢的趋势维护、加减仓、持有和平仓判断
- 周期性重评高时间级别结构

#### 5. Coordinator / Risk Agent

这是可选项，但强烈建议有。

职责：
- 把标的路由给正确的下游 agent
- 比较 long 和 short 的输出
- 执行统一风险约束
- 防止 agent 之间互相打架
- 决定是否允许执行

### 一个典型的多智能体工作流

一条比较实用的 multi-agent flow 可以是：

1. `Scout Agent` 扫描市场，输出：
   - shortlist
   - regime-shift watchlist
   - outlier watchlist
2. `Coordinator Agent` 选出优先级最高的候选。
3. 再把候选路由给：
   - `Long Agent`，或者
   - `Short Agent`
4. 如果仓位逐渐脱离纯短线冲击逻辑，就交给 `Trend Agent`
5. 如果系统允许执行，再经过单独的执行或风险闸门

### 共享状态应该怎么设计

如果真要把 `潮弧` 改造成 multi-agent system，建议各个 agent 共享结构化状态，而不是只共享自然语言聊天。

推荐的共享对象：

- `candidate_list`
- `watchlists`
- `position_state`
- `risk_state`
- `execution_permissions`
- `handoff_reason`
- `protection_plan`

底层可以用 JSON、Redis、数据库，或者任意能保留结构化 handoff 的 orchestration layer。

### 推荐升级路径

如果团队技术能力足够，最稳的方式不是一上来就做 fully autonomous multi-agent system，而是分阶段升级：

1. 先用当前单智能体版本
2. 把 `impulse-market-scanner` 拆成独立的 scout agent
3. 把 long 和 short 拆成独立方向 agent
4. 增加 coordinator / risk gate
5. 最后再考虑更深的执行自动化和趋势维护委托

关键点是：`潮弧` 不需要推倒重写，当前这套架构本身就已经在朝 multi-agent 的方向走了。


## 限制与诚实说明

`潮弧` 依然是一套 agent skills pack，不是完整无人值守的 HFT 引擎。

它不声称提供：

- 专有 order flow analytics
- liquidation heatmap
- 保证正确的方向预测
- 完整无人值守执行安全

它的强项不是“无所不知”，而是：

- 结构化判断
- 清晰 handoff
- 基于 OKX 真实可用数据面的实际工作能力

## 文件结构

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
