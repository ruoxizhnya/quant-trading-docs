# ADR-031: 内核可插拔 + 外围服务 NT-ready —— buy vs build 路线（NautilusTrader 侧车，整体替换否决）

> **Status**: **Accepted**（2026-10-12 若曦拍板固化）——阶段 1（契约化）与阶段 2（NT 侧车 PoC）立即执行；**决策门已于 2026-10-14 裁决（若曦拍板）：① 推进生产化**，阶段 3 随 K16 落地，详见 D4 末尾「决策门裁决记录」。**决议变更（2026-10-14，若曦拍板）**：内核定位扩展为**双完整核心热转换（路线二）**，见文末「决议变更记录」，附件 = [design/data-factor-unification.md](../design/data-factor-unification.md)。
> **Date**: 2026-10-12
> **Category**: Architecture
> **Related**: [ADR-027](adr-027-modular-decomposition-to-independent-services.md) · [ADR-028](adr-028-tiered-extension-of-strategy-expression-capability.md) · [ADR-029](adr-029-ai-layer-2026-agent-practice-alignment.md) · [ADR-030](adr-030-draft-observed-improvements.md) · core 仓 `docs/design/kernel/nautilus-trader-kernel-evaluation.md`（NT 深度调研，2026-10-12）· core 仓 `docs/design/kernel/kernel-pluggable-nt-ready.md`（需求与设计）· core 仓 `docs/design/kernel/target-architecture-modular-kernel.md`（模块化内核蓝图，D1–D5）
> **Upstream**: 若曦 2026-10-12 ——
> ①「维护一个内核的成本很大，想尽量用一个开源内核，集中精力做其他上层的开发」（buy vs build 动机）
> ②「如果不建议取代，至少在外围的服务上争取可以连接 NT，让外围服务能很快支持 NT」
> ③「把新需求固化下来……遵循长效文档机制……优先新需求，尽量不完全破坏原有设计」

---

## Context

### 动机（决定性）

替换内核的真实动机是 **buy vs build**：降低内核维护成本，把精力集中到上层（dashboard / AI 层 / 策略层）。**不是**回测保真度、不是速度。这个动机修正直接改变了方案排序——「策略资产零重写」「外围服务零/低改造」的权重高于「NT 的 L3 撮合保真度」。

### 研究结论（详见调研报告，36 来源）

NautilusTrader（NT，Rust 内核 + Python 控制面，LGPL-3.0）回测保真度显著高于自研内核，但对本项目有三个硬障碍：

1. **中国市场零覆盖**：无 A 股交易日历 / T+1 / 涨跌停 / 复权支持，维护者在 issue #4599 明确「近期路线图不含中国市场」，无公开 A 股回测先例；
2. **与 Go 技术栈正交**：无 Go 绑定、无对外 gRPC/REST 控制 API，C FFI 不含回测引擎 ⇒ 只能以**独立 Python 服务（侧车）**形态接入；
3. **时机风险**：v2.0.0 处于 RC 期，破坏性变更频繁。

**结论：整体替换否决**（25–45 人周，其中策略重写 6–12 + A 股规则 venue 级自建 6–12 是纯增量成本），但 NT 的价值可通过**侧车 + 契约化**以低一个量级的成本获取。

### 现状（代码库盘点）

- K0–K7 模块化内核主线已完成：模块契约冻结、StandardKernel 装配接管、流式 BarHandler、L2 算子、执行算法、paper 对账、WASM、信号注入全部落地。
- **策略层「定义在外、执行在内」**：策略配置存 Postgres（`/api/strategies` CRUD），执行是引擎进程内直调 `strat.GenerateSignals`——策略资产（约 1 万行，含 expression DSL 与插件）与内核的耦合点**只有这一个调用点**。
- **四契约已有现成雏形**：`BacktestRequest`/`TradingConfig`（输入）、`strategy.Signal` + K7 `quant.external_signals` 信号表（信号，含注入→成交 e2e 验证）、`BacktestResponse`/`domain.BacktestResult`（结果）、`contracts.BacktestRunner` 窄接口（执行）。

---

## Decision

**把内核降级为「可替换的执行器」，外围只依赖稳定契约；NT 以侧车 Python 服务按「信号—权重回放适配器」形态接入；Go 策略资产与外围服务零重写。** 分四条：

### D1 · 四契约冻结为内核边界

| 契约 | 内容 | 现状基础 | 补全项 |
|---|---|---|---|
| **RunSpec** | 回测输入 + 声明式规则配置（T+1 / 涨跌停 / 费率） | `BacktestRequest` + `TradingConfig` | 增加 kernel 路由字段；规则配置保持声明式（NT 侧由适配器解释） |
| **SignalSpec** | 信号供给（symbol / direction / weight / as_of） | `strategy.Signal` + K7 信号表 + `SignalStore`（as_of 拉取） | 「引擎进程内调 GenerateSignals」抽象为 `SignalProvider` 接口：本地注册表实现为默认，远程实现（HTTP / Redis）可插拔 |
| **ResultSpec** | 结果 schema（收益 / 夏普 / 回撤 / 交易明细 / 净值）+ **status 语义**（completed / invalid / failed） | `BacktestResponse` / `domain.BacktestResult` | 冻结为 canonical schema，指标口径（封闭系统日收益）写入规范；status 全链路一致呈现（含 UI） |
| **EngineRunner** | 执行入口 `RunBacktest(ctx, …) → *domain.BacktestResult` | `contracts.BacktestRunner`（runner.go 窄接口） | 可插拔 runner：Go 内核实现 + NT 侧车客户端实现**并存**，路由按 RunSpec 字段选择 |

### D2 · NT 侧车走「信号—权重回放适配器」

NT 侧实现一个通用 Python Strategy（权重回放适配器）：每个调仓日从 SignalSpec 信号流读目标权重 → 生成调仓订单。由此：

- **Go 策略资产零重写**（§6.2 工作流 4 的 6–12 人周整体消除）；策略层继续在 Go 侧开发维护，NT 只接管撮合 / 账务 / 费用内核；
- **A 股规则自建范围收缩到适配器层**：卖单 ≤ 昨仓可卖（T+1）、触板不追单（涨跌停）在适配器下单逻辑实现，不必扩展 NT venue 模型；印花税 / 过户费 / 最低佣金映射 fee model；
- **双内核可比性天然成立**：同一信号流喂两个内核，数值对齐是受控实验而非迁移阻塞项。

代价（已知、可接受）：策略被抽象为「日频目标权重」，NT 的逐订单事件模型仅部分发挥——但这本来就是现有信号—权重策略体系的表达能力上限，无损失。

### D3 · 外围服务 NT-ready，不为 NT 重写

Dashboard 与 AI 层**近零改造**（只依赖 canonical 结果 schema，不泄露内核特有字段）；编排层（job / batch / walk-forward）**留在 Go 侧**（NT 无内置参数网格/走查编排，Go 侧当编排器反而正确）；数据层增量提供 Parquet 导出（NT `ParquetDataCatalog` 口径）。

### D4 · 分阶段推进 + 决策门

| 阶段 | 内容 | 人周 | 独立价值 |
|---|---|---|---|
| 1. 契约化与解耦 | `SignalProvider` 接口化、RunSpec/ResultSpec 冻结、EngineRunner 可插拔 | 6–10 | **不接 NT 也成立**：降低内核耦合、明确模块边界 |
| 2. NT 侧车 PoC | 侧车服务 + 数据馈送 + 权重回放适配器（A 股最小规则）+ 结果映射 + 双跑对齐 | 10–16 | 实测回答「A 股规则在 NT 上的真实成本」 |
| 3. 双内核生产化 | kernel 路由、对齐回归、CI/CD（Python 进流水线）、监控、运维 | 6–10 | Go 内核降级为可退役的备选实现 |

**决策门（阶段 2 完成后）**：凭「双跑对齐数据 + NT v2.0.0 正式版发布」裁决——① 生产化推进 / ② 长期双内核 / ③ 停在阶段 1 回退；届时 VeighNa（A 股覆盖更好的 Python 候选）并列评估；K4 遗留的「impact 是否默认滑点」也在此门一并裁决。

> **✅ 决策门裁决记录（2026-10-14，若曦拍板）**：
> **裁决一（三选一 → ① 推进生产化）**：凭 K15 双跑数据（2024 全年两标的 × 两信号流，total_return |Δ| ≤ 0.086%、实测容差 ≤0.2%、CI `align-gate` 门禁上线，报告见 core 仓 `docs/design/kernel/k15-dualrun-report.md`）+ K16 六件套落地（2026-10-13，阶段 3 实际已推进），正式化为「推进生产化」。③ 回退（废弃侧车、落「借鉴设计增强 Go 内核」）**降级为「NT 正式版升级失败」的兜底路径**，不再是并列选项。版本锁定策略同步收口：**钉 rc6（`nautilus_trader==2.0.0rc6`）、不追 nightly**；NT v2.0.0 正式版发布后按 `guides/nt-sidecar-operations.md` 升级预案收口，**双跑对齐回归为升级验收门**。
> **裁决二（VeighNa 并列评估：取消）**：其触发条件是「诉求确认为『Python + 中国市场』」——从头未成立：生产后端必须 Go（D5），Python 仅侧车且已被 NT 以双跑对齐验证占住。VeighNa 主场是期货 CTA + A 股柜台网关接入（中泰 XTP/国君 HFT/华鑫/东方 OST/CTP 等），非日频股票回测内核；且评估它需为第二个 Python 引擎复刻 K14+K15 全套对齐资产（费率 parity + 权重回放 + 逐指标 diff），无对齐基线可比。其真正价值（A 股柜台网关）属将来**实盘接入阶段**的选型问题，届时以**网关候选**单独评估，不作为回测内核候选。
> **裁决三（K4 impact 默认滑点：维持 `fixed`）**：`"impact"` 保留为显式选项（`ExecutionConfig.ImpactSigma`），不设默认——现阶段 ADV 以当根 `quote.Volume` 代理，证据不足以支撑默认；K15 双跑基线以滑点=0 跑出，翻默认会使基线失效需整体重跑；**待有真实成交回执数据后单独再裁**（届时可用实测滑点校准模型参数）。

### D5 · 约束修订（显式）：「后端必须 Go」

原工程约束「后端必须使用 Go」修订为：

> **生产后端（对外服务、API 层、编排、策略层、默认内核）必须使用 Go；实验 / 执行侧车允许 Python（独立进程），且必须：① 只经 EngineRunner/四契约与 Go 侧交互，不直连数据库、不直面用户；② 进程隔离（LGPL-3.0 服务化使用，不链接、不嵌入，规避传染与 FFI 缺口）；③ 鉴权 / fail-closed / CORS 白名单 / loopback-only insecure 模式等安全约束一律不放松，侧车不对外暴露端口（仅 Go 侧回环访问）。**

除此之外不放松任何既有安全与工程约束（CI `-race`、RBAC `RequireRole`、ETL 错误上抛、schema 内联 DDL 等）。

---

## 否决的备选方案

| 方案 | 否决理由 |
|---|---|
| **A 整体替换**（NT 侧车承担全部回测） | 25–45 人周；策略重写 6–12 人周；A 股规则 venue 级自建 6–12 人周且无先例；v2.0.0 RC 破坏性变更风险；与 buy vs build 动机相悖（省下的内核维护被 A 股适配器 + 双跑偏差治理吃掉） |
| **cgo / FFI 嵌入 NT** | 引擎无 C ABI（FFI 只覆盖数据模型层）；进程内链接触发 LGPL 传染评估 |
| **C 只借鉴设计、不接 NT**（移植 fill model 到 Go 内核） | 保值动作（保留为阶段 1 的附带项），但 buy vs build 目标不兑现；若决策门裁决回退，落回此方案 |
| **VeighNa 直接替代** | A 股覆盖更好，但同样是 Python 栈、且未经本项目双跑验证；**决策门 2026-10-14 已裁决：不并列评估**（理由见 D4 末尾裁决二；其柜台网关价值归实盘接入阶段单独评估） |
| **为 NT 重写策略层 / 外围服务** | 直接违反本 ADR 的动机（上层才是要集中精力的地方） |

---

## 后果

**正向**：内核维护成本可转移给上游社区（决策门通过后）；策略层 / dashboard / AI 层零重写；阶段 1 独立成立、风险逐段释放；双内核可比性与 K5「同构对账」方法论同源。K3/K4/K6 的执行算法 / 冲击模型 / 沙箱投入全部保值（恰是双内核对比的基线资产）。

**负向 / 风险**：双栈运维（Python 运行时进 CI/CD）；双跑口径对齐是持续成本；NT v2 破坏性变更需跟随；权重回放适配器拿不到 NT 的 L3 撮合全能力（已知取舍）；约束修订引入 Python 侧车的合规面（用进程隔离 + 不对外暴露收敛）。

**回退路径**：任一阶段可停。阶段 1 成果（契约化）永久保值；阶段 2 若对齐数据差，回退到「C 借鉴设计」方案，侧车代码废弃不影响主链路。

---

## 与既有文档 / 任务的关系

- **承接模块化内核蓝图（target-architecture-modular-kernel.md）**：D1–D5（实盘-ready / L2 优先 / 契约先行 / EventStore / 执行算法）与 K0–K7 成果**全部保留有效**，本 ADR 在其上加「可插拔 + NT-ready」一层；蓝图中「拍板后从中抽 ADR-031」的注记由本 ADR 承接（模块化内核的决策记录由蓝图 §11 + TASKS K0–K7 执行记录承载，不再另抽）。
- **与 ADR-027（服务拆分）同向**：内核可插拔 = 服务边界的进一步冻结；「簇①回测核心判定不拆」在本 ADR 下含义收窄为「**Go 回测核心作为默认实现不拆服务**，但执行器地位可插拔」。
- **与 ADR-012（strategy-service standby）兼容**：`SignalProvider` 的远程实现是将来形态，strategy-service 维持 standby 不变。
- **任务落地**：core 仓 `docs/TASKS.md` 新主线 K8–K16（阶段 1–3）；K7 遗留①②、OBS-01 遗留①②、K0-P2-1、K1-P2-1~4 按调研报告 §8.6 并入对应新任务。**K1-P2-1~4 已由 K12 收口（2026-10-12，D1 稳定性前置）**：① Boot ctx 语义裁决为「仅限调用期间」（契约注释 + `TestLifecycleCtxIsCallScoped` 钉住，保留超时中断能力）；② 回滚期 Stop 失败并进返回 error（不静默）；③ **资源归属裁决：连接池是应用层资源**（`EventStoreModule.Stop` 不关 pool，谁构造谁释放，同步 core 仓 kernel 蓝图 §4.2 与 `adapters.go` 注释）；④ 「模块不得在生命周期方法内回调 `Module()`」契约化 + AST 护栏（`internal/repoguard`）。

## 未决问题（决策门 2026-10-14 全部关闭）

- ~~[TBD] 双跑数值对齐的容差阈值~~ → **已关闭（K15 实测，2026-10-13）**：per-metric 阈值表写回设计文档 §8 表 1（total_return/annual ≤0.2%、sharpe/sortino/calmar ≤0.05 绝对、maxDD ≤0.5%；win_rate 与 total_trades 不设硬阈，笔数守恒由 fills 对账腿执法），门禁执法者 = `cmd/dualrun -gate`（CI `align-gate` job）。
- ~~[TBD] NT 版本锁定策略~~ → **已关闭（决策门裁决一）**：钉 `nautilus_trader==2.0.0rc6`、不追 nightly；正式版发布后按升级预案收口，双跑对齐回归为验收门。
- ~~[TBD] Parquet 导出的复权口径~~ → **已关闭（K13，§8.2）**：`ohlcv_daily_qfq` 原样消费（不做二次复权）+ UTC 午夜纳秒时间戳。
- ~~[TBD] 侧车部署形态与监控口径~~ → **已关闭（K16，§8.6）**：compose 服务（`docker-compose.nt.yaml`）+ 健康检查 + 监控指标（`nt_sidecar_calls_total{outcome}` 等）；运维文档在 `guides/nt-sidecar-operations.md`。

---

## 决议变更记录：内核定位扩展为双完整核心热转换（2026-10-14，若曦拍板）

**变更**：NT 定位从 D2 的「信号—权重回放适配器（对拍考官）」**扩展**为**完整第二核心**——路线二：**国内 Go 全栈（策略 + 执行）/ 国外 NT 全栈（全 feature，Python 策略 + 原生执行）/ 外壳共用（RunSpec 路由、ResultSpec 回执、审计、UI）**，按市场**热转换**（配置级切换，非 run 级迁移）。**这是扩展不是推翻**：D1–D5、决策门三裁决、A 股对拍线（国内线 NT 侧车现状）全部维持。

**与本 ADR 各条款的关系**：

| 条款 | 关系 |
|---|---|
| D1 四契约 | **维持并升格**——RunSpec `kernel` 路由字段正是热转换的切换载体，四契约同时是拆仓的接口面 |
| D2 权重回放适配器 | **范围收窄为国内对拍线**；国外线新增 NT 原生模式（数据模型 + 订单类型 + 原生执行），策略用 NT 原生写（新增而非重写，无 6–12 人周包袱） |
| D3 外围 NT-ready | **维持**，「外壳共用」即其延伸 |
| D4 决策门三裁决 | **维持**（推进生产化 / VeighNa 取消 / impact 维持 fixed） |
| D5 Go 约束 | **维持**——NT 全栈指「国外市场策略与执行在 NT 侧车内」，侧车仍是 Python 独立进程、经四契约交互、安全约束不放松 |

**配套蓝图（附件）**：[design/data-factor-unification.md](../design/data-factor-unification.md)——热转换语义五条、性能红线（高频永不跨进程）、规范层 schema、时间语义红线（PIT fail-closed）、因子三元组与热插拔、三期实施路线、拆仓/服务化评估（已归口 ADR-027 复评节）。

**未决处置拍板（2026-10-14，蓝图 §9 七条全按建议落实）**：

| # | 未决 | 处置 |
|---|---|---|
| 1 | ADR 归属 | ✅ **修订本 ADR（决议变更）+ 蓝图挂附件**（即本节） |
| 2 | 15 分钟聚合细则 | 实施工单成文（工单内容可独立执行验证） |
| 3 | 因子注册表治理细则 | 参考外部门户 content_hash 先例，实施期定稿 |
| 4 | 国外市场数据源选型 | NT 原生数据通道优先，另调研 → **调研报告已落档（2026-10-14）**：[design/d9-foreign-market-data-source-selection.md](../design/d9-foreign-market-data-source-selection.md) |
| 5 | 分期优先级与工单化 | 地基工单（规范层 + PIT 红线）最先，因子三期 + NT 原生模式按蓝图 §6，六字段写进 core 台账 |
| 6 | 多频率进回测策略视野 | 「日频决策 + 日内特征」先行，分钟级撮合另立工单 |
| 7 | 拆仓结论归口 | ✅ 已落 ADR-027 复评节（2026-10-14） |

**文档同步纪律（用户 2026-10-14 指示，固化为流程约束）**：**上层设计（本 ADR / 蓝图）任何变更，必须同步更新下游文档——SPEC（契约 schema）、数据流（data flow / 内核蓝图数据流图）、schema（DB DDL 口径）、ARCHITECTURE.md**；不同步即为变更未完成，评审时按缺项打回。
