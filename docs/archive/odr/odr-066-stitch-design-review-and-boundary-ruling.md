# ODR-066: Stitch 设计稿评审与产品边界裁决

> **Status**: Accepted（裁决已定；标注稿降级与设计系统合一的实施仍在进行中，见 §Decision 第 3–5 条）
> **Date**: 2026-10-13
> **Category**: Audit
> **Related ADRs**: ADR-023（三层模型 / 可监督性）、ADR-024（执行载体是表达式，不是 LLM 生成的代码）
> **Supersedes**: n/a（不取代任何既有决策；本 ODR 是**重申**既有边界并对一份外部 artifact 做处置裁决）

---

## Context

- **触发**: 用户在 `quant-trading-ui` 仓放入一套 12 屏由 Stitch 生成的 UI 重构设计稿（`docs/design/stitch_quantlab_ux_redesign/`，含 12 × `screen.png` + 12 × `code.html` + 1 份设计系统 `obsidian_precision_quant/DESIGN.md`），要求核查其与既定功能设想的吻合度。
- **评审基线**: 本 ODR 的"设想"一侧**只认一份文档** —— [`docs/design/user-flows.md`](../../../quant-trading-ui/docs/design/user-flows.md) v1.0（2026-10-13，UI 重构的 UX 输入基线，含 §1 UX 硬约束、§3 目标 IA、§4 F1–F8 主线 flow、§5 F9–F18 仪器面、§6 横切模式、附录 B 83-path API 地图、附录 C G1–G8 后端缺口）。
- **取证纪律**: 每条结论必须落在 (a) 设计稿可指认的元素，且 (b) 与 `user-flows.md` 条款或**源码实测**冲突。源码侧实测手段：`grep` 跨 `quant-trading` / `quant-trading-agent` / `quant-trading-ui` 三仓 + `docs/openapi.yaml`。未做运行时取证（本机无 Go 工具链，且本次结论不依赖运行时行为）。

## Decision

实验室主任（用户）于 2026-10-13 对三项议题作出裁决，逐条记录如下。

### 1. 产品边界：设计稿越界，判「删」，不改 PRODUCT / VISION

设计稿在**四处**越过了 §1.5「明确不做」与 F16「实盘交易不做」的边界。裁决：**边界不动，设计稿越界元素一律视为待删**，不借重构扩产品边界。

> **裁决细化（同日追加，主任补充口径 —— 覆盖初版表述）**：判「删」的**依据不是"这些能力永远不做"，而是"前端不干预后端设计"**。已存在的契约文档（`PRODUCT.md` / `VISION.md` / `openapi.yaml` / 本条所引 `user-flows.md`）是前端唯一可假设的能力边界；设计稿**无权倒逼**后端新增能力。若将来确实要做盘中行情、L2 或实盘执行，那是一次**后端整体决策**（新 ADR + 数据层 + 服务 + 测试一起改），而不是"因为设计稿画了所以接上"。
> 同一口径下，主任亦声明自己此前对议题 ① 的回答**可能不准确**，以本细化为准；初版"边界不动"的结论**不变**，变的是它的理由与可推翻路径。

| # | 越界内容 | 出现位置 | 冲突条款 | 源码实测依据 |
|---|---|---|---|---|
| B1 | 实盘高频执行：`Q-ROUTER ACTIVE` / TWAP-VWAP 算法母单 / `FIX 4.4 CONNECTED` / 上交所极速托管 Colocation / 券商直连 DMA / 内部交叉盘撮合 / `100 FPS L2 TICK` / 亚毫秒路由 | execution 整屏 | F16「实盘交易不做（MockTrader 足够；XTP 是桩）。**UI 文案不得暗示实盘**」 | AGENTS.md §14：`LiveEngine` 生产零调用方，"接线到实时行情/真实券商之前先补集成测试，不要直接上资金" |
| B2 | 盘中与分钟级：`1D/60m/15m/5m` 周期切换、`2026-04-17 14:00` OHLC、`ENTRY: 14:15`、`REAL-TIME RADAR`、`60-MIN CANDLESTICK`、尾盘大单/主力净流入 | backtest、screener | §1.5「**日频是高频边界**，不做盘中监控」；原则 3（`ENTRY` 是买卖点=决策性措辞） | `grep -rn "1min\|5min\|level2\|逐笔\|tick" pkg/data pkg/domain` 除 ticker 词根外**零命中** —— 数据层无任何分钟/tick 结构 |
| B3 | L2 行情链路：顶栏常驻 `SSE · L2 FEED`、`上交所 Level2 逐笔 3秒快照 / 94,120 msg/s`、`A-Share L2 Synchronized 14:59:58` | 全站顶栏、data_sync、alerts | 同 B2；且见 §2 关于 SSE 的事实更正 | 同上 |
| B4 | 集群/算力布景：`4x RTX4090` / `A100 Tensor` / `EPYC 9654` / 上海机房 `A-102` / `PORT 9840` / `0.42ms` / `SLA 99.999%` / `84.2 TB 湖仓` | 各屏页脚与 data_sync | 单人自托管前提（AGENTS.md §1：多租户/高可用是纯负担）；这些数字会**误导主任对成本与容量的判断** | 栈为 Go + PostgreSQL + Redis + tushare/akshare 日频摄取 |

### 2. 评审自我更正：SSE 结论过强（记录在案，供后续评审引用）

初版评审曾断言「后端无 SSE，设计稿的 SSE 是幻觉」。**该断言经源码复核后判定为过强，已推翻并收窄**：

- ✅ **SSE 确实存在**：`cmd/data/sync_handlers.go:745` 置 `Content-Type: text/event-stream`；`web/src/stores/sync.ts:162` 真实构造 `new EventSource('/api/sync/jobs/{id}/progress')`；`cmd/analysis/handlers_proxy.go:42` 与 `handlers_ai_proxy.go:53` 以 `FlushInterval = -1` 做 SSE 逐写直通。
- ❌ **不存在的部分**：探索 run 无推送通道 —— `quant-trading-agent/cmd/ai/handlers_explore.go:163-165` 只有 `POST /runs`、`GET /runs/:runID`、`POST /runs/:runID/stop`；agent 仓 `grep event-stream` 零命中。任何**行情侧**推送也不存在。
- **收窄后的结论**：设计稿的问题不是「SSE 这个技术不存在」，而是**把 SSE 说成行情推送通道并全站常驻**（等于宣称一条不存在的行情链路）。
- **连带修文档**：`user-flows.md` §6.7「当前后端**无** WebSocket/SSE，全部轮询」对 explore 成立、对 sync 作业**不成立** → 本次一并更正。

### 3. 语义契约违规：判「按契约改图」，不是改契约

F5/原则 3/原则 6 是**不得改动的语义**（F5 原文）。设计稿的违规点：

| # | 违规 | 位置 | 契约条款 |
|---|---|---|---|
| S1 | 二值裁决语言：`因果链校验通过` / `VALIDATED` / `VERIFIED 2024-Q2` | review_desk、explore、factors_ic | F5「输出 = 概率估计 + 质疑清单，**绝无通过/不通过二值**」 |
| S2 | 六维画成五维：缺「冗余/正交」维 | review_desk（explore 与 factors_ic 是完整 6 维） | F5 六维固定：统计/经济/稳健/偏差/冗余 + 因果 |
| S3 | `unassessed` 全站仅出现 1 次，其余屏维度条恒为满值 | 除 review_desk 外 11 屏 | F5「`unassessed` **必须显示为未评估** —— 没查不等于没问题」+ 该条的破坏验证验收 |
| S4 | 决策性/前瞻性措辞：`建议进入实盘模拟沙盒`、`组合构建方案已就绪 期望夏普 2.68` | explore、compare | 原则 3「禁止推荐/建议买入等决策性措辞」 |
| S5 | 摘要卡三要素错位：「路径缩略图」被累计超额收益曲线顶替（收益曲线 ≠ 探索路径）；质疑清单的 blocking/warning/note 分级缺失 | review_desk | F2 卡片必备三要素 |
| S6 | 采纳动作缺「理由输入 + 概率快照」（否决有"记录原因"，采纳没有） | review_desk | F7「采纳**必须**填写理由 + 记录概率估计快照」 |

### 4. 幻觉能力清单：判「标注为设想/删除」，实现时不得当作既有契约

以下元素在后端**无对应端点或无对应技术栈**，设计稿却以"已可用"形态呈现。逐条点名，避免后续实现 agent 误接：

- **数据栈错位**：Polars / Parquet / DuckDB / Arrow IPC / Iceberg / OSS 冷归档（factors_ic、data_sync、backtest 的 `耗时 1.42s (Polars GPU)`）—— 实际栈为 Go + PostgreSQL + Redis。
- **策略语言错位（违反 ADR-024，最严重的一项）**：strategies 屏生成 **Python** `class LateSessionVolumeDivergence(AlphaStrategy): def compute_signals(self, context, market_data)` + `market_data.resample('5min')` + `导入 Python/Q-AST` + `PyPy 7.3 JIT / C++ AST Core` + `.qpkg`。执行载体是**表达式**，可审阅 artifact 是 **Go**，且 canonical `Strategy` 接口签名是 `GenerateSignals(ctx, bars, portfolio) ([]Signal, error)` —— 三处同时不符。
- **审计链布景**：`Merkle Root` 冷密钥签名 / `不可篡改审计链` / `Git Snapshot 对齐 #8f2da` / `导出投委会采纳报告 (PDF)`（journal）—— 无此能力。
- **量化能力**：Risk Parity / MV 权重优化、VaR(99% 1D)、HMM-GARCH 状态概率、VAR(3)/Granger/ADF/Johansen 检验、Barra CNE6 风格基准 —— 83-path 地图中**无一个**对应端点。
- **导出格式**：`导出 (xlsx/.pdf)` —— `openapi.yaml:285` 明写 "currently only `html`"。
- **动作类**：`一键风险隔离 (Isolate)`、`导出风控审计单`、`一键采纳修正` —— 无端点。
- **G1–G6 缺口按可用态画**：摘要聚合、指方向、追问、标记、实验日志深查、采纳记录 —— UI 未给 disabled/placeholder 态，会让实现顺序卡在 agent 仓。

### 5. 覆盖缺口（设想有、图里没有）—— 列为后续设计任务

| 缺口 | 条款 |
|---|---|
| F17 合规（9 端点）无屏；F18 系统管理（auth/admin/users/audit/bootstrap）无屏；`/api/plugins*` 无承载 | §3.2「系统 System」 |
| F9 扩展 walkforward / batch（5 端点）无承载 | 附录 B |
| F3 四干预未做到"三处上下文入口一致"：叫停(explore/review_desk) / 指方向+追问(仅 strategies) / 标记(仅 explore 1 处) | F3 验收 |
| F6 证据质问仅 4/12 屏出现，而**恰好缺回测报告、因子 IC、对比页** —— 那正是最该被质问的数字 | F6 适用范围 |
| F15「策略健康度周报」与「失效 → 回到 F1 候选生成」的漏斗回流动作未画 | F15 |

### 6. 设计系统合一：以设计稿为标准，改 `DESIGN.md` 与 `visual.md`

主任裁决：**以设计稿为准**修订两份规范文档。落地前必须先解决一个**设计稿内部**的矛盾 —— 12 份 `code.html` 各带一套内联 Tailwind 主题，实测**没有两份相同**：

| 维度 | `DESIGN.md` 规定 | 12 份 HTML 实测 |
|---|---|---|
| background | `#10131a` | `#090a0c`（8 屏）/ `#090d16`（review_desk）/ `#08090b`（compare）/ 未指定（2 屏） |
| 结构字体 | Geist | Inter（6 屏）/ Geist（1）/ Plus Jakarta Sans（1）/ 未指定（3） |
| primary | `#b4c5ff` | 仅 review_desk 为 `#2563eb`；strategies 的 `primary` 竟是 `#cbd2db`（灰） |

即"以设计稿为标准"存在**多个互不相同的候选基准**。主任对融合口径的细化（同日追加）：

1. **着重参考新设计，但不把它当完成稿** —— 设计稿给出的是**大概的功能与 UX 方向**；凡涉及细节（字段、指标、控件能拿到什么数据），一律**以实际后端能力为准**。这与第 1 条同源：设计稿是方向的输入，不是契约的来源。
2. **顺序：先更新 `DESIGN.md`，再回头对齐 `visual.md`** —— `DESIGN.md` 是这套新设计的名义规范，先把它内部收敛成一份自洽的真相源（含解决上表的 token 分歧），`visual.md` 再据其修订。避免两份文档同时改导致第三套漂移。
3. 由 ① 可推出**收敛规则**：token 分歧由评审方（AI）按"多数屏实测值 + 与新设计基调一致"自行定夺并**逐条写明取舍理由**，不必回问；但凡某 token 隐含了后端给不出的东西（如行情推送带来的高频刷新色、tick 流的等宽抖动），该 token 随功能一并删。

另两项连带决策：

- **导航标签漂移**：backtest 屏把标签改成「回测分析/探索观测/因子工厂/全市场选股/数据通道」；review_desk 侧栏**缺**「数据管道 Data Sync」。
- **实现栈**：稿子是 Tailwind CDN + Material Symbols；现仓是 Vue 3 + Naive UI + `@vicons/ionicons5` + Chart.js，**未装 Tailwind**。引入 Tailwind 属 AGENTS.md「Ask First」项，需单独批准。
- **可访问性底线（不因"以设计稿为准"而放松）**：`::-webkit-scrollbar{display:none}` 隐藏滚动条；`label-caps` 10px/0.08em 在 `outline-variant #2a3449` 暗底上对比度存疑（原则：正文 ≥ 4.5:1）。

## Consequences

**正面**
- 产品边界（日频 / 不做实盘 / 能力层守数据）在文档层被**重申并加固**，且留下了可 grep 的越界清单 —— 后续任何 artifact 再引入 L2/分钟线/实盘措辞，可直接引用 B1–B4 否决。
- 主线（F1–F7）的设计质量得到确认：IA 三分组、给方向表单含"别碰什么"负向约束、F4 四阶段回放与"共剪枝 18 个无价值分支"、`probability = 已评估维度最小值` + 最弱项高亮、Sharpe 与 P(>2) 并置、K9 三值 `INVALID` 显式 + 对比页 `GATE EXCLUDED` 不静默丢弃 —— 这些**与设想同构，应作为施工基准**。
- 评审自身的一条过强结论被源码复核推翻并记录在案，同时暴露出 `user-flows.md` §6.7 的事实性过时 —— 错误进了文档而不是进了代码。

**负面 / 代价**
- 12 屏中 execution 与 data_sync 两屏**可用面积最低**（几乎整屏需重画），screener 与 backtest 需切除盘中/L2 腿。设计稿的"完成度"因此显著低于其观感给出的预期。
- "以设计稿为标准"合一前，仓库里同时存在**三套设计真相源**（`DESIGN.md` / 12 份 HTML / `visual.md`），任何提前施工都会固化错误。**实施时核出后两套，实为五套**：`web/src/styles/variables.css`（26 个 `--q-*`，被消费约 100 次 —— 真正上线的那一份）与 **Naive UI 内置 `darkTheme`**（`App.vue` 从未传 `themeOverrides`，故所有 N 组件独立走 Naive 调色板）。只改前两者页面不会有任何变化。
- 设计稿把 G1–G6 画成可用态，实现顺序与后端排期之间出现一段必须显式管理的落差。

## Artifacts

| 文件 | 动作 |
|---|---|
| `docs/archive/odr/odr-066-stitch-design-review-and-boundary-ruling.md` | 新建（本文件） |
| `docs/ADR.md` | ODR Index 追加 ODR-066 行 + 尾注记账 |
| `quant-trading-ui/docs/design/user-flows.md` | v1.0 → v1.1：§6.7 SSE 事实更正；§1.5 追加边界重申与本 ODR 引用；顶部依据区标注设计稿定位（参考件 ≠ 规格）并指向标注稿 |
| `quant-trading-ui/docs/design/stitch_quantlab_ux_redesign/ANNOTATIONS.md` | **新建（第 2 步产出）** —— 12 屏逐控件 → API 端点处置清单（留/改/删/待后端/装饰），含横切 6 项、缺屏 5 项、施工优先级 12 序。端点实测自 `openapi.yaml`（76 path）+ `cmd/data/main.go` `/api/sync/*`（14 条，契约外）+ `quant-trading-agent/cmd/ai/handlers_*.go` |
| `quant-trading-ui/docs/design/index.md` | 目录追加标注稿入口（挂在 user-flows 之下） |
| `quant-trading-ui/docs/design/stitch_quantlab_ux_redesign/obsidian_precision_quant/DESIGN.md` | **v1.0 → v2.0，升格为唯一真相源**（第 3 步）：五处收敛 + §0 逐条取舍记录 + §1.3/§3 确定性↔概率性双通道 + §4 状态三值 + §4.1 涨跌色偏离登记 + §7 全量对比度实测 + §8 边界负面清单 |
| `quant-trading-ui/docs/design/visual.md` | **降级为人读视图**并指向 DESIGN.md；§1 色彩/§2 字体/§4 圆角阴影/§6 变量全表按 v2.0 重定值；新增双通道、三值、数字严格性；**废除抬升式 hover**（与"不用戏剧性投影"冲突） |
| `quant-trading-ui/web/src/styles/variables.css` | 运行时实现：26 个既有 `--q-*` **全部保名改值**（约 100 处消费零破坏）+ 新增 8 个语义 token（`--q-ai*` / `--q-danger-text` / `--q-focus`）+ `prefers-reduced-motion` 降级 |
| `quant-trading-ui/web/src/App.vue` | **新增 `themeOverrides`**（此前只有 `darkTheme` 无覆写）—— 让 Naive 组件与 `--q-*` 同源；`body` 背景由硬编码 `#0d1117` 改 `var(--q-bg)` |
| `quant-trading-ui/docs/design/stitch_quantlab_ux_redesign/{screen.png, code.html}` | **未改动**（12 屏原稿保留作方向参考，处置由 ANNOTATIONS.md 承担） |

| `quant-trading-ui/web/src/styles/palette.ts` | **新建（第 4 步 B 项）** —— 代码侧唯一字面量源。Chart.js 与 Naive `themeOverrides` 吃不了 CSS 变量，此前字面量散在组件里，造成「改了 token、图表还是旧色」 |
| `quant-trading-ui/web/src/styles/palette.test.ts` | **新建：漂移守卫** —— ① `palette.ts` ↔ `variables.css` 18 对逐字相等 ② 废弃旧值不得回流 ③ WCAG 对比度不变量（文字三级/状态色/填充白字/边框非文本）。**破坏验证已做**：改坏 `--q-text3` → 2 条断言红；改坏 `palette.text3` → 相等 + 对比度各 1 条红；还原后 25/25 绿，探针无残留 |
| `quant-trading-ui/web/src/components/backtest/EquityChart.vue`、`web/src/composables/useBacktestChart.ts` | 图表色改取 `chartPalette`（21+/20-，纯替换无逻辑改动），旧调色板**零残留** |
| `quant-trading-ui/docs/design/stitch_quantlab_ux_redesign/obsidian_precision_quant/DESIGN.md` | 追加 §0.2 图表系列色规则（参照序列用中性灰而非 amber，因 amber 是 `invalid` 专用语义色）+ `derived-by` 补 palette.ts 与守卫测试 |

**实机验证（第 4 步 A/B）**：起后端（PG/Redis 宿主机已在跑，compose 起 analysis/data/strategy 三服务，`/health` healthy）→ 浏览器读**计算后样式**：`n-card` 底 `rgb(17,22,32)`=`#111620`、primary 钮底 `rgb(37,99,235)`=`#2563eb`、输入框 `--n-color: #111620` ⇒ `themeOverrides` 确实穿透到组件层。**并因此撞出两个只在真页面上才看得见的新缺陷**：① Naive 把 primary/info 实心钮文字派生成 `#000`，压 `#2563eb` 只有 **4.06:1** → 已按 Button peer 覆写为白字（实测 `--n-text-color: #ffffff`）；② 输入框底走 `common.inputColor` 而非 `color`，未覆写时留在 Naive 自带 `rgba(255,255,255,0.1)` → 已补。**图表像素级取证**：canvas 采样得 `180,197,255`(新 `#b4c5ff`) 2430 px、`88,166,255`(旧 `#58a6ff`) **0 px**。

**门禁（累计）**：`npm run typecheck` 0 错 / `npm test` **247 passed (21 files)** / `npm run build` 成功 / 新增与改动的非存量文件 `eslint` **0 warning**（全仓 745 warning 为存量，0 error）。

## Metrics

- 评审覆盖：**12 / 12 屏** + 1 份 `DESIGN.md` + 12 份 `code.html`（含 token 逐屏比对）。
- 结论分布：吻合项 **9 类**（主线密度、四干预部分、回放、裁决契约、K9 三值、概率并置、拍板台账、提案视觉分域、ADR-024 编译不执行）；硬冲突 **8 条**（B1–B4 边界 + S1–S6 语义，去重后 8 项）；幻觉能力 **7 组**；覆盖缺口 **5 类**；设计系统不一致 **4 项**（token / 字体 / 导航 / 栈）。
- 自我更正 **1 条**（SSE 结论过强 → 收窄并修文档）。
- 产品边界变更：**0**（PRODUCT / VISION 未动）。

**遗留（第 3 步明确未收，待第 4 步）**

| # | 项 | 判据 |
|---|---|---|
| 1 | **67 处硬编码 hex 绕过 token**，且多为**旧调色板**（`#58a6ff`×12 / `#f85149`×6 / `#3fb950`×6 / `#8b949e`×5 / `#21262d`×4 / `#484f58`×3），集中在 `EquityChart.vue`(18) 与 `useBacktestChart.ts`(8) | Chart.js 不能直接吃 CSS 变量，需一个 `styles/chart-palette.ts` 从 DESIGN.md 同源导出字面量。**属实现层收口，不在 token 合一里做**（否则一次改两个变量，无法判断红点来源） |
| 2 | **Inter / Geist / JetBrains Mono 全部未加载** —— `index.html` 无 `<link>`、`public/` 无 woff2，实际仍渲染系统字体 | 自托管 woff2 涉二进制（AGENTS 禁区，需批准）；外部 CDN 与"自托管"定位冲突。**已在 DESIGN.md §5.1 与 variables.css 注释双向登记**，等主任裁决 |

## Lessons Learned

1. **"看起来很专业"是评审的对立面**。这套稿子的密度、配色、术语都在传递"已经想清楚了"，而它同时在 Python 上跑策略、在 L2 上做选股、在 Merkle 树上做审计 —— 观感与契约相反。凡是传递能力存在的视觉信号，都必须回源码点名核对。
2. **子代理/外部 artifact 的每一条"能力断言"都要落到可 grep 的判据**。本次三条结论（Polars、无 SSE、xlsx 导出）里有一条被自己复核推翻，正是因为它听起来与文档一致（`user-flows.md` §6.7 确实写了"无 SSE"）。**文档一致 ≠ 事实一致**。
3. **多屏生成工具会静默地产出"多套设计系统"**。12 份 HTML 的 token 两两不同，只有逐屏提取比对才看得见；`DESIGN.md` 与 HTML 谁都不服从谁。今后接外部设计 artifact，第一步就应该是 token 一致性比对，而不是看渲染图。
4. **越界内容总是与最高质量的部分同屏出现**。execution 屏是全套里最精致的一屏，也是唯一整屏违规的一屏 —— 评审若按"精致度"排序注意力，会正好把风险最高的部分放到最后看。
