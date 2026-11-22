# Portfolio Tracker (Python) V1

**系统设计与技术实施方案 · Architecture Baseline V1.0**

本方案的核心目标是建立一个：

- 以 Python 为统一开发语言的个人投资管理工具。
- 以 SQLite 为持久化数据库的可审计交易账本。
- 以 Streamlit 和 Plotly 为交互界面的本地 Web App。
- 可以长期维护、独立测试和逐步扩展的投资分析基础平台。

V1 仍然只关注股票买卖、持仓成本和交易收益，不扩大业务范围。

> 文档说明：架构以原 V1.0 为基线。本版使用两份本地 Trading 212 CSV 核验导入字段、金额规则与持仓重放，证据范围见 §13。代码片段为实施示意。

## 1. 总体技术架构

```text
Development Environment (Windows + WSL2 Ubuntu)
    VS Code Remote WSL
    Python · uv · pytest · Ruff · Pyright · Git
                         │
                         ▼
Data Sources
    Trading 212 CSV          Market Data (prices / FX)
                         │
                         ▼
Python Application Layer
    CSV Importer             Trade Validator
    WAC Engine               Portfolio Engine
    Market Data Service      Reconciliation
                         │
                         ↕ Repository Layer
SQLite Database
    Transactions · Positions · Calculations
    Instruments · Market Data
                         │
                         ↕ Application Services
Streamlit + Plotly
    Dashboard · Holdings · Stock Detail
    Transactions · Import Data · Settings
                         │
                         ▼
Windows Browser · localhost
```

### 1.1 最终技术栈

| 层级 | 技术选型 | 职责 |
|---|---|---|
| 操作系统 | Windows + WSL2 Ubuntu | 本地开发和运行 |
| IDE | VS Code Remote WSL | 代码编辑与调试 |
| 编程语言 | Python | 所有核心逻辑 |
| 依赖管理 | uv | 虚拟环境、安装和版本锁定 |
| 数据模型 | dataclasses | 交易、持仓等数据结构 |
| CSV 读取 | csv 标准库 | 原始交易解析 |
| 数据分析 | pandas | 展示及数据整理 |
| 金融计算 | decimal.Decimal | 高精度成本和收益计算 |
| 数据库 | SQLite | 本地持久化存储 |
| 数据库接口 | sqlite3 | SQL 查询与事务管理 |
| Web 界面 | Streamlit | 页面、筛选和交互 |
| 图表 | Plotly | 买卖点、持仓分布 |
| 市场行情 | Market Data Provider | 独立行情接口 |
| 自动化测试 | pytest | 单元及集成测试 |
| 代码规范 | Ruff | Lint 和格式化 |
| 静态类型检查 | Pyright | 类型安全 |
| 版本控制 | Git | 源码管理与回滚 |

### 1.2 V1 采用模块化单体架构（Modular Monolith）

虽然系统划分为数据层、业务逻辑层、服务层和展示层，但它们都运行在同一个 Python 应用中。

不需要独立运行 FastAPI 后端，不需要 REST API，也不需要 React 前端。

Streamlit 可以直接调用 Python Application Services，后者再调用计算引擎和数据库。这种架构保留了前后端职责分离的好处，同时避免维护多个服务的复杂度。未来如果需要独立 Web API，可以在现有服务层之上增加 FastAPI，而不必重写业务逻辑。

## 2. V1 功能范围

### 2.1 纳入的功能

| 功能模块 | 具体需求 |
|---|---|
| Data Import | 一次上传多份 Trading 212 CSV |
| Data Normalization | 标准化交易字段、币种、时间及费用 |
| Deduplication | 重复成交自动去重，冲突记录报警 |
| Transaction Ledger | 永久保留每次买卖价格、数量和时间 |
| WAC Engine | 逐笔计算加权平均持仓成本 |
| Realized P/L | 每次卖出的已实现盈亏 |
| Unrealized P/L | 当前未实现盈亏 |
| Total Trading P/L | 累计股票买卖盈亏 |
| Multi-currency | 原币与 EUR 双重核算 |
| Holdings | 当前持仓及历史清仓股票 |
| Stock Detail | 完整交易历史及买卖价格图 |
| Dashboard | 组合市值、盈利、行业和主题敞口 |
| Market Data | 股价及汇率更新、缓存、手动覆盖 |
| Reconciliation | 与 Trading 212 实际股数及成本对账 |
| Data Export | 将交易及持仓结果导出为 CSV |

### 2.2 不纳入 V1 的功能

V1 不处理股息、现金利息、入金、出金和现金余额，也不计算整个券商账户的 XIRR、每日历史净值、税务收益、复杂风险模型或自动交易。

暂时也不开发云端部署、用户注册登录、多人协作及 Trading 212 API 自动同步。

股票拆分、合股等影响持仓正确性的公司行为仍需要识别和处理，但不需要为它们建立复杂的独立业务模块。

**V1 的核心不是完整的财富管理系统，而是一个可靠、可追溯的股票成交账本及其可视化工具。**

## 3. 系统设计原则

以下原则应贯穿整个开发过程，也可以直接作为未来使用 Codex 或其他 coding agent 开发项目时的约束条件。

| 设计原则 | 实施要求 |
|---|---|
| Single Source of Truth | SQLite 中的标准化交易账本是交易事实的唯一权威来源 |
| Immutable Raw Data | 原始 CSV 永久保留，不覆盖、不修改 |
| Separation of Concerns | 核心算法、数据库、行情接口及 UI 相互独立 |
| Deterministic Calculation | 相同输入、配置及算法版本必须产生相同结果 |
| Idempotency | 重复导入同一笔交易不能改变持仓和收益 |
| Precision First | 金融计算使用 Decimal，避免提前舍入 |
| Atomicity | 导入和成本重算必须保持数据库状态一致 |
| Auditability | 每个收益数字都能追溯对应的原始成交 |
| Fail Closed | 数据缺失时明确标记异常，不静默生成错误结果 |
| Reproducibility | 所有派生计算结果都能从交易账本重新生成 |
| Privacy by Default | 真实交易数据默认只保存在本地 |
| Test Before Release | 核心计算经过自动化测试后才能用于正式数据 |
| Minimal Complexity | 不引入 V1 不需要的框架、服务和基础设施 |

### 3.1 所有金融计算由 Python 后端完成

本方案不把计算任务分散到前端公式中。

包括以下内容，都应该由 Python 统一计算：

- WAC 与持仓成本；
- 已实现和未实现盈亏；
- EUR 汇率折算；
- 个股仓位与行业仓位；
- 组合市值及累计收益。

Streamlit 只负责获取结果、展示数据、筛选、排序和处理用户操作。

例如，Dashboard 不应独立计算一遍总盈亏，Holdings 再计算一遍。两者应调用相同的 Portfolio Service。

### 3.2 前端不能成为唯一数据存储

`st.session_state`、Streamlit Cache 和 pandas DataFrame 都不是永久账本。

关闭浏览器、重启 WSL2 或重新运行 Streamlit 后，所有已经成功导入的交易必须仍然存在。

### 3.3 计算引擎必须可脱离应用运行

即使不启动 Streamlit、不连接行情 API、不访问 SQLite，WAC Engine 也应该能够使用模拟交易数据独立运行和测试。

这是未来扩展其他应用的基础。

## 4. SQLite 数据库设计

SQLite 将替代原 Google Sheets 中的七个工作表所承担的数据存储职责。

推荐使用一个数据库文件：`data/portfolio.db`。

### 4.1 数据库表结构

| 数据表 | 职责 | 数据性质 |
|---|---|---|
| `accounts` | 券商账户及账户配置 | 主数据 |
| `instruments` | 证券标识、代码、行业、报价币种 | 主数据 |
| `import_files` | CSV 文件来源、哈希与导入状态 | 审计数据 |
| `transaction_sources` | 成交与原始文件、行号之间的关联 | 审计数据 |
| `transactions` | 标准化股票买卖流水 | 核心事实 |
| `position_adjustments` | 拆股、合股、期初持仓等调整 | 核心事实 |
| `trade_calculations` | 每笔成交后的成本及收益 | 派生数据 |
| `positions` | 每个账户与证券的最新持仓状态 | 派生数据 |
| `prices` | 股票市场价格及来源 | 市场数据 |
| `fx_rates` | 当前及历史参考汇率 | 市场数据 |
| `reconciliation_snapshots` | 券商持仓快照与对账结果 | 校验数据 |
| `import_runs` | 每次导入的统计、异常及状态 | 审计数据 |
| `settings` | 应用配置 | 配置数据 |

虽然有多个表，但它们都保存在同一个 SQLite 文件中，不需要部署独立数据库服务器。

### 4.2 数据表之间的关系

```text
accounts (account_id) ────────────┐
                                  ▼
instruments (instrument_id) ───► transactions (account_id, event_id)
                                  │       ▲
                                  │       │
import_files (file_id, hash) ─► transaction_sources (file_id, trade_id)
                                  │
transactions ─────────────────────┘
       │
       ▼
trade_calculations (trade_id, WAC, P/L)
       │
       ▼
positions (shares, cost, realized P/L) ◄ · · prices / fx_rates
```

核心关系示意：行情用于估值，不改变历史成本。

### 4.3 Transactions 字段规格

这张表是整个应用最重要的数据表。

| 字段 | 数据类型 | 用途 |
|---|---|---|
| `trade_id` | INTEGER | 内部主键 |
| `account_id` | INTEGER | 账户标识 |
| `instrument_id` | INTEGER | 内部证券标识 |
| `event_id` | TEXT NOT NULL | Trading 212 CSV 的 `ID`，按原文本保存 |
| `executed_at_utc` | TEXT | UTC 成交时间 |
| `execution_sequence` | INTEGER，可空 | 同一账户、证券和时间内经证据确认的成交顺序 |
| `side` | TEXT | BUY / SELL |
| `raw_action` | TEXT | CSV 原始 Action |
| `quantity` | TEXT (Decimal) | 股数，始终为正 |
| `execution_price` | TEXT (Decimal) | 标准化原币成交价 |
| `quote_currency` | TEXT | 标准化报价币种 |
| `gross_native` | TEXT (Decimal) | 不含费用的原币成交额 |
| `gross_eur` | TEXT (Decimal) | 从 CSV 净额扣除买入费用或加回卖出费用得到的未含费 EUR 金额 |
| `net_eur` | TEXT (Decimal) | 由已核验 EUR 格式的 `Gross Total` 直接读取，买入记含费成本，卖出记净收入，均为正值 |
| `fx_rate_to_eur` | TEXT (Decimal) | 每单位标准化报价货币对应的 EUR，换算方向见 §6.5 |
| `fx_fee_eur` | TEXT (Decimal) | 外汇转换费用 |
| `other_fees_eur` | TEXT (Decimal) | 其他交易费用和税费 |
| `broker_result_eur` | TEXT (Decimal)，可空 | 券商报告的有符号已实现盈亏，仅当 `Currency (Result)` 为 EUR 时填入。空值与零值分开 |
| `validation_status` | TEXT | 数据有效性 |
| `created_at` | TEXT | 记录创建时间 |

原始成交价、原始币种、原始汇率和 CSV 字段通过 `transaction_sources` 关联的原始文件保留，不会因标准化丢失。

### 4.4 数据库约束

数据库必须具备唯一性约束，例如：

```sql
UNIQUE (account_id, event_id)
```

同时启用外键、必填字段的非空约束和 BUY/SELL 检查。成交 `event_id` 必须非空且非空白，缺失时进入审核。

证券使用内部 `instrument_id`。已核验样本中每个 ISIN 对应唯一的 Ticker 和报价币种，但有一个 Ticker 对应两个 ISIN，且两者报价币种不同。该格式按 ISIN 建立初始证券映射，以 `account_id + instrument_id` 分隔 WAC 成本池。行情所需交易所和 Provider 代码另行确认，CSV 未提供交易所列。后续若同一 ISIN 出现不同报价币种、代码或持仓身份，先审核映射，再决定如何重放历史。

### 4.5 金融数据的存储精度

SQLite 的 `REAL` 类型采用二进制浮点数，不适合作为精确的金融账本存储格式。

两份样本的股数与报价最多 10 位小数，原始汇率最多 8 位小数。V1 保留源值精度，以上位数用于覆盖样本测试。建议采用以下存储方式。

| 数据 | 存储方式 |
|---|---|
| 成交 ID | TEXT |
| 成交数量 | Decimal 字符串 |
| 成交价格 | Decimal 字符串 |
| 历史汇率 | Decimal 字符串 |
| 剩余持仓成本 | Decimal 字符串 |
| 已实现盈亏 | Decimal 字符串 |
| 时间 | ISO 8601 UTC 字符串 |

例如：

```text
quantity = "0.1234567890"
execution_price = "105.2300"
```

Python 从数据库读取这些字段后，转换成 `Decimal` 进行计算。

这样做会牺牲一部分直接使用 SQL 数值聚合的便利性，但对于个人投资规模，性能完全不是首先需要解决的问题。

**不能直接对这些 TEXT 金额使用 SQLite 的 SUM、数值排序或隐式浮点转换来完成权威金融计算。** 相关计算和数值排序应在 Python 中进行。

## 5. Trading 212 CSV 导入流程

本方案支持直接从 Windows 浏览器将一份或多份 CSV 上传到 Streamlit。

Streamlit 的文件上传组件原生支持多个文件，但上传文件默认进入内存，因此应用需要主动将原始文件持久化保存。

### 5.1 导入页面设计

```text
Import Transactions
──────────────────────────────────
Target Account: Trading 212 Invest ▾

Upload Trading 212 CSV files
支持多份 CSV 和重叠日期范围

  trades_2025.csv  Ready for validation
  trades_2026.csv  Ready for validation

Import Preview
──────────────────────────────────
Total Records     1,000
New Trades          810
Duplicates          140
Excluded Events      50

[ Confirm Import & Recalculate ]
```

上述按钮、文件和数量仅为 UI 设计示意，非实际导入结果。

### 5.2 导入数据管线

导入应采用两阶段流程：先预览，再确认写入。

```text
Select CSV Files
       │
       ▼
Parse and Normalize
       │
       ▼
Deduplicate and Validate
       │
       ▼
Import Preview
       │
  User Confirmation
       │
       ▼
SQLite Transaction
       ├── Save new trades
       ├── Recalculate WAC
       ├── Update positions
       └── Update import status
       │
       ▼
Commit / Rollback
       │
       ▼
Refresh Application Views
```

### 5.3 数据导入规则

| 处理事项 | 规则 |
|---|---|
| 原始文件识别 | 使用 SHA-256 文件哈希 |
| 账户归属 | CSV 没有账户标识列，由导出记录或用户确认来源，并显式选择目标账户 |
| 成交识别 | 对已核验格式使用账户 + CSV `ID`。两份样本的成交 ID 均有值且唯一，仍需支持冲突检测 |
| 重叠 CSV | ID 与标准化成交事实一致时只记一笔，保留每个来源文件和行号。不同 ID 独立保存 |
| ID 冲突 | 同一账户内相同 ID 的关键成交事实不一致时阻断，记录两个来源供复核 |
| 原始时间 | 读取 `Time (UTC)` 的显式 `+00:00` 偏移，存为统一的 ISO 8601 UTC。导入及重放保留秒级精度 |
| 成交顺序 | 按 UTC 时间重放。同账户同证券同秒且影响逐笔结果时，要求券商记录等可核验证据。CSV 行号用于追溯，ID 或上传顺序仅可用于互不影响事件的稳定展示 |
| 已支持的排除事件 | `Interest on cash`、`Deposit`、`Withdrawal`、`Dividend (Dividend)` 保留在原始归档并计入排除统计，跳过成交必填校验 |
| 未知 Action | 待审核。可能影响持仓或收益时阻断确认，查明后归入支持的调整事件或已知排除事件 |
| 导入成功 | 更新交易账本版本和成功导入时间 |
| 导入失败 | 回滚成交与派生结果，并在回滚后单独保存失败状态和错误记录 |
| 新增早期历史 | 从完整交易历史重新计算 |

已核验的成交 Action 映射为 BUY 或 SELL，同时保留原始文本。

`Market buy`、`Limit buy`、`Market sell`、`Limit sell`、`Stop sell`、`Stop limit sell`。

#### 已核验 CSV 字段映射

两份文件均为 UTF-8 逗号分隔 CSV，表头分别有 25 列和 19 列。按字段名解析，支持 UTF-8 BOM、引号内逗号和选填列缺失。已核验格式的映射如下。

| CSV 列 | 导入规则 |
|---|---|
| `Action`、`Time (UTC)`、`ID` | 分类后校验成交身份及带偏移时间。样本中的 17 条股息均无 ID |
| `ISIN`、`Ticker`、`Name` | ISIN 用于初始证券映射，Ticker 和名称用于展示与核查，Ticker 可重复 |
| `No. of shares`、`Price / share`、`Currency (Price / share)` | 成交股数和报价以正数 Decimal 读取，已核验报价币种为 USD、EUR、GBX |
| `Exchange rate`、`Gross Total`、`Currency (Gross Total)` | 该格式要求总额币种为 EUR，汇率和总额为正数。计算及核验见 §6.5 |
| `Currency conversion fee` 及其币种列 | 两份均有该组列。该格式的空白值按零，非空费用校验 EUR 币种 |
| `Stamp duty reserve tax` 及其币种列 | 长区间文件含该组列，短区间缺失。该格式的缺列或空值按零，非空税费校验 EUR 币种 |
| `Result`、`Currency (Result)` | 可空的券商报告盈亏。样本中全部 345 笔卖出有值，均为 EUR。保留符号与显式零 |
| `Withholding tax`、`Charge amount`、`Deposit fee` 及其币种列 | 样本非空值出现在股息或入金。遇到新的成交收费项目时先核验含费口径 |

非 EUR 总额、成交上出现未覆盖的费用币种或计费方式、缺少关键列等情况进入审核。交易所和真实结算币种需有独立证据时另行记录。Trading 212 Invest 支持以资产币种结算，CSV 的 EUR 报告值应按已核验的格式解释。[Trading 212 多币种账户说明](https://helpcentre.trading212.com/hc/en-us/articles/11669719976093-What-is-a-multi-currency-account)

确认时以当前账本与本批新增事件的并集重新验证身份、冲突和完整性，使用同一事务提交成交、来源关联和派生结果。预览之后若账本已变化，重新生成预览。存在未解决的阻断项时整批暂缓，可连同早期历史或有据可查的期初持仓重新提交。

### 5.4 原始 CSV 的保存方式

建议采用内容哈希命名，避免文件名相同导致覆盖：

```text
data/raw/
├── 2025/
│   └── <sha256>.csv
└── 2026/
    └── <sha256>.csv
```

数据库记录原始文件名、哈希、账户、导入时间和文件路径。

文件归档与 SQLite 写入并不是同一个原子事务。因此建议先完成原始 CSV 归档，再执行数据库事务。数据库回滚时允许保留未成功导入的原始文件，但必须明确记录状态，确保之后可以安全重试。

## 6. WAC 成本计算引擎

按账户与证券分别维护原币和 EUR 成本池，交易与已确认调整按同一时序重放。

### 6.1 独立计算两种成本池

| 成本池 | 定义 |
|---|---|
| Native Price Cost | 以股票报价货币计算，不包含交易费用 |
| EUR All-in Cost | 按历史成交汇率折算，包含实际买入费用 |

这里应注意，两种口径的收益含义不同。

原币口径衡量股票价格本身带来的交易收益；EUR 口径则包含历史汇率变化以及实际交易费用的影响。

### 6.2 买入规则

设交易前持仓股数为 $Q$，账面成本为 $B$，本次买入股数为 $q$，买入成本为 $C$。

$$
Q_{\mathrm{new}}=Q+q
$$

$$
B_{\mathrm{new}}=B+C
$$

$$
WAC_{\mathrm{new}}=\frac{B_{\mathrm{new}}}{Q_{\mathrm{new}}}
$$

对原币成本池，$C$ 取 `gross_native`。对 EUR 成本池，$C$ 取 `net_eur`，其中已含该格式报告的买入费用。

### 6.3 卖出规则

本次卖出释放的成本：

$$
C_{\mathrm{released}}=q\times WAC_{\mathrm{before}}
$$

本次已实现盈亏：

$$
P/L_{\mathrm{realized}}=\text{Sale Proceeds}-C_{\mathrm{released}}
$$

交易后剩余成本：

$$
B_{\mathrm{new}}=B-C_{\mathrm{released}}
$$

部分卖出保持剩余持仓的加权平均成本，计算使用账面成本和持仓比例，展示均价只在输出时舍入。原币卖出收入取 `gross_native`，EUR 卖出收入取已扣费的 `net_eur`。完全清仓时，两套成本池一次性释放各自剩余成本，股数和成本归零，累计已实现盈亏保留。团队在实现前固定 Decimal 运算精度和舍入规则，并用循环小数与 10 位小数股验收。

### 6.4 WAC 计算示例

原互动计算器转换为静态示例：

| 操作 | 数量 | 成交价格 |
|---|---:|---:|
| 第一次买入 | 10 股 | $100 |
| 第二次买入 | 10 股 | $90 |
| 第三次买入 | 10 股 | $80 |
| 卖出前持仓 | 30 股 | WAC $90 |
| 卖出 | 12 股 | $95 |

假设当前价格为 $95：

| 指标 | 结果 |
|---|---:|
| Remaining Shares | 18 股 |
| Average Cost | $90.00 |
| Remaining Cost | $1,620.00 |
| Market Value | $1,710.00 |
| Realized P/L | +$60.00 |
| Unrealized P/L | +$90.00 |
| Total Trading P/L | +$150.00 |

原示意允许调整卖出数量（0–30 股）、卖出价格和当前价格（均为 $50–150）。在该示例条件下，剩余股数为 30 减去卖出数量，已实现盈亏为卖出数量乘以卖出价格与 $90 之差，未实现盈亏为剩余股数乘以当前价格与 $90 之差。完全清仓时，均价不再显示。

示例采用原币 WAC 口径，不考虑手续费和汇率变化。

### 6.5 Trading 212 特殊规则

该 EUR 格式的原始 `Exchange rate` 表示每 EUR 对应的**原始报价单位数**。设 CSV 股数为 `q`、报价为 `p_raw`、原始汇率为 `r_raw`，费用 `f` 为 EUR 换汇费与印花税之和。

| 项目 | 实施规则 |
|---|---|
| USD、EUR 报价 | 标准化价格为 `p_raw`，`fx_rate_to_eur = 1 / r_raw`。样本 EUR 报价的原始汇率均为 1 |
| GBX 报价 | 原始价格与汇率留档。标准化报价为 GBP，`execution_price = p_raw / 100`，`fx_rate_to_eur = 100 / r_raw` |
| 原币未含费金额 | `gross_native = q × execution_price`，币种为标准化报价币种 |
| EUR 入账金额 | `net_eur = Gross Total`。核验未含费金额用 `q × p_raw / r_raw`，买入加 `f`，卖出减 `f`，再与 CSV 总额比较 |
| EUR 未含费金额 | 账本中的 `gross_eur` 由已入账的 `net_eur` 反推，买入减 `f`，卖出加 `f`。汇率推算值用于核验 |
| 券商盈亏 | `Result` 单独保存和对比。WAC 结果由实际入账的成本与收入重算 |

逐笔核验覆盖全部 1,034 笔样本成交，其中 1,000 笔有换汇费，5 笔买入有印花税。上述核验公式与 CSV `Gross Total` 的绝对差额均不超过 €0.005。V1 对该格式采用 €0.01 的导入复核阈值，容纳报价精度、汇率精度和分币舍入。超过阈值时暂停该批入账并展示差额。所有核验保留源数据精度，费用不再对 `net_eur` 二次加减。

清仓后重新买入开始新的持仓成本，历史已实现盈亏继续累计。拆股与合股按有证据的生效时间和比例调整股数，保留两套既有总成本。涉及现金补偿或未支持的证券转换时先审核其数量和成本影响。

### 6.6 WAC 重算策略

V1 不采用复杂的增量计算。

每次新增或修正历史交易、期初持仓或公司行为后，从已确认的账本事件重新计算成本与持仓，并在一个数据库事务内更新派生结果。期初事件记录生效时间、股数、原币成本、EUR 成本和来源。补录覆盖期初日期的早期成交时，先确认期初记录如何被替换或停用，防止重复计入。

对于目前个人交易记录的规模，完整重算相比增量维护更容易验证，也更能避免历史补录导致的错误。

股价更新则不需要重算 WAC，只需要重新计算当前市值和未实现收益。

## 7. Streamlit 用户界面设计

本方案采用多页面 Web App。

建议设置六个主要页面。

| 页面 | 功能 |
|---|---|
| Dashboard | 股票组合整体情况 |
| Holdings | 当前持仓及历史清仓股票 |
| Stock Detail | 单只股票的完整交易时间线 |
| Transactions | 全部标准化历史交易 |
| Import Data | CSV 上传、预览、导入及对账 |
| Settings | 行情、证券分类、账户及备份 |

全局侧边栏负责页面导航、账户选择、显示币种及数据状态。

### 7.1 Dashboard

**Portfolio Dashboard — All Accounts · EUR**

页面顶部 KPI 示例：

| 指标 | 示例值 |
|---|---:|
| Stock Market Value | €52,480 |
| Remaining Cost Basis | €46,200 |
| Unrealized P/L | +€6,280 |
| Realized P/L | +€3,450 |
| Total Trading P/L | +€9,730 |
| Open Positions | 21 |

**Sector Allocation**（环形图示意数据）

| 行业 | 市值占比 |
|---|---:|
| Semiconductors | 42% |
| Industrials | 28% |
| Healthcare | 15% |
| Others | 15% |

**Top Holdings**（横向条形图示意数据）

| 股票 | 市值占比 |
|---|---:|
| FORM | 15.4% |
| TER | 12.8% |
| NVDA | 11.2% |
| CLS | 9.5% |
| VRT | 8.3% |

页面底部显示 `Last Import: 2026-11-20 18:30` 和 `Market Data: Last available close`。

以上全部为虚构数据，仅展示页面布局。

Dashboard 还应包含：

- 个股盈利与亏损贡献排名。
- 前五大和前十大持仓集中度。
- 自定义投资主题敞口，例如 Semiconductor、AI Infrastructure。
- 未完成对账、缺失报价和不完整交易历史的提示。

行业分类与投资主题需要分开。一只股票可以属于一个主要行业，同时属于多个投资主题，因此主题敞口加总可能超过 100%。

### 7.2 Holdings

采用可排序、可搜索、可筛选的数据表。

| 字段 | 含义 |
|---|---|
| Ticker | 股票代码 |
| Company | 公司名称 |
| Shares | 当前持仓数量 |
| WAC Native | 原币平均成本 |
| Last Price | 当前价格 |
| Cost Basis EUR | 当前持仓欧元成本 |
| Market Value EUR | 当前欧元市值 |
| Unrealized P/L | 未实现盈亏 |
| Unrealized % | 当前持仓收益率 |
| Realized P/L | 累计已实现盈亏 |
| Total P/L | 累计买卖盈亏 |
| Weight | 当前股票组合中的权重 |
| Sector | 所属行业 |
| Status | 数据完整性及对账状态 |

默认只显示 Open Positions，可切换至 Closed Positions 或 All Traded Stocks。

选择某行股票后进入 Stock Detail，不再需要像 Google Sheets 那样手动切换单元格下拉菜单。

### 7.3 Stock Detail

这是整个项目最重要的页面，继续采用完整交易历史优先的设计。

**NVDA — NVIDIA · NASDAQ · USD — Open Position**

| 指标 | 示例值 |
|---|---:|
| Shares Held | 18 |
| Average Cost | $90.00 |
| Last Price | $95.00 |
| Market Value | $1,710 |
| Realized P/L | +$60 |
| Unrealized P/L | +$90 |

**Execution Price & Average Cost**

图表设计：以实际成交日期为横轴，价格为纵轴；绿色向上三角标记买入，红色向下三角标记卖出，蓝色阶梯线表示 WAC，灰色虚线表示当前价格。

**Transaction History**

筛选项：`All Trades` / `Buy` / `Sell`。

| Date | Side | Qty | Price | Held | WAC | P/L |
|---|---|---:|---:|---:|---:|---:|
| 2026-10-02 | BUY | +10 | $100 | 10 | $100 | — |
| 2026-10-09 | BUY | +10 | $90 | 20 | $95 | — |
| 2026-10-20 | BUY | +10 | $80 | 30 | $90 | — |
| 2026-11-01 | SELL | −12 | $95 | 18 | $90 | +$60 |

原互动设计可切换买卖记录。以上示例无手续费及汇率变化，不代表真实账户数据。

相比原 V1，这里增加了一条很有价值的图表序列：**加权平均持仓成本随时间变化的阶梯线（WAC Step Line）。**

这意味着除了每次买入和卖出价格之外，你还可以直接观察每次加仓如何改变自己的平均持仓成本。

正式页面中的交易明细仍需包含完整成交日期、时间、股数、价格、成交汇率、手续费、交易后持仓、原币及 EUR 均价、本次及累计已实现盈亏。

Plotly 图表支持缩放、悬停查看成交数量和日期，以及切换图表数据系列。

第一版不强制加入历史市场价格曲线。只有在取得可靠历史行情后，才将真实股价与买卖点叠加。

### 7.4 Transactions

作为完整交易账本，支持按账户、股票、时间范围、BUY/SELL、币种、交易金额和数据状态筛选。

历史交易默认只读。用户可以导出筛选后的 CSV，但不能任意修改原始成交价格或数量。

若发现错误交易，需要通过受控的重新导入、冲突处理或审计修正流程处理。

### 7.5 Settings

集中维护：

| 设置 | 内容 |
|---|---|
| Accounts | Trading 212 账户标识 |
| Instruments | 股票代码、ISIN、交易所 |
| Classification | 行业和投资主题 |
| Market Data | 行情服务和代码映射 |
| Manual Price | 手动覆盖股票价格 |
| FX Rates | 当前估值汇率来源和覆盖值。修改历史成交汇率须走审计修正流程 |
| Reconciliation | 券商持仓快照 |
| Backup | 数据库备份与恢复 |
| Diagnostics | 数据错误、导入和计算日志 |

## 8. Market Data Provider：独立行情模块

这是 Python 版本相比原 Google Sheets 方案最需要重新设计的模块。

移除 Google Sheets 后，我们不再拥有 `GOOGLEFINANCE()`。

### 8.1 行情来源选择

| 数据类型 | V1 推荐来源 |
|---|---|
| 历史成交价格 | Trading 212 CSV |
| 历史成交汇率 | Trading 212 CSV |
| 当前股票价格 | yfinance 候选接口 |
| 当前外汇汇率 | 行情 Provider |
| 每日参考汇率 | ECB 等官方来源 |
| 手动报价 | 用户输入 |
| 历史市场价格 | 后续扩展 |

建议首先评估 `yfinance`，但不将其作为不可替换的核心依赖。它并非 Yahoo 官方 SDK，项目说明也明确要求遵守相关数据使用条款，不能将其视为有稳定性保证的商业实时行情服务。

### 8.2 Provider 接口

建议使用 Python 的 `Protocol` 定义行情接口。

```python
from typing import Protocol


class MarketDataProvider(Protocol):
    def get_quote(self, symbol: str) -> PriceQuote:
        ...

    def get_fx_rate(
        self,
        base: str,
        quote: str,
    ) -> FXQuote:
        ...
```

以上为接口示意，正式实现时会定义 `PriceQuote`、`FXQuote` 和错误类型。

具体实现可以有：

```text
MarketDataProvider
├── YFinanceProvider
├── ManualPriceProvider
└── FutureOfficialAPIProvider
```

### 8.3 行情缓存策略

不允许每次切换股票、调整筛选条件或打开 Dashboard 都重新调用外部行情接口。

推荐流程为：

```text
Refresh Market Data
        ↓
Check local SQLite cache
        ↓
Fetch required prices and FX
        ↓
Validate and normalize
        ↓
Save successful quotes
        ↓
Recalculate current valuation
```

行情记录至少包含：

`instrument_id | price | currency | source | quote_at | fetched_at | status`

其中 `quote_at` 是实际行情时间，`fetched_at` 是程序获取数据的时间，两者必须区分。

默认采用用户主动点击 `Refresh Market Data` 的方式刷新，并在后续考虑缓存过期后自动更新。

若接口失败，应保留最近一次有效报价并标记为 `STALE`，没有任何有效报价则标记 `MISSING`。

**不允许使用零元作为缺失股价的默认值。** 否则会造成组合市值和收益突然大幅下降。

对于缺少报价的证券，应同时提示已完成估值的持仓范围，不能将部分持仓的市值汇总冒充为完整组合市值。

## 9. Python 本地项目目录结构

推荐项目名称：`portfolio-tracker-python`。

WSL2 中的项目路径：`~/projects/portfolio-tracker-python/`。

完整目录结构：

```text
portfolio-tracker-python/
│
├── src/
│   └── portfolio_tracker/
│       ├── __init__.py
│       │
│       ├── core/
│       │   ├── models.py
│       │   ├── wac_engine.py
│       │   ├── currency.py
│       │   ├── portfolio.py
│       │   └── validation.py
│       │
│       ├── adapters/
│       │   ├── trading212/
│       │   │   ├── csv_parser.py
│       │   │   └── csv_mapper.py
│       │   │
│       │   └── market_data/
│       │       ├── base.py
│       │       ├── yfinance_provider.py
│       │       └── manual_provider.py
│       │
│       ├── database/
│       │   ├── connection.py
│       │   ├── repositories.py
│       │   └── migrations/
│       │       └── 001_initial.sql
│       │
│       ├── services/
│       │   ├── import_service.py
│       │   ├── portfolio_service.py
│       │   ├── market_data_service.py
│       │   ├── reconciliation_service.py
│       │   └── backup_service.py
│       │
│       ├── config.py
│       └── cli.py
│
├── app/
│   ├── app.py
│   ├── views/
│   │   ├── dashboard.py
│   │   ├── holdings.py
│   │   ├── stock_detail.py
│   │   ├── transactions.py
│   │   ├── import_data.py
│   │   └── settings.py
│   │
│   └── components/
│       ├── charts.py
│       ├── tables.py
│       └── formatting.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── regression/
│   └── fixtures/
│
├── data/
│   ├── raw/
│   └── portfolio.db
│
├── backups/
├── logs/
├── docs/
│   ├── architecture.md
│   ├── data_model.md
│   └── calculation_rules.md
│
├── .streamlit/
│   └── config.toml
│
├── .vscode/
│   └── settings.json
│
├── pyproject.toml
├── uv.lock
├── .python-version
├── .gitignore
└── README.md
```

### 模块之间的依赖关系

```text
Presentation (Streamlit / CLI)
                │
                ▼
Application Services (Import / Portfolio / Reconciliation)
                │
                ├──► Core Domain (WAC / Models / Validation)
                │
                └──► Infrastructure (SQLite / CSV / Market Data)
```

Core Domain 不依赖 Streamlit、SQLite 或外部行情 API。

尤其需要避免以下几种错误的依赖关系：

- `wac_engine.py` 中出现 `import streamlit`。
- Streamlit 页面中编写一套独立的 WAC 算法。
- CSV 解析器直接操作 UI。
- 行情服务修改历史成交或历史成本。
- 数据库 Repository 负责投资收益业务逻辑。

## 10. WSL2 + VS Code 开发与运行

### 10.1 环境准备

建议使用：

| 工具 | 安装位置 |
|---|---|
| VS Code | Windows |
| WSL Extension | Windows VS Code |
| Ubuntu | WSL2 |
| Python 3.13 | WSL2，由 uv 管理 |
| uv | WSL2 |
| Git | WSL2 |
| Python Extension / Pylance | VS Code WSL 环境 |

Python 3.13 是建议的初始版本，正式开发时需要确认各依赖的兼容性并锁定版本。

项目文件和 SQLite 数据库都放在 WSL2 Linux 文件系统中，不推荐放在 `/mnt/c/` 或普通网盘实时同步目录中。

### 10.2 项目初始化

以下命令仅作为后续实施说明，目前尚不执行。

```bash
mkdir -p ~/projects/portfolio-tracker-python
cd ~/projects/portfolio-tracker-python

git init

uv init --package --python 3.13

uv add streamlit plotly pandas yfinance

uv add --dev pytest pytest-cov ruff pyright

code .
```

核心计算所需的 `decimal`、`csv`、`sqlite3`、`dataclasses`、`hashlib` 等均来自 Python 标准库，无需额外安装。

`uv.lock` 和 `.python-version` 应纳入 Git，保证未来在不同电脑上能够复现环境。

### 10.3 日常开发命令

| 命令 | 用途 |
|---|---|
| `uv sync` | 同步项目依赖 |
| `uv run pytest` | 运行自动化测试 |
| `uv run pytest --cov=portfolio_tracker` | 检查测试覆盖率 |
| `uv run ruff check .` | 代码规范检查 |
| `uv run ruff format .` | 代码格式化 |
| `uv run pyright` | 静态类型检查 |
| `uv run streamlit run app/app.py` | 启动 Portfolio Tracker |

应用启动后，在 Windows 浏览器访问：

`http://localhost:8501`

建议 Streamlit 默认仅监听本地地址：

```toml
[server]
address = "127.0.0.1"
port = 8501

[browser]
gatherUsageStats = false
```

V1 不开放公网访问，也不需要配置 Nginx、Docker 或云服务器。

### 10.4 保留 CLI 操作入口

虽然主要通过 Streamlit 使用，但建议同时保留少量命令行入口。

规划中的命令包括：

```bash
uv run portfolio-tracker import ./trades.csv

uv run portfolio-tracker recalculate

uv run portfolio-tracker validate

uv run portfolio-tracker backup
```

这些是未来需要实现的 CLI 命令，不是目前已经可用的程序。

好处是可以不启动 UI，直接在 VS Code 中调试导入和计算流程。

## 11. Streamlit 特有的开发注意事项

Streamlit 与普通前后端分离应用最大的区别之一，是用户操作组件后，Python 页面脚本通常会重新执行。

因此，需要明确处理以下问题。

| 问题 | 设计要求 |
|---|---|
| 页面 Rerun | 不应自动重复执行数据库写入 |
| Session State | 只保存选择的股票、过滤条件等临时 UI 状态 |
| CSV 上传 | 只有显式确认按钮才能触发正式导入 |
| Cache | 交易导入后清除或更新相关缓存 |
| Market Data | 不在每次 Rerun 时请求行情 |
| Database Connection | 使用短生命周期连接，避免跨线程不安全地共享连接 |
| Navigation | 通过统一状态传递所选证券 |
| Error Handling | 页面显示可理解的错误，不直接暴露底层敏感日志 |

例如，`st.cache_data` 可以缓存只读的交易查询和组合分析结果，但应以数据库版本或快照 ID 作为缓存失效依据。

不建议缓存会修改数据库的操作，也不应该依赖 Streamlit Cache 保存唯一一份交易数据。

## 12. 数据库安全、备份与恢复

这是新版相对于 Google Sheets 必须额外承担的工作。

### 12.1 数据库事务

一次成功的交易导入应该是原子的。

```text
BEGIN
    Insert validated trades
    Recalculate WAC
    Update trade calculations
    Update positions
    Update import status
COMMIT
```

如果发生异常，执行 `ROLLBACK`，旧交易账本和持仓结果仍然保持一致。

Python 标准库的 `sqlite3` 已经支持事务控制和数据库备份，因此 V1 不需要引入额外的数据库框架。

数据库实现还需要启用外键约束、参数化 SQL 查询，并设置合理的并发写入控制。

### 12.2 DEV / PROD 环境分离

| 环境 | 数据来源 | 数据库 |
|---|---|---|
| DEV | 模拟交易 | `portfolio_dev.db` |
| TEST | pytest 测试数据 | 临时 SQLite |
| PROD | 实际 Trading 212 历史 | `portfolio.db` |

测试代码不得默认连接 PROD 数据库。

### 12.3 备份策略

| 备份对象 | 策略 |
|---|---|
| Database Backup | 每次重要导入和数据库迁移之前生成一致的数据库备份 |
| Raw CSV Archive | 保留券商导出的原始文件，用于重新构建历史账本 |
| Git Repository | 备份源码、算法规则、数据库迁移脚本和文档 |

建议通过 SQLite 的 Backup API 创建数据库副本，而不是在数据库仍然运行时简单复制数据库文件。

此外，源码、数据库和原始 CSV 应有独立备份。特别是 WSL2 的整个虚拟磁盘损坏时，保存在其中的 Git 仓库和数据库都可能同时丢失。

因此可以把经过一致性处理的数据库备份保存到另一块磁盘或加密的云存储中，但不建议直接实时同步正在使用的 SQLite 数据库。

### 12.4 Git 忽略规则

建议至少排除：

```gitignore
.venv/
__pycache__/
.pytest_cache/
.ruff_cache/
.coverage

data/
backups/
logs/
/trading212_account_tracking_csv/

*.db
*.db-wal
*.db-shm

.env
.streamlit/secrets.toml
```

`tests/fixtures/` 中不含隐私信息的合成 CSV 则应该纳入 Git。

## 13. 数据验证与 Trading 212 对账

分别校验 CSV 金额、账本重放和券商快照。快照需记录账户、证券、时点、币种及证据来源。

| 检查项 | 处理方式 |
|---|---|
| 当前持仓数量 | 与同账户、同证券、同一时点的券商快照对齐，按双方可核实的股数精度比较 |
| 原币平均成本 | 与券商同口径成本比较，记录券商显示精度 |
| EUR 平均成本 | 检查账本的成交汇率与费用，确认券商快照的币种和成本口径 |
| 已实现盈亏 | 双列保留自行计算值和 Broker Result，报告差额及比较口径 |
| 交易完整性 | 检查负持仓、缺失字段及冲突记录 |
| 市场价值 | 允许由行情获取时间产生的合理差异 |

建议设置以下数据状态：

| 状态 | 含义 |
|---|---|
| `VERIFIED` | 指定账户、证券和时点已完成适用的券商快照对账及校验 |
| `UNVERIFIED` | 已有可重放结果，券商快照仍待核验 |
| `INCOMPLETE_HISTORY` | 已发现缺少必要历史成交、期初成本或持仓调整 |
| `CONFLICT` | 同一成交存在不一致的记录 |
| `INVALID` | 数据不合法 |
| `STALE_PRICE` | 行情过期 |
| `MISSING_PRICE` | 缺少有效行情 |

交易数据完整性和行情新鲜度应作为独立状态管理，避免某只股票因为股价暂时缺失就被错误标记为历史交易不完整。

### 两份 CSV 的核验结果

以下为本次样本的导出区间和行数汇总。重放时暂按两份文件来自同一账户处理，正式导入需确认账户归属。

| 文件名标示的导出区间 | 表头列数 | 数据行 | 买入 | 卖出 | 排除事件 |
|---|---:|---:|---:|---:|---:|
| 2025-10-05 至 2026-10-04 | 25 | 1,375 | 648 | 319 | 408 |
| 2026-10-05 至 2026-11-22 | 19 | 121 | 41 | 26 | 54 |
| 合计 |  | 1,496 | 689 | 345 | 462 |

- 实际成交时间为 2025-10-06 至 2026-11-20 UTC。合并后的 1,034 笔成交均有 ID，样本内和跨文件均无重复成交 ID。成交覆盖 112 个 ISIN、111 个 Ticker。
- 有 4 组同秒成交，组内均为不同 ISIN，现有样本无需推断同一证券的成交先后。
- 按 ISIN、UTC 时间和精确股数从零重放，两份合并时没有出现负持仓，期末为 **19 个正持仓、93 个已清仓 ISIN**。仅用短区间从零重放时，有 **12 个 ISIN** 在某个时点变为负仓。旧方案中“16 只证券无法重建”的结论来自另一份导出区间，本次将上述实测结果作为实施基线。
- 以 CSV 入账金额做 EUR All-in WAC 重放，345 笔卖出中仅 **8 笔**与 CSV `Result` 的绝对差额在 €0.01 以内。两列应分别保留。差异报告用于核实券商成本口径和历史覆盖，发布前仍需取得同一时点的股数及成本快照。

当前样本的重放结果可用于开发验收，账户级对账暂为 `UNVERIFIED`。从零重放通过只能说明样本未触发负仓，仍需通过券商快照确认导出区间外的期初持仓和成本。采用期初事件时，界面显示已实现盈亏的统计起点。若股数与估值输入可靠而成本仍缺失，可展示市值，相关成本及盈亏标记为缺失。

## 14. 测试体系

Python 版本使用 pytest 作为主要测试工具。

### 14.1 单元测试（Unit Tests）

不依赖 SQLite、Streamlit 或联网行情。

| 测试对象 | 测试场景 |
|---|---|
| WAC Engine | 买入、部分卖出、清仓及重新建仓 |
| Multi-account | 不同账户独立成本 |
| Instruments | 不同报价币种及证券映射 |
| Decimal | 高精度小数股与舍入 |
| Currency | USD、EUR、GBP、GBX 转换 |
| Fees | 买卖费用不重复扣除 |
| Validation | 负持仓、异常价格和缺失数据 |
| Corporate Actions | 拆股前后的持仓数量与成本 |

核心验收用例沿用此前示例。

| 操作 | 剩余股数 | WAC | 累计已实现盈亏 |
|---|---:|---:|---:|
| $100 买 10 股 | 10 | $100 | $0 |
| $90 买 10 股 | 20 | $95 | $0 |
| $80 买 10 股 | 30 | $90 | $0 |
| $95 卖 12 股 | 18 | $90 | +$60 |
| $95 卖 18 股 | 0 | — | +$150 |
| $110 买 5 股 | 5 | $110 | +$150 |

### 14.2 集成测试（Integration Tests）

使用临时 SQLite 数据库，验证完整的数据管线。

主要测试：

- 多个 CSV 合并后是否正确去重。
- 导入顺序改变是否影响最终结果。
- 同一份 CSV 重复导入是否保持幂等。
- 补录早期交易是否正确重算。
- 数据库写入失败是否完整回滚。
- 派生计算数据删除后是否能够重建。
- 备份恢复后是否与原始数据库一致。

### 14.3 回归测试（Regression Tests）

将本次验证的 CSV 情形写成不含隐私信息的合成测试案例。

- 同批读取 25 列和 19 列格式，按表头映射。覆盖缺少选填费用列，以及无 ID 的股息仍能计入排除统计。
- USD 买入和卖出直接使用 `Gross Total` 入账，验证换汇费只计一次。例如 2 股@$100、原始汇率 1.25、费用 €1，CSV 买入总额 €161 与卖出净额 €159 均应直接入账。
- GBX 买入同时含换汇费和印花税。例如 10 股、每股 850 GBX、原始汇率 85 GBX/EUR、换汇费 €0.15、印花税 €0.50，标准化报价为每股 £8.50，未含费金额为 €100，入账成本为 €100.65。
- 不同证券同秒可独立重放。同一证券同秒且影响逐笔结果时，无可靠顺序则阻断确认。
- 相同账户、相同成交 ID 的重复行及冲突行，多个 ISIN 共用 Ticker，以及缺少可靠成交 ID 的情况。
- 短区间单独导入时发现历史缺口，合并早期记录后可重放，以及期初调整被历史补录替换后股数和成本无重复。
- 具有循环小数均价的完全清仓，两套成本均准确归零。Broker Result 显式零与缺失值分开，且保留与 EUR All-in WAC 的差额。

每修复一个具有代表性的问题，都应增加对应回归测试。

### 14.4 UI 测试

Streamlit 页面主要验证页面导航、数据筛选、导入确认、股票详情、错误提示及缓存刷新。

可以使用 Streamlit 的 AppTest 测试支持，配合人工检查关键交互。

建议核心算法测试覆盖率达到 90% 以上作为开发目标，但不能用覆盖率代替真实的金融计算用例。

## 15. 完整项目实施路线

建议将项目分成九个阶段。

| 阶段 | 工作内容 | 完成标准 |
|---|---|---|
| P0 — Data Preparation | 确认账户归属、导出范围及期初依据 | 两份样本字段和费用口径已核验，账户归属及期初边界有据可查 |
| P1 — Project Setup | WSL2、uv、Python、Git、pytest、Ruff、Pyright | 本地环境稳定，基础测试通过 |
| P2 — CSV Importer | 解析 CSV、规范化字段、币种、费用及去重 | 两种表头可合并，成交、排除、重复和异常计数正确，金额按 §6.5 验证 |
| P3 — WAC Core Engine | 逐笔计算成本、已实现及未实现盈亏 | 双成本池、含费入账、清仓和历史回补测试通过，同一账户样本可复现 §13 重放汇总 |
| P4 — SQLite Persistence | 建立数据表、迁移、约束、Repository 与事务 | 数据可靠保存并支持回滚 |
| P5 — Application Services | 整合导入、重算、对账及备份 | CLI 可以完整执行数据管线 |
| P6 — Streamlit UI | Dashboard、Holdings、Stock Detail、Import | 核心用户操作可正常完成 |
| P7 — Market Data | 行情接口、FX、缓存、手动覆盖 | 当前组合估值正确，缺失行情有提示 |
| P8 — Validation & Release | 券商快照对账、错误恢复及正式环境配置 | 至少一次同一时点的股数与成本对账，记录 Broker Result 差异。完成备份恢复及重算验证 |

这里有一个重要的执行策略：

**先建立能够独立运行的 Python Core 和 CLI，再开发 Streamlit 页面。**

理想情况下，当 P5 完成时，即使整个项目还没有任何图形界面，也已经能够完成 CSV 导入、成本计算、结果存储和数据验证。

到 P6 时，Streamlit 只是在已经经过验证的系统上增加用户界面。

这样能够显著降低 UI 和业务逻辑相互耦合造成的返工。

## 16. 项目完成后的日常使用

新版的日常操作会比原 Google Sheets 方案更加直接。

1. 在 WSL2 中启动 Portfolio Tracker，通过 Windows 浏览器打开应用。
2. 从 Trading 212 导出新增股票买卖 CSV，可以一次导出多份文件。
3. 打开 Import Data，上传 CSV，查看新交易、重复交易和异常记录预览。
4. 确认导入，系统自动更新交易账本、WAC、持仓和收益。
5. 点击 Refresh Market Data，获取可用的最新股价和汇率。
6. 在 Holdings 和 Stock Detail 查看每次买入、卖出的价格及交易结果。
7. 定期与 Trading 212 实际持仓对账，并进行数据库备份。

停止 Streamlit 或关闭电脑不会丢失已保存的数据。

但与 Google Sheets 不同，**电脑关闭后不能访问本地应用，也不会继续自动获取行情。**

这是当前本地部署方案有意接受的取舍。

## 17. Portfolio Tracker (Python) V1 最终设计基线

**Architecture Baseline 1.0**

| 项目 | 设计基线 |
|---|---|
| Architecture | Modular Monolith |
| Development | Windows VS Code + WSL2 Ubuntu |
| Language | Python |
| Dependency Management | uv |
| Database | SQLite |
| Database Access | sqlite3 |
| UI | Streamlit |
| Visualization | Plotly |
| Financial Precision | Decimal |
| Testing | pytest |
| Code Quality | Ruff + Pyright |
| Version Control | Git |
| Data Source | Trading 212 CSV |
| Cost Basis | WAC · Native / EUR |
| Deployment | Localhost |
| Business Scope | 股票买卖，以及重放持仓所需的已确认期初和公司行为 |

### 推荐与设计取舍

建议将这套方案作为 V1 的正式候选架构。其中三个具体选型是基于当前项目规模作出的建议，尚未经过实际开发验证：

- 使用原生 `sqlite3`，暂不引入 SQLAlchemy 和 Alembic。
- 采用本地单用户 Streamlit 应用，暂不引入 FastAPI、React 或云端部署。
- 优先评估 yfinance，并始终保留可替换的行情接口和手动价格机制。

这三个取舍能在开发复杂度、数据可靠性和未来扩展能力之间取得较好的平衡。

尤其是考虑到未来可能将 Portfolio Tracker 与个股估值、SOTP、同业比较、行业敞口和组合风险分析结合起来，Python 版本会提供一个更合适的技术基础。

但 V1 不应提前实现这些扩展功能。

**最重要的开发目标仍然是：建立一个正确、可审计、可重复计算的股票交易账本，然后在其上构建友好的投资管理界面。**

目前保留两套独立设计：

- **Portfolio Tracker V1**：TypeScript + Google Apps Script + Google Sheets。
- **Portfolio Tracker (Python) V1**：Python + SQLite + Streamlit + Plotly。

本文件记录 Python V1 的实施规格及两份样本的离线核验结果。项目实现和券商快照对账待后续阶段完成。
