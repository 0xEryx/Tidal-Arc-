# Tidal Arc / 潮弧

`Tidal Arc` 是这套 OKX 永续交易技能体系的名字，中文可以叫 `潮弧`。

这个名字对应这套系统的结构：

- `Tidal` 指市场里的流动、参与、扩张、压力变化
- `Arc` 指从选标的，到方向脉冲，再到趋势维护之间的弧线式切换

它是一个为 OpenClaw 整理出来的四技能交易框架，但本质上是“策略优先”，不是“工具优先”。

## 核心思路

`潮弧` 不是一个“大一统万能 skill”。

它是一套分层流程：

1. 先扫描市场，找出可交易或有异动的合约
2. 再把标的交给单方向的 impulse manager
3. 如果行情进入更慢的趋势维护阶段，再交给 rebalance 层

这样做的好处是可以把这几件事拆开：

- 市场筛选
- 日内方向判断
- 中高周期仓位维护

## 包含的 Skills

### 1. `impulse-market-scanner`

角色：
- 在 OKX 永续里找适合短线的标的
- 识别流动性、波动、参与度和异动
- 额外产出 regime-shift 和 outlier 观察名单

主要模式：
- `balanced`
- `aggressive`
- `outlier`

最适合的使用位置：
- 在调用两个 impulse manager 之前

### 2. `long-impulse-manager`

角色：
- 偏激进的单向做多日内决策引擎
- 支持首次开仓、加仓、持有、减仓、去风险、平仓

时间结构：
- `1H` 偏向
- `15m` 决策
- `5m` 触发

### 3. `short-impulse-manager`

角色：
- 偏激进的单向做空日内决策引擎
- 支持首次开仓、加仓、持有、减仓、去风险、平仓

时间结构：
- `1H` 偏向
- `15m` 决策
- `5m` 触发

### 4. `trend-rebalance-manager`

角色：
- 更慢节奏的高时间级别趋势和仓位维护层
- 用来做周期性的加仓、减仓、持有、平仓管理

时间结构：
- `1D` 背景
- `4H` regime
- `1H / 4H` 细化

## 系统逻辑

`潮弧` 把四个问题拆开处理：

### 市场发现

由 `impulse-market-scanner` 负责。

它回答的是：
- 哪些标的流动性够用？
- 哪些标的波动够大？
- 哪些标的参与度够强？
- 哪些标的异动明显，值得注意？

### 做多方向脉冲

由 `long-impulse-manager` 负责。

它回答的是：
- 现在做多是否合理？
- 应该开仓、加仓、持有、减仓，还是去风险？

### 做空方向脉冲

由 `short-impulse-manager` 负责。

它回答的是同样的问题，只不过对象变成空头。

### 大级别趋势维护

由 `trend-rebalance-manager` 负责。

这一层的存在，是为了让整套系统不至于只剩短线思维。

## 风险风格

`潮弧` 不是保守型体系。

它的 impulse 部分本来就是为中高风险日内交易设计的，特别适合：

- 主动参与方向
- continuation trading
- 顺势加仓
- 事件驱动或异动标的捕捉

其中 rebalance 层相对更稳，但依然是主动管理，不是纯被动持有。

## 统一保护标准

这套体系有一个稳定的原则：

- 首次开仓必须带明确的 invalidation / stop-loss 逻辑
- 后续加仓必须重评整仓保护

这点很重要，因为它保证这套系统不是一个只会发信号、不会保护仓位的东西。

## 建议使用顺序

建议按这个顺序用 `潮弧`：

1. 先跑 `impulse-market-scanner`
2. 选出最值得做的标的或观察名单
3. 交给 `long-impulse-manager` 或 `short-impulse-manager`
4. 如果后面演化成中周期仓位维护，再交给 `trend-rebalance-manager`

## 为什么这个名字合适

`潮弧` 这个名字适合这套系统，因为它本来就是围绕这些东西搭起来的：

- 流动
- 扩张
- 参与
- regime 切换
- 方向延续

它既不是纯 scalper 体系，也不是纯 trend following 体系，而是在两者之间形成一条连续的路径。

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
