# ADR-031: 内核可插拔 + 外围服务 NT-ready —— buy vs build 路线（NautilusTrader 侧车，整体替换否决）

> **Status**: **Accepted**（2026-10-12 若曦拍板固化）——阶段 1（契约化）与阶段 2（NT 侧车 PoC）立即执行；阶段 3（双内核生产化）与「Go 内核是否退役」由阶段 2 后的**决策门**裁决。
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
| **VeighNa 直接替代** | A 股覆盖更好，但同样是 Python 栈、且未经本项目双跑验证；留到决策门并列评估，不预先押注 |
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

## 未决问题（[TBD]，到决策门前必须关闭）

- [TBD] 双跑数值对齐的容差阈值（按指标分别定义，须在阶段 2 实验中实测确定）。
- [TBD] NT 版本锁定策略（PoC 钉 rc 版本还是跟随；v2.0.0 正式版发布时间表）。
- [TBD] Parquet 导出的复权口径与 `ohlcv_daily_qfq` 的对应关系（阶段 2 数据馈送任务内裁决）。
- [TBD] 侧车的部署形态（compose 服务 vs 独立进程）与监控口径。
