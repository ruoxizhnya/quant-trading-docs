---
status: evergreen
type: research
last-verified: 2026-10-14
verified-by: D9 执行 agent——每条事实均带引文与采集日期（附录 A）；标【判断】者为推断/估算，非实测
---

# D9 · 国外市场数据源选型调研报告

> **任务来源**：core 仓 `docs/TASKS.md` §一 D9（六字段权威定义）。
> **上游裁决**：蓝图 [data-factor-unification.md](data-factor-unification.md) §9 #4「国外市场数据源选型 → NT 原生数据通道优先，另调研」；[ADR-031](../adr/adr-031-kernel-pluggable-nt-ready.md)「决议变更记录」未决处置表 #4 同。
> **服务对象**：D7（NT 原生模式国外线的数据通道）与后续国外 connector 工单。
> **边界（D9 ⑥）**：**只调研不实施**——不实现 connector、不签约付费源、不动现有数据链路。
> **读者**：后续「国外 connector」工单的执行 agent——§0 结论与 §7 接口需求清单设计为可直接抄进该工单的 ②上下文 与 ③要求。

**标注约定**：【事实】= 有引文可核（附录 A 给出 URL/文件路径 + 采集日期）；【判断】= 本报告基于事实做的推断、估算或推荐，**不是**供应商承诺。

---

## 0. 结论速览（可直接抄进后续工单 ②上下文）

1. **主选：Databento（`nt_native` 通道）**。在 NT 官方 adapter 清单（rc6 与 rc7 官方表逐行一致）中，它是**唯一覆盖国外股票/期货的 Data-only adapter**——其余股票/期货类 adapter（IBKR 等）均为 `Data/Execution` 且需经纪账户，另两个 `Data` 型 adapter（Blockchain / Tardis）只做链上与 crypto 历史，数据模型自带四时间戳（`ts_event`/`ts_recv`/`ts_in_delta`/`ts_out`，ns 精度、PTP/GPS 同步、`ts_recv` 按 symbol 单调），**时间语义既「可声明」又「由引擎执行层强制」**（NT 回测按 `ts_init` 稳定排序喂数据）。【事实+判断】
2. **零成本样例通道：交易所 crypto（Binance / Bybit / OKX，均 NT 官方 adapter）**。供 D7「一个国外市场样例 run」立即跑通用；不入主链路。【判断】
3. **外部候选（需自写 connector 进规范层）**：Polygon（已更名 **Massive**）、Tiingo、Tushare `us_daily`。三者 **无 NT 官方 adapter** ⇒ 接入成本高于 Databento。【事实+判断】
4. **一票否决已生效**：Yahoo/yfinance、`Tushare us_daily_adj`、以及 Polygon/Tiingo 的**复权序列**（`adjusted=true` / `adjClose`）**时间语义声明不了**（值会被未来事件回溯改写）⇒ **排除**；但同源的**未复权**形态可以保留（同一源两种用法，一个死一个活）。【事实】
5. **落地形态建议**：D7 打样走 NT 原生 adapter 直连 catalog；**正式链路必须回灌 D1 规范层**（否则违反「不允许绕过规范层直连」红线，见 §6.3）。

---

## 1. 调研范围与方法

### 1.1 范围

| 维度 | 覆盖 | 不覆盖（边界外，另立） |
|---|---|---|
| 市场 | 国外股票（美股为主）、国外期货、crypto | A 股（国内线，已有 Tushare/mootdx）、中国香港市场（license 与通道另成一档，见 §6.5） |
| 数据类 | 行情（OHLCV / tick / 订单簿） | 国外基本面、另类数据（属未来 connector 工单） |
| 频率 | 日频、分钟频、事件流 | 逐笔撮合精度升级（另立工单） |

### 1.2 证据采集方式（可复现）

1. **NT 侧事实取本地检出的冻结版本**：`C:\Users\ruoxi\Workspace\nautilus_trader` 仓库，`git show v2.0.0rc6:...`（本项目侧车 pin 为 `nautilus_trader==2.0.0rc6`，见 core 仓 `nt-sidecar/requirements.txt`）——**不用记忆、不用在线文档**。官方 adapter 清单在 rc6 与 rc7 两个 tag 下逐行一致。【事实】
2. **供应商事实取官方页面**（定价页 / API 文档），采集日期 2026-10-14；第三方对比页仅作旁证，凡与官方冲突**以官方为准并注明冲突**。
3. **硬判据引用冻结契约**：D2「时间语义 + 频率口径 fail-closed 拒收」、蓝图 §4.2 时间语义红线。

---

## 2. 硬判据（一票否决）：时间语义可声明性

**依据**：蓝图 §4.2「connector 注册的 schema 里**必须声明时间语义**（按公告日可见 / 收盘后可见 / 次日开盘可见 / …），声明不了或声明错了 → **fail-closed 拒收**」+ D2「未声明、声明非法、或声明与数据实际时间戳形态矛盾 → 注册失败/入库拒收，**不静默放行、不降级**」。D9 ③2 明写：**时间语义声明不了的源直接排除（D2 fail-closed 红线在选型期生效）**。

把「可声明」拆成三条可判定子项（本报告的执行口径）：

| 子项 | 判定问题 | 不满足即排除 |
|---|---|---|
| **H1a 可定位** | 每个数据点是否自带事件/窗口时间戳字段（精度足够区分可见先后）？ | 只有"日期/序号"、无事件时刻 ⇒ 排除 |
| **H1b 不重述** | 该数据点在可见之后，其值**是否会被未来事件（拆股/分红/修正/重述）回溯改写**？ | 会改写（restated）⇒ 排除；若同源另提供**未复权原值 + 当期因子**，则该形态保留 |
| **H1c 可见时点可给** | 能否给出"这一行数据什么时候才能被拉到/被用到"（收盘后 X 时 / 次日 / 实时）？ | 说不出 ⇒ 排除 |

**为什么 H1b 是硬判据**（而非"注意项"）：重述序列在回测里表现为「过去某天的值今天变了」——回测曲线照样漂亮，但那笔交易在**当时根本不可能按那个价格成交**。这与蓝图 §4.2 的行业依据（Databricks / SageMaker Feature Store：泄漏极难事后发现）是同一个失效模式，且**H1b 型泄漏无法用 PIT 查询修复**，因为"当时的值"根本没有留存。

---

## 3. 对比表（覆盖市场 / 频率 / 时间语义可声明性 / 价格与配额 / license）

> 价格为 2026-10-14 采集的官网标价，单位为美元/月；凡第三方与官方冲突处已注明。

### 3.1 NT 原生通道组（`nt_native`：NT 官方 adapter 直进 NT 数据模型/catalog）

| ID | 源 / 通道 | 覆盖市场 | 频率 | 时间语义可声明性 | 价格与配额 | license |
|---|---|---|---|---|---|---|
| **N1** | **Databento**（NT 官方 adapter，`Data`） | 美股：**全部美股交易所与 ATS，20,000+ 股票与 ETF**（`EQUS.MINI` = 自有混合盘口；`EQUS.SUMMARY` = 全市场延迟 + consolidated EOD OHLCV）；另有 CME/CBOT/NYMEX/COMEX 期货期权、OPRA 期权、Eurex 等，合计 **50+ 交易场所、3M+ 符号** | `ohlcv-1s/1m/1h/1d` → NT `Bar`；另 MBO/MBP-1/MBP-10/TBBO/TRADES/BBO-1s/1m/definition/statistics/status/imbalance | **强（H1a✅ H1b✅ H1c✅）**：DBN 四时间戳 ns 精度，`ts_recv` 按 symbol 单调、PTP/GPS 亚微秒；NT 映射规则显式（quote/trade 用 `ts_recv`→`ts_event`；bar 用区间时间戳，`bars_timestamp_on_close` 控制开/闭）；**回测按 `ts_init` 稳定排序喂入** ⇒ 可见性由引擎强制 | 注册送 **$125 credits**（仅历史，6 个月有效，每团队一套）；按量付费 from **$0.50/GB**（未压缩二进制；CSV/JSON 转码免费；batch 30 天内可重复下载不另计费）；**Standard $199/月**（个人 ≤2 设备、含实时、**免 license 费**、L0 含 16+ 年、L1 12 个月、L2/L3 1 个月）；Plus $1,750/月、Unlimited $4,500/月（年合同，含外部分发权） | 订阅协议；**再分发取决于数据集**（Databento 自身不加限制，转嫁原始发布方限制，多数数据集 24 小时后可内/外部分发）；Standard 定位个人与小型团队 |
| **N2** | **交易所 crypto**（Binance / Bybit / OKX / Coinbase / Kraken / Deribit / Hyperliquid 等，均 NT 官方 adapter） | 各所现货与永续/交割合约 | 1s kline 起（Binance 现货订阅与历史请求支持 1s）→ NT `Bar`；另有 trade/quote/book | **强**：klines 带 openTime/closeTime（ms），历史不可变；NT 侧统一为 `Bar` 的 `ts_event`/`ts_init` | **免费公开行情**，无 license 费；限速按端点权重（NT 集成文档有逐端点权重表） | 各所公开 API 条款；内部研究用途 |
| **N3** | **Interactive Brokers**（NT 官方 adapter，`Data/Execution`） | 美股/期权/期货/外汇/全球多市场 | 历史 bar + 实时；深度受订阅档位限制 | **中（H1c 需核）**：历史 bar 有时间戳，但**可取性受账户订阅与 pacing 限制**，且 IB 亦提供复权口径 ⇒ 落库前必须先把口径钉死 | 需 IB 账户 + 各交易所行情订阅（按交易所付费） | IB 客户协议；**数据不可再分发** |

### 3.2 外部候选组（无 NT 官方 adapter ⇒ 需自写 connector 进规范层）

| ID | 源 | 覆盖市场 | 频率 | 时间语义可声明性 | 价格与配额 | license |
|---|---|---|---|---|---|---|
| **E1** | **Polygon（现 Massive）** | 美股（全交易所 + 暗池）、期权、外汇、crypto、指数 | 聚合 bar（任意 multiplier × minute/hour/day/…）、trades、quotes、快照、WebSocket | **可用但需选对用法**：`t` = 聚合窗口起点（ms）⇒ H1a✅；**默认 `adjusted=true`（按拆股复权）⇒ H1b❌**；改用 **`adjusted=false`** 取未复权原值 ⇒ H1b✅（复权须自行按 PIT 应用） | Basic 免费：**5 次/分钟、日终数据、2 年历史**；Starter $29/月（无限调用、15 分钟延迟、5 年）；Developer $79/月（10 年 + trades）；Advanced $199/月（实时、15+ 年、quotes） | 各档标注 **Individual Use**（个人/非专业用途）；**禁止再分发**；商用与期权/其他资产类别单独定价 |
| **E2** | **Tiingo** | 美股 + 中国股票 + ETF/基金（约 99k–108k 全球证券）、crypto、外汇、新闻 | EOD 为主（30+ 年历史）；IEX 实时与 crypto 走 WebSocket | **可用**：同时提供 **raw（open/high/low/close/volume）** 与 **调整后（adjOpen…adjClose）+ 当期 `divCash`/`splitFactor`** ⇒ raw 形态 H1b✅；官方明示「多数美股价格 17:30 ET 可用，交易所修正可能持续到 20:00 ET，我们会在晚间持续更新」⇒ H1c 可给（保守声明 = 当日 20:00 ET） | Starter 免费：**50 次/小时、1,000 次/天、500 唯一符号/月、1 GB/月**；Power **$30/月 个人 / $50/月 内部商用**（10,000 次/小时、100,000 次/天、40 GB/月、全符号） | **Internal Use Only**（仅自用，不得展示或分享给其他个人/组织）；再分发单独授权；基本面与长历史新闻为付费加项 |
| **E3** | **Tushare `us_daily`** | 美股（NYSE/NASDAQ/OTC 等） | **仅日线**（未复权） | **可用**：`trade_date` 日频 + **未复权**价格 ⇒ H1a✅ H1b✅（复权另算，见 X2）；H1c = 收盘后可见（保守起见按实际可拉取时间校准，见 §7 I-05） | **120 积分可试用**；正式权限需开通；单次最大 6,000 行，可按日期循环或分页 | Tushare 平台协议/积分体系；**本项目已有 token 与 connector**（core 仓 `pkg/data/tushare.go`），边际成本最低 |

### 3.3 排除组（硬判据 H1b 不通过 / 无合法通道）

| ID | 源 | 排除理由（硬判据） | 证据 |
|---|---|---|---|
| **X1** | Yahoo Finance / `yfinance` | **H1b❌ + 无合法通道**：非官方端点（无 SLA、历史上断过），`Adj Close` 为**全历史重述**序列；且其条款不提供可商用授权 | 第三方对比（2026-06-04）：「Free, unofficial… No formal SLA… Endpoints have broken historically without notice… Not appropriate for production」 |
| **X2** | Tushare `us_daily_adj`（美股复权行情） | **H1b❌**：官方原话「价格 * 复权因子 = 复权价格…**复权因子历史数据可能除权等被刷新，请注意动态更新**」——**供应商亲口承认历史会被未来除权事件刷新** | Tushare 官方文档 `us_daily_adj` 注 |
| **X3** | Polygon/Massive 聚合的 **`adjusted=true`（默认）** 序列 | **H1b❌（模式级）**：同一行的值会随未来拆股被改写 | Polygon 官方文档：「Whether or not the results are adjusted for splits. **By default, results are adjusted.** Set this to false to get results that are NOT adjusted」 |
| **X4** | Tiingo 的 **`adj*` 复权序列**（CRSP 方法，含拆股与分红） | **H1b❌（模式级）**：复权值含未来分红/拆股回溯；且官方明示晚间仍在更新当日价 | Tiingo 官方文档：「adjustment methodology… (CRSP)… incorporates both split and dividend adjustments」+ 修正窗口说明 |

> **注意 X2/X3/X4 是「同源不同用法」**：源本身不废，**废的是那个会被回溯改写的字段**。E1/E2/E3 之所以通过，是因为它们都提供未复权原值（或当期因子），使「当时的值」可留存——这正是 D2 声明能写实的前提。

---

## 4. 硬判据筛查记录（逐条，含引文）

| ID | H1a 可定位 | H1b 不重述 | H1c 可见时点 | 结论 | 关键证据（引文/出处，采集 2026-10-14） |
|---|---|---|---|---|---|
| N1 Databento | ✅ | ✅ | ✅ | **通过（主选）** | 「`ts_event` Matching-engine-received timestamp in ns；`ts_recv` Capture-server-received timestamp in ns」；「Quote and trade-like schemas map Databento `ts_recv` to Nautilus `ts_event` because it is more reliable and monotonically increases」；「`ts_recv`… synchronized against UTC with sub-microsecond accuracy… always guaranteed to be monotonic for any given symbol」；「Data is ordered by `ts_init` using a stable sort」 |
| N2 crypto 交易所 | ✅ | ✅ | ✅ | **通过（样例）** | Binance 集成文档：历史 bar 请求返回 `Bar`；现货支持 1 秒 klines；klines 带开/闭窗时间 |
| N3 IBKR | ✅ | ⚠️ 需先钉口径 | ⚠️ 受订阅/pacing | **条件通过**（执行通道为主） | NT 官方 adapter 表列为 `Data/Execution`；无公开重述说明 ⇒ 落库前须自行声明并测试（属后续工单核实项，非本报告可结） |
| E1 Polygon/Massive | ✅（`t`=窗口起点 ms） | ❌ 默认态 / ✅ `adjusted=false` | ✅ | **通过（限定用法）** | 「`t` integer The Unix millisecond timestamp for the start of the aggregate window」；「By default, results are adjusted… Set this to false」 |
| E2 Tiingo | ✅ | ✅（raw + 当期 `splitFactor`/`divCash`） | ✅（17:30 ET 可用，20:00 ET 前可能修正） | **通过（限定用法）** | 「Both raw prices and adjusted prices are available」；「Most US Equity prices are available at 5:30 PM EST, however exchanges may send corrections until 8 PM EST. As we obtain corrections, we update prices throughout the evening」 |
| E3 Tushare `us_daily` | ✅（`trade_date`） | ✅（**未复权**） | ✅ | **通过（仅日线）** | 「接口:us_daily 描述:获取美股行情(**未复权**),包括全部股票全历史行情」；「要求:120积分可以试用查看数据」 |
| X1 yfinance | ⚠️ | ❌ | ❌ | **排除** | 见 §3.3 引文 |
| X2 `us_daily_adj` | ✅ | ❌ | ❌ | **排除** | 「复权因子历史数据可能除权等被刷新，请注意动态更新」 |
| X3 Polygon `adjusted=true` | ✅ | ❌ | ✅ | **排除（模式级）** | 见 §3.3 引文 |
| X4 Tiingo `adj*` | ✅ | ❌ | ✅ | **排除（模式级）** | 见 §3.3 引文 |

**筛查结论**：10 个候选项中 6 通过（N1/N2/N3/E1/E2/E3）、4 排除（X1–X4）。排除项**全部**倒在同一条判据上：**H1b 重述**——即「过去的值会被未来改写」。这印证了把它设为一票否决的必要性：它是唯一一个**事后无法用查询手段修复**的失效模式。

---

## 5. 候选登记表（机器可读，供工单与校验脚本消费）

> 下表是本报告的**唯一登记表**，**附录 B** 的校验脚本直接解析它；后续 connector 工单可直接抄 `id` 与 `declared_time_semantics`。

```json
{
  "task": "D9",
  "verified_on": "2026-10-14",
  "candidates": [
    {
      "id": "N1",
      "name": "Databento（NT 官方 adapter）",
      "channel": "nt_native",
      "markets": "US equities / CME-CBOT-NYMEX-COMEX futures+options / OPRA / Eurex",
      "frequency": "ohlcv-1s,1m,1h,1d + MBO/MBP-1/MBP-10/TBBO/TRADES/BBO",
      "ts_declarable": true,
      "verdict": "recommend_primary",
      "declared_time_semantics": "按可见时点声明：ts_event(事件)/ts_recv(采集，单调)；bar 默认打区间闭时刻戳(bars_timestamp_on_close=true)，可见时点 = bar 收盘时刻",
      "declared_frequency": "日频 1d / 分钟频 1m / 秒频 1s / 事件流(MBO/MBP/TRADES)",
      "cost_note": "$125 credits(历史,6个月) 起步；按量 from $0.50/GB；Standard $199/月",
      "license": "订阅协议；再分发取决于数据集，多数 24h 后可分发",
      "evidence": "nautilus_trader@rc6 ADAPTERS.md(Official adapters 表) + docs/integrations/databento.md + databento.com/pricing + databento.com/docs/standards-and-conventions/common-fields-enums-types"
    },
    {
      "id": "N2",
      "name": "交易所 crypto（Binance/Bybit/OKX…NT 官方 adapter）",
      "channel": "nt_native",
      "markets": "Crypto spot & derivatives（各所）",
      "frequency": "1s kline 起 → NT Bar；trade/quote/book",
      "ts_declarable": true,
      "verdict": "recommend_stage0",
      "declared_time_semantics": "klines openTime/closeTime(ms)；历史不可变；可见时点 = 窗口闭时刻",
      "declared_frequency": "分钟频 1m（可到 1s）/ 日频由 1m 聚合",
      "cost_note": "免费公开行情，无 license 费；限速按端点权重",
      "license": "各所公开 API 条款，内部研究用途",
      "evidence": "nautilus_trader@rc6 ADAPTERS.md + docs/integrations/binance.md（Bars and historical market data 节）"
    },
    {
      "id": "N3",
      "name": "Interactive Brokers（NT 官方 adapter）",
      "channel": "nt_native",
      "markets": "US/global equities, options, futures, FX",
      "frequency": "历史 bar + 实时（深度受订阅档位限制）",
      "ts_declarable": true,
      "verdict": "conditional",
      "declared_time_semantics": "历史 bar 带时间戳；口径（复权/未复权）须在 connector 内显式钉死并测试",
      "declared_frequency": "日频 / 分钟频（视订阅）",
      "cost_note": "需 IB 账户 + 各交易所行情订阅",
      "license": "IB 客户协议；数据不可再分发",
      "evidence": "nautilus_trader@rc6 ADAPTERS.md（Data/Execution）；口径与 pacing 属后续工单核实项"
    },
    {
      "id": "E1",
      "name": "Polygon（现 Massive）",
      "channel": "external",
      "markets": "US equities(全交易所+暗池) / options / FX / crypto / indices",
      "frequency": "聚合 bar 任意倍率×timespan；trades/quotes/snapshot/WebSocket",
      "ts_declarable": true,
      "verdict": "alternative",
      "declared_time_semantics": "仅 adjusted=false 形态可用；t = 窗口起点(ms)，可见时点 = 窗口闭时刻；复权须自行按 PIT 应用",
      "declared_frequency": "日频 / 分钟频 / 事件流(trades,quotes)",
      "cost_note": "Basic 免费 5次/分+2年；Starter $29；Developer $79；Advanced $199",
      "license": "Individual Use；禁止再分发",
      "evidence": "polygon.io/docs 聚合接口文档（adjusted 参数与 t 字段）+ polygon.io 定价页（Individual Use）"
    },
    {
      "id": "E2",
      "name": "Tiingo",
      "channel": "external",
      "markets": "US + China equities, ETFs/mutual funds, crypto, FX, news",
      "frequency": "EOD 为主（30+ 年）；IEX 实时与 crypto 走 WebSocket",
      "ts_declarable": true,
      "verdict": "alternative",
      "declared_time_semantics": "自定义时点：max(当日 16:00 ET 收盘, 当日 20:00 ET 修正收敛) ⇒ visible_at = trade_date 20:00 America/New_York；仅取 raw 价 + 当期 splitFactor/divCash",
      "declared_frequency": "日频（EOD）",
      "cost_note": "Starter 免费 50次/时、1000次/天、500 符号/月；Power $30/月个人、$50/月内部商用",
      "license": "Internal Use Only；再分发单独授权",
      "evidence": "tiingo.com/documentation/end-of-day（字段表 + 17:30/20:00 ET 修正窗口说明）+ tiingo.com/about/pricing"
    },
    {
      "id": "E3",
      "name": "Tushare us_daily",
      "channel": "external",
      "markets": "US equities（NYSE/NASDAQ/OTC）",
      "frequency": "仅日线（未复权）",
      "ts_declarable": true,
      "verdict": "alternative",
      "declared_time_semantics": "收盘后可见（trade_date 当日美股收盘后）；落库前以实际可拉取时间校准，若实测次日才可拉取则改声明为「次日可见」",
      "declared_frequency": "日频 1d",
      "cost_note": "120 积分可试用；正式权限需开通；单次 6000 行",
      "license": "Tushare 平台协议/积分体系；本项目已有 token 与 connector",
      "evidence": "tushare.pro/wctapi/documents/254.md（us_daily：未复权、120 积分、单次 6000 行）"
    },
    {
      "id": "X1",
      "name": "Yahoo Finance / yfinance",
      "channel": "external",
      "markets": "Global（非官方端点）",
      "frequency": "日线为主",
      "ts_declarable": false,
      "verdict": "exclude",
      "declared_time_semantics": null,
      "declared_frequency": null,
      "cost_note": "免费但非官方：无 SLA、端点历史上无预警失效、易被限流/IP ban",
      "license": "无官方公开 API 授权，不具备可商用/可依赖条款",
      "evidence": "H1b❌：Adj Close 为全历史重述序列 + 非官方端点无 SLA（第三方对比 2026-06-04：Free, unofficial… No formal SLA）"
    },
    {
      "id": "X2",
      "name": "Tushare us_daily_adj（美股复权行情）",
      "channel": "external",
      "markets": "US equities",
      "frequency": "日线（复权）",
      "ts_declarable": false,
      "verdict": "exclude",
      "declared_time_semantics": null,
      "declared_frequency": null,
      "cost_note": "120 积分可试用（同 us_daily，但字段不可用）",
      "license": "Tushare 平台协议",
      "evidence": "H1b❌：官方原话「复权因子历史数据可能除权等被刷新，请注意动态更新」——供应商明示历史值会被未来除权刷新"
    },
    {
      "id": "X3",
      "name": "Polygon/Massive 聚合 adjusted=true 序列（模式级）",
      "channel": "external",
      "markets": "US equities",
      "frequency": "聚合 bar",
      "ts_declarable": false,
      "verdict": "exclude",
      "declared_time_semantics": null,
      "declared_frequency": null,
      "cost_note": "同 E1（禁用的只是默认参数取值）",
      "license": "同 E1",
      "evidence": "H1b❌：官方文档「By default, results are adjusted（for splits）」——默认即未来拆股回溯改写"
    },
    {
      "id": "X4",
      "name": "Tiingo adj* 复权序列（模式级）",
      "channel": "external",
      "markets": "US equities",
      "frequency": "EOD",
      "ts_declarable": false,
      "verdict": "exclude",
      "declared_time_semantics": null,
      "declared_frequency": null,
      "cost_note": "同 E2（禁用的只是复权字段）",
      "license": "同 E2",
      "evidence": "H1b❌：官方文档「adjustment methodology… (CRSP)… incorporates both split and dividend adjustments」+ 官方明示晚间持续更新当日价"
    }
  ]
}
```

---

## 6. 推荐结论 + 接入成本估算

### 6.1 推荐（分阶段，每阶段独立可交付）

| 阶段 | 选谁 | 干什么 | 为什么不一步到位 |
|---|---|---|---|
| **阶段 0**（D7 立即需要） | **N2 交易所 crypto**（Binance 优先） | 用 NT 官方 adapter 拉历史 klines 进 catalog，跑通「NT 原生策略 + 国外市场 + ResultSpec 回执」端到端 | D7 ③4 明写「样例数据即可，真实数据源等 D9」——**零成本、零商务流程**，不阻塞 D7；代价是 crypto 不是最终目标市场 |
| **阶段 1**（国外股票主链路） | **N1 Databento**：日频回测优先用 **`EQUS.SUMMARY`**（全市场 consolidated EOD OHLCV，正是日频所需粒度）；需要盘口/逐笔再用 `EQUS.MINI` 或 `Nasdaq TotalView-ITCH`；期货走 `GLBX.MDP3` | 先以 NT 原生 adapter 直连 catalog 打通（研究态），再补规范层回灌（正式态） | Databento 是官方 adapter 清单内唯一覆盖国外股票的纯数据通道，时间语义最强，且能用 $125 credits 零现金启动 |
| **备选 A**（若 Databento 商务/成本不落地） | **E2 Tiingo** | 外部 connector 进规范层，日频起步 | $30/月封顶 + raw 价与当期因子齐全，是"最便宜的合规日频"；缺点：无分钟频、免费档 500 符号/月太窄 |
| **备选 B**（若需要分钟/逐笔深度） | **E1 Polygon/Massive** | 外部 connector，**必须 `adjusted=false`** + 自管 PIT 复权 | 深度与历史最长，但复权要自己做（成本在这里，不在 adapter） |
| **备选 C**（最省事但不推荐当主源） | **E3 Tushare `us_daily`** | 复用现有 token 与 connector 加一个接口 | 边际成本最低，但**只有日线、只有未复权**，覆盖与频率受限 |

**明确不推荐**：X1–X4（硬判据不通过，见 §4）；N3 IBKR 仅作为**将来执行通道**候选，不作为回测数据主源（数据可取性受账户订阅与 pacing 约束，且口径需先钉死）。

### 6.2 接入成本估算

【判断】单位为人日（含自测），假设：单人维护、D1 规范层与 NT 翻译层（`export_bars`）已就位（D7/D1 前置）、**不含商务签约与等待时间**、复用既有 connector 集成先例（宏观 P2-1、mootdx）。

| 落地形态 | 组成 | 估算 |
|---|---|---|
| 阶段 0 · crypto 样例（N2） | 依赖不变（pin rc6）+ 拉 klines 进 catalog + 一个 NT 原生策略 + 回执映射 | **0.5–1 人日** |
| 阶段 1a · Databento 直连 NT adapter（研究态） | API key + DBN 拉取 + catalog 写入 + `bars_timestamp_on_close` 口径确认 + `metadata.get_cost` 成本预估脚本 | **1–1.5 人日** |
| 阶段 1b · Databento 经规范层（正式态） | 1a + Go connector（HTTP/DBN 解析）+ DDL 加表（追加 `postgres.go` 数组）+ D2 声明注册 + NT 翻译映射行 + PIT/聚合测试 | **+3–4 人日**（累计 4–5.5） |
| 备选 A · Tiingo（E2） | REST 客户端 + 限流/分页（免费档 50 次/时）+ 规范层落库 + 「20:00 ET 可见」声明 + 测试 | **2–3 人日** |
| 备选 B · Polygon（E1） | 聚合 REST + `adjusted=false` + **自管 PIT 复权（拆股/分红）** + 限流 + 测试 | **3–5 人日**（复权占大头） |
| 备选 C · Tushare `us_daily`（E3） | 复用现有 connector/token + 新接口适配 + 声明 + 测试 | **1–2 人日** |
| 共性（一次性，摊到首个源） | 规范层新表 DDL + 市场→内核映射配置（`market.US → nautilus`）+ **美股交易日历**（半日市/假日）+ 配额守门 | **1–2 人日** |

**license 费的一个实质利好**【事实引官方 FAQ】：`EQUS.MINI` 与 `EQUS.SUMMARY` 属 Databento 自有数据集，**"no license fees on distribution and does not require exchange reporting… instantly approved once you subscribe"**；而 `NYSE Trades` 等交易所直连 feed 的 license 费显著更高。这直接影响选型：**日频回测走 `EQUS.SUMMARY` 可绕开交易所报备与 license 费**，是本项目（单人实验室、无合规人力）的最优解。

**现金成本（首年，最省路径）**【判断】：阶段 0 = **$0**；阶段 1 用 $125 credits 起步 → 美股日线极小数据量下 credits 可支撑较长评估期（数据量级：日线每符号每年约 250 行，量级远低于 GB 级）；进入常规使用再按 Standard $199/月 或按量付费。**美股档位的每 GB 单价须以 Databento 官方 `metadata.list_unit_prices` 端点为准**（本报告只核到 CME 档「from $0.50/GB」，不替供应商报美股单价）。

### 6.3 落地形态的关键约束（红线，别踩）

【判断，依据为冻结契约】NT 原生 adapter 直连 catalog 是**最快路径**，但它**不经过 D1 规范层**。而 D7 ③5 要求「国外线数据经 D1 规范层翻译进 NT」，蓝图 §2.2 亦明写「不允许为了『性能』绕过规范层直连 DB」。因此：

- **允许**：阶段 0/1a 以 NT 原生 adapter 直连 catalog 做**打样与研究**（D7 只要求样例数据跑通）；
- **必须**：进入正式链路时，数据须落规范层（Go 侧 connector）+ 经 NT 翻译层进 catalog，**规范层是唯一真相**；
- 若坚持 adapter 直连作为长期形态，则必须补**规范层回灌**（同一份数据同时进规范层），否则违反红线。**这一点是后续工单的硬约束，不是可选项。**

### 6.4 中国市场语境下的额外提示

【判断】美股收盘（16:00 ET）对应北京时间次日 04:00–05:00（夏令时）。若同步作业跑在北京时间当天，"当日美股收盘数据"**物理上还没产生**。因此 D2 声明里的「收盘后可见」必须写成**交易所时区**（`America/New_York`）的绝对时点，再由系统换算——**不能用北京时间日期做可见性判断**，否则会造出「同一份数据在东八区看起来早了一天」的假前视。

### 6.5 未纳入 / 边界外

- 中国香港市场：通道与 license 另成一档（HKEX 官方收费、券商 OpenAPI 需账户），本报告未做供应商级调研，建议届时单独立单。
- 国外基本面/另类数据：属后续 connector 工单，不在 D9 范围。

---

## 7. 接口需求清单（喂给 D7 与未来国外 connector 工单）

> 编号 I-xx；「判据」列指向本报告的硬判据或冻结契约条目。**本清单即后续工单 ③要求的直接输入。**

| # | 需求 | 判据 / 为什么 | 验收方式（可机检） |
|---|---|---|---|
| **I-01** | connector 注册**必须**声明 `time_semantics`（枚举：按公告日可见 / 收盘后可见 / 次日开盘可见 / 自定义时点，自定义须给出判定表达式）**与** `frequency_spec`（日频/分钟频/事件流 + 聚合口径） | D2 fail-closed；蓝图 §4.2 | 缺任一声明 → 注册失败且报错点名缺哪个字段（单测） |
| **I-02** | 每个数据点必须同时提供 `ts_event`（事件发生）与 `ts_init`（可见/到达），且 **`ts_init >= ts_event`** | H1a/H1c；NT 回测按 `ts_init` 排序喂入 ⇒ 可见性由引擎执法 | connector 侧断言 + 违规样本被拒收的单测 |
| **I-03** | bar 时间戳语义必须显式声明（区间**开**或**闭**），默认**闭**（对齐 NT `bars_timestamp_on_close=true`） | H1a；NT Databento 集成文档 | 声明与数据形态矛盾的样本被拒收（单测） |
| **I-04** | **只消费未复权原值**；复权在规范层按 PIT 自主应用。**禁止**入库任何会被回溯改写的序列 | H1b（X2/X3/X4 的死因） | 单元测试：源侧复权序列入库 → 拒收 |
| **I-05** | 可见时点必须落在**交易所时区**绝对时点上（如 `20:00 America/New_York`），禁止用本地时区日期推断 | §6.4 | 时区换算单测（含夏令时切换样本） |
| **I-06** | **美股交易日历**独立供给（半日市 13:00 ET 收盘、假日、临时休市），与 core 仓既有 `trading_calendar` 同构 | D1 聚合与 NT venue 日历都需要 | 日历缺失日 → 数据不入回测（fail-closed） |
| **I-07** | 市场→内核映射配置扩展：`market.US → nautilus`（并存 `market.CN → go`）；未注册市场 → **显式拒绝，不回退 Go** | D7 ③2 | 未注册映射返回 400/显式错误的单测 |
| **I-08** | 规范层落库：新表或复用现有表由 D1 定；**DDL 只追加进 core 仓 `pkg/storage/postgres.go` 的 `migrationStatements()` 数组末尾**（工程红线，不建 migration 文件） | D1 ④ | 无 migration 文件新增；DDL 在数组内（`grep` 断言） |
| **I-09** | NT 翻译层加映射行（规范层 → `Bar`/`BarType`），Parquet catalog 导出沿用 `export_bars` 口径 | D7 ③5；蓝图 §4.1「加新数据类 = 映射表加一行」 | 映射正确性单测（字段名/时间戳口径/单位） |
| **I-10** | **成本守门**：调用前用 Databento `metadata.get_cost` 预估，超预算即拒（fail-closed），并记录实际花费 | §6.2 现金成本可控 | 超预算请求被拒的单测 |
| **I-11** | 配额/限流适配：免费档（Polygon 5 次/分、Tiingo 50 次/时）必须显式限速与退避，不得靠撞限流 | §3.2 | 限速器单测；无 429 突增 |
| **I-12** | 溯源列 + 幂等重放沿用现有 ETL 惯例（`source`、`ingested_at` 等） | 既有惯例 | 重放两次结果一致（单测） |
| **I-13** | 缺失语义：**缺失≠0**，存在性单独标记 | 蓝图 §5.2 三条铁律 | 缺失样本产出存在性标记而非 0（单测） |
| **I-14** | 价格精度与币种：美股常见定点 1e-9（Databento）→ 规范层统一精度与币种字段 | H1a 之外的数据正确性 | 精度换算单测 |
| **I-15** | 源不可用 → **fail-closed 显式报错**，不静默降级/不回退 | D7 ③2 / ADR-031 fail-closed | 源不可用时 run 显式 failed（单测） |
| **I-16** | **PIT 破坏测试**（护栏）：把可见时点改宽（如「收盘后可见」→「盘中即可见」）→ 测试必须变红 | D2 ⑤破坏验证先例 | 改宽声明 → 红；还原 → 绿（transcript 留档） |

---

## 8. 验收自证（对照 D9 ⑤）+ 破坏验证记录

### 8.1 四要素逐条自证

| D9 ⑤ 要求 | 落点 | 自证 |
|---|---|---|
| ① 对比表（覆盖市场/频率/时间语义可声明性/价格配额/license） | §3.1 / §3.2 / §3.3 | 三组共 10 个候选项，五列齐备，每格带引文（附录 A） |
| ② 硬判据筛查记录 | §2（判据定义）+ §4（逐条筛查表，H1a/H1b/H1c 三子项 + 引文） | 6 通过 / 4 排除，排除项全部倒在同一判据并留证据 |
| ③ 推荐 | §6.1（分阶段推荐表）+ §0 速览 | 主选 N1、阶段 0 用 N2、备选 E1/E2/E3，并明确不推荐项 |
| ③ 接入成本估算 | §6.2（人日表 + 现金成本） | 标注为【判断】并写明假设与不含项 |
| ④ 接口需求清单 | §7（I-01…I-16，含判据与可机检验收） | 每条给出验收方式 |
| 结论可直接抄进后续工单 ②上下文 | §0 + §5 JSON 登记表 | 工单可直接引用 `id` 与 `declared_time_semantics` |

### 8.2 破坏验证（改坏 → 红 → 还原 → 绿）

校验脚本见**附录 B**（stdlib-only，无第三方依赖），断言：

- **A1** 报告含四要素章节（对比表 / 硬判据筛查记录 / 推荐 / 接口需求清单）
- **A2** 候选项 ≥ 3，其中 `nt_native` ≥ 1、`external` ≥ 2
- **A3** 所有**未被排除**的项（`verdict != exclude`，含 recommend / alternative / conditional），`ts_declarable` 必须为 `true`
- **A4** 所有 `ts_declarable == false` 的项，`verdict` 必须为 `exclude`（硬判据执法）
- **A5** `recommend_primary` 恰好一个
- **A6** 接口需求清单条目 ≥ 10（扫描 `I-xx` 编号）
- **A7** 每个候选项都有非空 `evidence`

三组破坏实验与结果（完整 transcript 见**附录 C**）：

| # | 改坏动作 | 预期 | 实测 | 还原后 |
|---|---|---|---|---|
| B1 | **重新启用被硬判据排除的源**：X1（yfinance）`verdict: exclude → alternative` | A3 + A4 红 | **红**（exit 1；A3、A4 两条均点名 X1） | 绿，sha256 与基线一致 |
| B2 | 删除「## 7. 接口需求清单」章节标题 | A1 红 | **红**（exit 1，点名缺章节） | 绿，sha256 与基线一致 |
| B3 | **让重述序列当主选**：X2（`us_daily_adj`）`verdict: exclude → recommend_primary` | A3 + A4 + A5 红 | **红**（exit 1；三条断言点名 X2） | 绿，sha256 与基线一致 |

**结论**：硬判据不是只写在文档里——A3/A4 两条断言使「被排除的源被重新启用」与「重述序列被推荐」这两种漂移**当场变红**。

---

## 附录 A · 证据来源清单（采集日期 2026-10-14）

| # | 来源 | 类型 | 用于 |
|---|---|---|---|
| E-1 | `nautilus_trader` 本地检出 `git show v2.0.0rc6:ADAPTERS.md`（Official adapters 表；与 rc7 工作区一致） | 离线冻结版本 | N1/N2/N3 官方 adapter 身份；E1/E2/E3 无官方 adapter |
| E-2 | `git show v2.0.0rc6:docs/concepts/data/index.md`（§Timestamps / Environment-specific behavior） | 离线冻结版本 | `ts_event`/`ts_init` 语义；「Data is ordered by `ts_init` using a stable sort」 |
| E-3 | `docs/integrations/databento.md`（本地 rc7 工作区）+ nautilus 在线集成文档 | 离线 + 在线 | schema→NT 类型映射、bar 时间戳开闭、Databento 四时间戳与 NT 映射规则、$125 credits |
| E-4 | `docs/integrations/binance.md`（本地工作区 §Bars and historical market data） | 离线 | crypto 历史 bar 可拉、1s klines |
| E-5 | databento.com/pricing | 官网 | 四档价格、$125 credits、license 与再分发条款、按 GB 计费口径 |
| E-6 | databento.com/docs/standards-and-conventions/common-fields-enums-types | 官网 | 四时间戳定义、`ts_recv` 单调性与 PTP/GPS 同步 |
| E-7 | polygon.io 聚合接口文档（`adjusted` 参数、`t` 字段）+ polygon.io 定价页 | 官网 | E1 频率/时间语义/价格/`Individual Use`；X3 排除依据 |
| E-8 | tiingo.com/documentation/end-of-day + tiingo.com/about/pricing | 官网 | E2 字段（raw 与 adj + `splitFactor`/`divCash`）、17:30/20:00 ET 修正窗口、价格与 `Internal Use Only`；X4 排除依据 |
| E-9 | tushare.pro/wctapi/documents/254.md（`us_daily`）+ tushare.pro/document/2?doc_id=338（`us_daily_adj`） | 官网 | E3 通过依据（未复权/120 积分/单次 6000 行）；X2 排除依据（复权因子会被刷新） |
| E-10 | 第三方对比页（tradeodds.io，2026-06-04；geekflare；quantvps，2026-10-10） | 旁证 | X1 无 SLA 与 unofficial；Databento 价格交叉核对（**与官方冲突处以 E-5 为准**） |
| E-11 | core 仓 `nt-sidecar/requirements.txt`（`nautilus_trader==2.0.0rc6`） | 本仓冻结文件 | 版本 pin，决定本报告按 rc6 取证 |
| E-12 | databento.com/equities（美股数据集页，含 FAQ） | 官网 | N1 美股覆盖（20,000+ 股票与 ETF、50+ 场所、3M+ 符号）、`EQUS.MINI`/`EQUS.SUMMARY` 定位与「无交易所 license 费、无需报备」 |

**已知冲突（如实登记）**：Databento Standard 档月费，第三方页面出现 $179 与 $199 两个数字；**以官网 $199/月 为准**（E-5）。美股（EQUS）数据集的每 GB 单价与档位未在本次取证中核到具体数字，仅核到 CME 档「from $0.50/GB」⇒ 列为后续工单核实项（I-10 的成本守门会强制先查 `metadata.list_unit_prices`）。

---

## 附录 B · 可复现校验脚本（stdlib-only，复制即可跑）

```python
#!/usr/bin/env python3
# D9 报告自检：解析报告内首个 ```json 登记表 + 扫描章节，执行硬判据断言。
# 用法: python d9_check.py <report.md>   （exit 0 = 全绿, exit 1 = 红）
import json, re, sys, pathlib

REQUIRED_SECTIONS = [
    "对比表", "硬判据筛查记录", "接口需求清单", "推荐",
]

def load_register(text):
    m = re.search(r"```json\n(.*?)\n```", text, re.S)
    if not m:
        raise SystemExit("FAIL: 报告内未找到 ```json 候选登记表")
    return json.loads(m.group(1))

def main(path):
    text = pathlib.Path(path).read_text(encoding="utf-8")
    reg = load_register(text)
    cands = reg.get("candidates", [])
    errs = []

    # A1 四要素章节
    for s in REQUIRED_SECTIONS:
        if not re.search(r"^#{1,4}.*" + re.escape(s), text, re.M):
            errs.append(f"A1: 缺少章节 -> {s}")

    # A2 候选构成
    nt = [c for c in cands if c.get("channel") == "nt_native"]
    ext = [c for c in cands if c.get("channel") == "external"]
    if len(cands) < 3:
        errs.append(f"A2: 候选项 < 3 (实际 {len(cands)})")
    if len(nt) < 1:
        errs.append("A2: nt_native 通道 < 1")
    if len(ext) < 2:
        errs.append(f"A2: external 候选 < 2 (实际 {len(ext)})")

    # A3 未被排除者必须时间语义可声明（覆盖 recommend/alternative/conditional 全部）
    for c in cands:
        if c.get("verdict") != "exclude" and c.get("ts_declarable") is not True:
            errs.append(f"A3: {c.get('id')} verdict={c.get('verdict')} 但 ts_declarable != true（未被排除者必须可声明）")

    # A4 硬判据：声明不了 => 必须排除
    for c in cands:
        if c.get("ts_declarable") is False and c.get("verdict") != "exclude":
            errs.append(f"A4: {c.get('id')} ts_declarable=false 但 verdict={c.get('verdict')}（应为 exclude）")

    # A5 主选唯一
    prim = [c for c in cands if c.get("verdict") == "recommend_primary"]
    if len(prim) != 1:
        errs.append(f"A5: recommend_primary 数量 = {len(prim)}（应为 1）")

    # A6 接口需求清单条目数
    items = set(re.findall(r"\bI-\d{2}\b", text))
    if len(items) < 10:
        errs.append(f"A6: 接口需求条目 < 10 (实际 {len(items)})")

    # A7 每个候选必须有证据
    for c in cands:
        if not str(c.get("evidence", "")).strip():
            errs.append(f"A7: {c.get('id')} 缺少 evidence")

    if errs:
        print("RED — 断言失败：")
        for e in errs:
            print("  - " + e)
        return 1
    print(f"GREEN — 断言全通过（候选项 {len(cands)}：nt_native {len(nt)} / external {len(ext)}；接口需求 {len(items)} 条）")
    return 0

if __name__ == "__main__":
    sys.exit(main(sys.argv[1] if len(sys.argv) > 1 else "d9-foreign-market-data-source-selection.md"))
```

---

## 附录 C · 破坏验证 transcript（实际执行输出）

**执行环境**：Windows / Python 3.13.14（stdlib-only，无第三方依赖）；校验脚本由**附录 B 的代码块原样提取**执行（非另写一份，保证「文档里那份」就是「跑过的那份」）。

```
baseline sha256 = 7cc34da177aa20d3bc0a3af34137da3ee8120e70d11342181dd54b83ea7b1d9b
[BASELINE] python d9_check.py <report>
  exit=0
  GREEN — 断言全通过（候选项 10：nt_native 3 / external 7；接口需求 16 条）

[B1] 改坏：重新启用被硬判据排除的源：X1(yfinance).verdict exclude -> alternative
  exit=1
  RED — 断言失败：
    - A3: X1 verdict=alternative 但 ts_declarable != true（未被排除者必须可声明）
    - A4: X1 ts_declarable=false 但 verdict=alternative（应为 exclude）
  => RED(预期)
  还原：exit=0  sha256一致=True
  GREEN — 断言全通过（候选项 10：nt_native 3 / external 7；接口需求 16 条）
  => GREEN + 完全还原

[B2] 改坏：删除章节标题：'## 7. 接口需求清单'
  exit=1
  RED — 断言失败：
    - A1: 缺少章节 -> 接口需求清单
  => RED(预期)
  还原：exit=0  sha256一致=True
  GREEN — 断言全通过（候选项 10：nt_native 3 / external 7；接口需求 16 条）
  => GREEN + 完全还原

[B3] 改坏：让重述序列当主选：X2(us_daily_adj).verdict exclude -> recommend_primary
  exit=1
  RED — 断言失败：
    - A3: X2 verdict=recommend_primary 但 ts_declarable != true（未被排除者必须可声明）
    - A4: X2 ts_declarable=false 但 verdict=recommend_primary（应为 exclude）
    - A5: recommend_primary 数量 = 2（应为 1）
  => RED(预期)
  还原：exit=0  sha256一致=True
  GREEN — 断言全通过（候选项 10：nt_native 3 / external 7；接口需求 16 条）
  => GREEN + 完全还原
```

**⚠️ 过程记录（如实）**：B1 的第一版设计是「把 X1 的 `ts_declarable` 由 `false` 改成 `true`」，**实测 exit=0 未变红**——因为当时的 A3 只覆盖 `verdict` 以 `recommend` 开头的项，`alternative`/`conditional` 是覆盖缺口（即**假护栏**）。已把 A3 加固为「`verdict != exclude` 者一律必须 `ts_declarable == true`」，并把 B1 改造成更贴题意的动作「重新启用被排除的源」。加固后三组全部按预期变红。**这条记录本身就是护栏有效性的证据：护栏被真正打过，而不是写完就宣称有效。**

**最终验收（附录 C transcript 写入后，对最终文件复跑）**：

```
$ python d9_check.py docs/design/d9-foreign-market-data-source-selection.md
GREEN — 断言全通过（候选项 10：nt_native 3 / external 7；接口需求 16 条）
exit=0
```

> **关于 sha256 的说明（可验证）**：上面的 `baseline sha256` 是**执行验证那一刻**的报告快照（正文与 §5 登记表已定稿）。把 transcript 文本写进附录 C 会改变文件字节，因此最终文件的 sha256 与它不同——**这不构成问题，因为 A1–A7 七条断言只消费三样东西：§5 的 JSON 登记表、章节标题、`I-xx` 编号；附录 C 的文字不参与任何一条断言**。**已实测**：把附录 C 整段删除后对残余文件复跑，仍为 `exit=0 / GREEN — 断言全通过（候选项 10：nt_native 3 / external 7；接口需求 16 条）`。

---

## 维护本文件

- 本报告为 **调研产物**（D9 ⑥：只调研不实施），**不改动**任何冻结契约；如后续选型变更（换源、换档位、价格变动），请**新开调研单**或在本文件追加「修订记录」，不要静默改写 §5 登记表（A3/A4 断言会拦住其中两类漂移）。
- 报告中价格/配额为 2026-10-14 快照；后续工单开工前须重核（尤其 Databento 美股档单价与 Polygon/Massive 更名后的条款）。
- 与代码事实冲突时以代码为准并回写本文件（沿用蓝图「维护本文件」节的同款纪律）。
