# Monoli Platform产品手册
```
version: beta/v1
```

# Unified Trade

## Account API
### 设计目标
管理用户与交易所之间的连接账户（Exchange Accounts / Wallets），并提供该账户的元数据、状态、功能支持情况、统计与交易实体联系数据。
该 API 系列提供的操作主要有：
- 账户创建 / 连接交易所
- 账户更新 / API key 重新设置
- 账户基础资料获取
- 账户相关的统计与活动数据
- 账户余额与图表数据
- 特定账户行为触发（如卖出全部资产）
- 删除 / 重命名账户
- 指标指标查询（交易量/网络信息/存取款信息等）
这些操作围绕 一个中心实体 — Account 构建。

### 数据模型

#### 基础标识与连接信息
| 字段                         | 类型        | 说明                         |
| -------------------------- | --------- | -------------------------- |
| `id`                       | integer   | 账户在 Monoli 内部唯一标识         |
| `exchange_name`            | string    | 交易所名称（例如：Binance Spot）     |
| `market_code`              | string    | 内部市场代码（如 `bybit_spot`）     |
| `name`                     | string    | 用户定义的账户别名                  |
| `created_at`, `updated_at` | timestamp | 账户连接时间/最后更新时间              |
| `api_keys_state`           | string    | API key 当前状态（ok、invalid 等） |
| `api_key_invalid`          | boolean   | API key 是否失效               |

> 此部分主要是账户唯一标识、关联交易所信息和 API 状态字段，用于 UI 展示与分布式系统唯一对接。

#### 可用功能与执行能力标识
| 字段                                              | 类型      | 说明                |
| ----------------------------------------------- | ------- | ----------------- |
| `trading_supported`                             | boolean | 是否支持trading services  |
| `market_buy_supported`, `market_sell_supported` | boolean | 是否支持市价买/卖         |
| `conditional_buy_supported`                     | boolean | 是否支持条件单           |
| `bots_allowed`, `multi_bots_allowed`            | boolean | 是否允许 Bot 策略       |
| `grid_bots_allowed`                             | boolean | 是否支持 Grid Bot     |
| `fast_convert_available`                        | boolean | 是否支持快速资产转换        |
| `hedge_mode_available/hedge_mode_enabled`       | boolean | 是否支持/启用对冲模式       |

> 这些布尔标识直接反映账户连接的交易所是否支持某些交易功能。对于调用方来说，这相当于能力发现接口，使流程在执行前可先做功能判断。

#### 账户动态
| 字段                                                       | 类型     | 说明                 |
| -------------------------------------------------------- | ------ | ------------------ |
| `usd_amount`                               | string | 当前账户总余额（USD, 按BTC计价不再支持） |
| `day_profit_btc`, `day_profit_usd`                       | string | 当日盈亏值              |
| `day_profit_btc_percentage`, `day_profit_usd_percentage` | number | 当日盈亏百分比            |
| `total_btc_profit`, `total_usd_profit`                   | number | 累计盈亏               |
| `primary_display_currency_*`                             | object | 按展示币种汇总的数据         |

> 这是账户在不同维度和展示货币下的资产与收益指标。模型中将 “货币值” 与 “百分比” 分开，适合前端展示或进一步计算。

#### 聚合控制 & 显示控制属性
| 字段                                                   | 说明                      |
| ---------------------------------------------------- | ----------------------- |
| `include_in_summary`, `available_include_in_summary` | 控制是否在总体 summary 中展现该账户  |
| `supported_market_types`                             | 支持的市场类型（如 Spot、Futures） |

### API设计
> 基础 CRUD → 行为操作 → 展示/分析数据
设计核心围绕以下原则：
1. CRUD + 扩展行为
2. 功能行为扩展
3. 统计 & 图形数据

### 权限与安全模型
Platform API 使用签名认证机制（SIGNED），并区分权限作用域，例如：
- `ACCOUNTS_READ`：用于读取账户信息
- `ACCOUNTS_WRITE`：用于创建/修改账户
签名和权限控制确保账户操作安全性。开发者需在创建 API key 时设定相应权限，后端 API 也按照权限分类执行。


## Trading Services

### 什么是Trading Services
Trading Services是Platform的增强型交易工具，用于在连接了交易所账户后创建、管理、执行业务级的交易策略和订单，并支持高级条件逻辑（如止损、止盈、跟踪等）。它不是单个订单，而是一种包含多个订单与条件的“交易计划”。例如：
- 支持同时设置 Take Profit（止盈）与 Stop Loss（止损）
- 支持 Trailing（追踪止盈/止损）
- 支持分级卖出（Step Sell）
- 支持在满足条件时自动执行市场收盘
- 管理整个交易生命周期（从开仓到闭仓）
- 可通过 API 创建与管理交易计划



> 在API 语义上，Trading Services是一个高级交易控制服务实体，比单纯的“下单”更接近于一个"带规则的交易计划/策略模板"。

### API设计
API覆盖Trading的全生命周期：创建 → 更新 → 管理 → 关闭/删除。

### 数据模型

`Trading`的具体数据结构，描述了一个交易的完整“计划与状态”。这个实体主要包含以下部分：

#### 唯一标识与交易基本信息
| 字段 | 含义 |
|------|------|
| `id` | Trading 的唯一系统 ID |
| `pair` | 交易对（如 `USDT_BTC`） |
| `account` | 关联的交易账户（ID、类型、名称、市场） |
| `instant` | 是否为“立即买入/卖出”（简单交易） |
| `status` | 当前 Trading 的状态 |

#### 主要交易属性
**仓位 Position**
描述交易的开仓类型及数量：

- `type`（buy / sell）
- `order_type`（market / limit / conditional）
- `units.value`（开仓量）
- `price.value`（限价时的价格）

#### 条件配置
这些字段表示 Trading 各类执行条件及其状态：

| 子对象 | 说明 |
|---------|------|
| `take_profit` | 止盈条件及分步（可配置多个目标） |
| `stop_loss` | 止损或条件止损的配置 |
| `reduce_funds` | 分级减仓步骤 |
| `market_close` | 通过市价直接平仓的信息 |
| `leverage` | 杠杆信息（如启用、类型、值） |

这些配置允许 Trading 在市场到达某些条件时自动触发对应子订单，而不需要人工追踪。

#### 交易元信息
- `data.current_price`（当前价格）
- `data.average_enter_price`（平均进入价格）
- `profit` / `margin`（盈亏与保证金信息）
- `is_position_not_filled`（开仓是否未完成的标记）

这些字段用于 UI 展示和策略状态判断。

### Trading Services中由系统创建的** Trade Orders **
Trading Services 内部会根据条件自动拆分成多个 "Trade Orders" —— 即真正提交到交易所的订单或市场操作单元，包括：
- Position trade（开仓单）
- Take profit trade（止盈单）
- Stop-loss trade（止损单）
- Reduce funds trade（分级减仓单）
- Market close trade（市价平仓单）

这些 Trades 是 Trading 的下级实体，它们反映了 Trading 的执行进度和单独订单状态。


 ### Trading Services 的典型使用场景

从业务侧看，Trading Services 实际用于实现以下模式：

1. **一单多条件执行**  
   设置 Stop Loss 与 Take Profit 并行工作。

2. **自动跟踪（Trailing）策略执行**  
   价格变化时自动追踪并调整止盈/止损订单。
3. **分级出场（Step Sell）**  
   价格达到多个不同目标时分阶段卖出资产。

4. **强制市价平仓**  
   在指定条件外手动或通过 API 直接结束交易计划。

5. **编程自动化与 webhook 联动**  
   可与 TradingView webhook 等信号源联动触发 Trading 自动执行。

6. 概念对比：Trading vs 单纯订单

| 维度 | 传统订单 | Trading |
|------|----------|-------------|
| 结构 | 单一订单 | 多条件、多订单组合 |
| 管理 | 手动执行 | 自动逻辑触发 |
| 条件 | 简单 Market/Limit | 支持 TP/SL、Trailing、Reduce 等 |
| 适用 | 基础下单 | 智能策略执行 |
| API | 普通交易 API | 智能交易 API |  

Trading Services 本质上是在“订单体系”上叠加了交易生命周期管理的功能。


---
## Quick Trade
