---
status: evergreen
type: how-to
last-verified: 2026-10-14
verified-by: K16 ③ 交付时逐条对代码核实——§1 三条纪律与 API 盯防清单来自
  core 仓 `docs/design/kernel/kernel-pluggable-nt-ready.md` §8.5（2026-10-13 裁决）；
  §2 fail-closed 语义经 dualrun 停侧车实测（exit=1 点名 kernel "nautilus" unreachable）；
  §3 命令与 nt-sidecar/README.md §2 逐字核对；§5 指标名与 `pkg/observability/metrics.go`
  核对；§6 微坑全部来自 K15-R1 / K16 ② 实测踩坑记录。
  §4（D7 市场档案与原生模式）由 D7 交付时核实：`--market` 参数与
  `sidecar/markets.py` 的档案表逐字核对；§4.4 的两处导出缺陷经真库实测复现，
  并已验证「补齐后重导 catalog ⇒ `cmd/dualrun -gate` 全绿」（2026-10-14）；
  同日按口径裁定（成交量单位 = 股）**正式修复**并复验（详见 §4.4 末「已修复」段，
  含对国内线数值影响与尚未覆盖的 fallback 缺口）。
---

# NT 侧车运维：版本升级与故障回退

> NautilusTrader 侧车（`nt-sidecar/`，独立 Python 进程）的运维手册。
> **首节是 NT 版本升级流程**——按 K16 ③ 验收要求，它是本文档存在的首要理由：
> 正式版发布后立升级工单时，执行 agent 只需读完 §1 即可开工。
>
> 侧车的架构约束（回环 only / 不直连 DB / 只经四契约）见 core 仓
> `docs/design/kernel/kernel-pluggable-nt-ready.md`（下称「设计文档」）§8；
> 本文只写**运维操作**，不重复设计裁决。

---

## 1. NT 版本升级流程（升级工单的唯一入口）

**触发**：`nautilus_trader` 发布需要跟随的版本（2.0.0 正式版，或 RC 期某个
含落在我们面上的 breaking change 的版本）→ 立升级工单，按本节执行。

**爆炸半径（先读这个，决定工作量预期）**：NT API 变化只波及 `nt-sidecar/`
一个 Python 目录（engine.py / catalog.py / runspec.py / strategies.py /
ashare.py）。Go 核心、四契约、ResultSpec 冻结字段全部绝缘。最坏情况 =
改十来个 import 点 + Python 测试重跑（设计文档 §8.5.1）。

### 1.1 三条纪律（硬约束，写进升级工单验收）

1. **版本 pin**：`nt-sidecar/requirements.txt` 锁**精确版本**
   （当前 `nautilus_trader==2.0.0rc6`），无 `>=`/`~=`，不追 nightly；
   Dockerfile 注释同步标注 pin 理由。
2. **升级预案即交付物**：升级完成时本文档 §1 与 §6 必须同步更新
   （API 面变化、新踩的坑），否则工单不算收口。
3. **只盯三个真正在我们面上的 API 面**（其余 breaking change 不管——
   Rust 侧类型、IB/Kraken 等我们不用的适配器都在爆炸半径外）：

| # | API 面 | 消费点 | 要核对什么 |
|---|---|---|---|
| 1 | `BacktestRunConfig` / `BacktestVenueConfig` / `BacktestDataConfig` / `BacktestEngineConfig` 签名 | `nt-sidecar/sidecar/engine.py` 组装处 | 构造参数名/顺序/必填项 |
| 2 | fee model 接口（`ashare.AShareFeeModel.get_commission` 覆写语义） | `nt-sidecar/sidecar/ashare.py` | **返回全项 total**（佣金+印花税+过户费）的 K15-R1 口径不许退化成只返回佣金 |
| 3 | 报告 API：`generate_fills_report` / `generate_account_report` / `get_engine_portfolio` | `engine.py` 的 `_map_result` | 返回结构/列名/索引语义 |

### 1.2 升级步骤（半天量级，按序执行）

1. **bump 版本**：`requirements.txt` 改 pin（一次只升一个版本号）。
2. **Python 全套测试**：`cd nt-sidecar && .venv/bin/python -m unittest discover -s tests -t .`
   ——必须全绿（当前 63 条）。
3. **重跑双跑对齐**（升级安全网，K15 工具）：
   ```bash
   go run ./cmd/dualrun -strategy bh
   go run ./cmd/dualrun -strategy quarterly
   ```
   前置：PG `ohlcv_daily_qfq` 就绪 + 侧车已按 §3 起好。**阈值表逐项比对**
   （total_return |Δ| ≤ 0.2% 等，见设计文档 §8 表 1）；超阈即升级未过，
   先归因再收口，不得放宽阈值。
4. **门禁模式终验**：`go run ./cmd/dualrun -strategy bh -gate` exit=0。
5. **收口**：更新本文档 frontmatter `last-verified` / §1.1 当前版本号 /
   §6 新坑；升级工单以「对齐回归绿」为验收门关闭。

**验收门 = 对齐回归**。Python 测试绿但双跑超阈 = 升级未通过——那说明
数值行为变了，必须归因（价源/费用/撮合）而不是调阈值。

---

## 2. 故障回退流程

### 2.1 先理解语义：零回退是结构属性，不是配置

`nautilus.Runner` **没有任何回退分支**：侧车不可用（拒连/超时/5xx/
`status:failed`）时全部路径返回 `(nil, err)`，绝不会静默换成 go 内核的
结果（FR-1.3）。所以「故障回退」不是自动的，是**运维动作**：

| 故障现象 | 系统行为（不用做任何事） | 运维动作 |
|---|---|---|
| 侧车进程挂了，请求 `kernel:"nautilus"` | 请求显式 `failed`，错误点名 `kernel "nautilus" unreachable` | 按 §3 重启侧车；或临时按 2.2 退 go |
| 侧车没起，但服务已注册 nautilus | 同上（执行期显式透出，不吞错） | 同上 |
| `NT_SIDECAR_ENABLED` 未设（生产缺省） | `kernel:"nautilus"` 请求在路由处被拒，错误点名内核名 + 已注册集合 | 无需动作（这就是缺省形态） |
| 对齐门禁超阈（CI `cmd/dualrun -gate`） | exit 2 + critical 告警（`$DUALRUN_ALERT_WEBHOOK`） | 按 §1 归因；未归因前**不得**放行 nautilus 流量 |

### 2.2 临时退到 go 内核（唯一合法回退路径）

生产放量的控制点是**路由开关**，回退 = 把它关回去：

1. `NT_SIDECAR_ENABLED` 置空/0（**不要**改代码、不要改 RunSpec），
   重启 `cmd/analysis`。装配日志应出现
   `NT sidecar kernel not registered (production default: go-only…)`。
2. 此后所有回测走 go 内核（RunSpec 不带 `kernel` 或带 `"go"`）；
   仍带 `"kernel":"nautilus"` 的请求会**显式被拒**（点名内核名），
   不会拿到一份冒充的 go 结果。
3. 停侧车进程（按 §6 的端口清单确认真停了）。
4. 恢复时反向操作：起侧车 → `/healthz` 200 → `NT_SIDECAR_ENABLED=1`
   重启服务 → 跑一条 `kernel:nautilus` 回测确认 round trip 成功。

**注意**：开关值非法（打错字）在装配期直接报错退出，不静默当关闭——
看到启动失败先查这个环境变量的拼写。

---

## 3. 起停与健康检查

```bash
cd nt-sidecar
.venv/bin/python -m sidecar --port 9100 --catalog nt-sidecar/catalog
```

- 服务**只绑 127.0.0.1**（ADR-031 D5），`--host 0.0.0.0` 直接报错退出（设计如此）。
- 健康检查：`curl -s http://127.0.0.1:9100/healthz` → `{"status":"ok","kernel":"nautilus"}`。
- Go 侧地址：`NT_SIDECAR_URL`（缺省 `http://127.0.0.1:9100`）。
- **数据喂给侧车的是 Parquet catalog**（`tools/export_bars.py` 离线导出），
  侧车服务本身**永不直连数据库**（D5）。底表刷新后重跑导出。
- **Windows 禁用 compose 容器形态**：Docker Desktop 的 `network_mode: host`
  进的是 Linux VM 网络命名空间，容器 healthz 绿但宿主 127.0.0.1 拒连
  （「看着全绿、Go 侧全挂」的假象）。生产形态 = 宿主独立进程
  （设计文档 §8.6 有实测对照表）。

---

## 4. 市场档案与 NT 原生模式（D7）

D7 给侧车加了**第二条执行通道**：A 股线（`weight_replay` + 最小规则集）不动，
新增**国外线 NT 原生模式**——NT 自带的订单类型与执行模型，**不做 A 股规则魔改**
（无 T+1、无涨跌停、无整手）。依据：ADR-031「决议变更记录」+ 蓝图 §2.1（按市场
热转换，配置级切换）。

### 4.1 两层配置，各管一段（不要混）

| 层 | 位置 | 管什么 |
|---|---|---|
| **路由** | core 仓 `config/analysis-service.yaml` → `kernel_routing.markets`（如 `CN: go`、`US: nautilus`） | 「哪个市场由哪个内核跑」。`POST /api/backtest` 接受可选的 `market` 字段，在路由**之前**解析成内核名写进既有的 `kernel` 字段 —— 四契约零改动（D7 ④） |
| **执行** | 侧车启动参数 `--market`（§4.2） | 「NT 内核跑这个市场时用哪套原生规则」：币种、精度、费率模型、执行模式。表在 `sidecar/markets.py` 的 `MARKET_PROFILES` |

两层都 fail-closed：未登记市场在**路由侧** 400（点名市场 + 已登记集合），在**执行侧**
装配期直接退出（`MarketError`）。**任何一层都不回退**——回退的代价不是报错，而是
一份用错规则/内核算出来的、看起来完全正常的回执（FR-1.3 的病，换了个维度）。

### 4.2 起一个国外市场的侧车

```bash
cd nt-sidecar
.venv/bin/python -m sidecar --port 19100 --market US --catalog <us-catalog>
```

- `--market` 缺省 `CN`：不带该参数的启动命令行为**逐字不变**。
- **一个侧车实例服务一个市场**：US 档案的实例只跑该市场的样本数据；CN 对拍线的
  实例仍按 §3 起（`--market` 不传）。
- 档案与 catalog 必须配套：US 档案要求 catalog 里的 instrument 是 USD 计价的
  （否则 `resolve_instruments` 找不到，显式报错）。

### 4.3 加一个新市场（三步，零路由代码改动）

1. `sidecar/markets.py` 的 `MARKET_PROFILES` 加一行（币种 / 精度 / 费率模型 / 执行模式）。
2. `config/analysis-service.yaml` 的 `kernel_routing.markets` 加一行（市场 → 内核）。
3. 为该市场准备数据 connector + catalog（本工单范围外，见 D9 选型报告）。

### 4.4 导出路径与成交量单位（2026-10-14 已修复）

> 历史沿革：本节原名「已知阻断（不属于 D7）」。下面的问题描述**保留原样**作为
> 证据链（记录当时的实测形态），修复记录见本节末「已修复」段。

§3 说「底表刷新后重跑导出」——**该动作当前会失败**。两处独立缺陷，都在 D1
（commit `b49629e`）引入的导出路径上：

1. **字段名不匹配**：`tools/export_bars.py` 的 SELECT 返回数据库列名 `trade_date`，
   而翻译层 `mapping.BAR_FIELD_MAP` 按**规范层字段名** `date` 取值 ⇒
   `KeyError: mapping: 行数据缺少规范层字段 'date'`。
2. **成交量严格化 vs 真实数据**：翻译层的 `as_quantity` 拒绝非整数成交量（"取整会
   悄悄改口径"，立意正确）；而刷新后的 `ohlcv_daily_qfq.volume` 是**带两位小数的
   手**（实测抽样 30 万行，81.5% 非整数）⇒ `ValueError`。

**后果**：`cmd/dualrun -gate` 用的 catalog **无法按文档动作刷新**。SYNC-01 之后调度器
已会真的同步底表（复权基准因此移动、volume 精度变化），于是对拍门禁会因**产物陈旧**
而红——看起来像数值分歧，实际是产物过期。

**实测证据（2026-10-14）**：把上面两处临时补齐后重导 catalog，再跑
`go run ./cmd/dualrun -strategy bh -gate`，输出
「对齐门禁通过：全部指标在 K15 阈值内，确定性腿一致，无成交 mismatch」——
**对齐本身完好，红灯纯属产物陈旧**。

**为什么当时不当场修**：第 2 条是**口径裁决**（成交量单位：手 vs 股，取整/换算发生在
规范层还是翻译层，见 `mapping.py` 的「换算请在规范层做」注释），改它会动国内线的
数值；D7 的 ④ 约束明令「不动国内对拍线现状」。须另立工单把口径定死后修。

**已修复（2026-10-14）**：口径定死为规范层**已经声明**的单位「股」
（`SPEC.md` 规范层 Schema 的 OHLCV.Volume 一直写「成交量（股）」，前端
`utils/format.ts` 的 `fmtVolume` 也按股格式化 —— 是**实现违背了已写明的契约**），
两处缺陷随之消解：

1. **字段名（已修）**：`export_bars.py` 的 SELECT 改输出规范层列名
   `trade_date AS date`。改 SELECT 而不改映射表是因为 `date` 是跨语言契约定下的
   json tag（"加一种数据类 = 映射加一行"的机制不动）。
2. **成交量单位（已统一）**：
   - 同步侧 `pkg/data` 按 `sharesPerLot`（一手 = 100 股）换算 + `math.Round`。
     取整是必须的：真库 392.6 万行里 **37 万行**的 `volume×100` 在 float64 下
     不是精确整数（最大误差 4.77e-07），不取整会被 `as_quantity` 的
     `is_integer()` 判成「不是整数」→ 整批数据被判损坏。
   - 存量由 **migration 052**（`pkg/storage/postgres.go` 的
     `migrationStatements()`，文档副本 `docs/migrations/052_ohlcv_volume_lots_to_shares.sql`）
     对齐。×100 **不**幂等，故用**哨兵 + 内容**双守卫：① `data_migrations` 表
     （本项目首个 DML 迁移标记设施，后续同类迁移复用）记录 052 已执行 ⇒ 至多生效
     一次；② 库里还有非整数（= 手的小数）才值得整表换算（防「清库后用新代码重灌」）。
     **不**写逐行 WHERE —— 真库 **19.24%** 的行本来就是整数「手」，逐行条件会漏掉
     它们 ⇒ 半股半手的混合表。
     **只靠 ② 不够**：迁移后若又落入手数据（旧二进制未换 / 手工导入 / 多节点混部），
     ② 会再次成立 ⇒ **整表**再 ×100 ⇒ 已迁移的股数据静默变成 ×10000。对照实验
     （临时表实跑）：初值 `{1.5, 2000, 3.7}` 首次迁移得 `{150, 200000, 370}`，
     再落一行手值 `9.9` 后跑无哨兵版得 `{990, 15000, 37000, 20000000}`，
     加哨兵版保持 `{9.9, 150, 370, 200000}`。
   - `as_quantity` 本身**不需要改**：上游已是整数股，它的整数纪律继续生效。

**验证（2026-10-14 实测）**：导出真跑通过（catalog 值 = 库值，逐条一致，全整数）；
`go run ./cmd/dualrun -strategy bh -gate` 输出「对齐门禁通过：全部指标在 K15 阈值内，
确定性腿一致，无成交 mismatch」——**对拍恢复全绿**。破坏验证三组（删 date 别名 /
去 ×sharesPerLot / 去 math.Round）均改坏变红、还原变绿。

**对国内线数值的影响**：volume 数值整体 ×100，其中两处是**修正既有量纲错位**
（订单量的单位是股，此前却与「手」相比）：`pkg/backtest/tracker` 的流动性阈值
`maxLiquidity = volume × 0.1` 此前偏小 100 倍；`pkg/execution` 的 impact 滑点
`orderQty/adv` 此前偏高 100 倍（默认 `SlippageModel="fixed"`，故默认不生效）。
因对拍的两内核同源同变，对齐仍在 K15 阈值内。

**尚未覆盖（登记在案）**：日线 fallback 链还有 mootdx / eastmoney
（`cmd/data/registry_init.go` 的 `DataTypeOHLCDaily` 链 = tushare → mootdx →
eastmoney）。mootdx 的 `vol` 单位在项目内**无实测依据**（TDX 惯例是「手」），
本单**未改**——启用该 fallback 前须实测并用同一 ×100 对齐。当前
`ohlcv_daily_qfq` 的 392.6 万行全部来自 tushare，故该缺口不影响现网。

---

## 5. 监控口径

| 盯什么 | 指标 | 说明 |
|---|---|---|
| 侧车延迟 | `nt_sidecar_call_duration_seconds{outcome}` | Histogram |
| 侧车失败率 | `nt_sidecar_calls_total{outcome}` | Counter；失败 = `outcome!="ok"` 占比 |
| 回退次数 | **无此指标，也永远不会有** | Runner 无回退分支，恒 0 是结构属性，由测试执法（no-silent-fallback 断言） |

`outcome` 封闭词表：`ok / unavailable / timeout / internal`（未知码 fail-closed
折叠为 internal）——告警规则直接按 outcome 键控。每次 round trip 恰好一条观测
（成功/失败/超时/解码失败全覆盖）。

---

## 6. 已知微坑（排查时先排除这些）

1. **Windows 端口复用绑定（高频）**：9100 可同时被多个进程监听，连接随机
   分配到新旧代码 ⇒ 「同请求不同结果」诡异现象先查
   `Get-NetTCPConnection -LocalPort 9100` 再查代码。K15-R1 与 K16 ② 各撞到
   一次。**停侧车后务必确认端口真清了**（残留进程 PID 直接 kill）。
2. **NT `Logger.warning` 不接受 `%` 占位参数**（只接受已格式化字符串）。
3. **`generate_account_report(rid, venue=)` 返回 DataFrame 索引 = 事件时刻**，
   `total` 列 = 现金（非总权益）；日末现金须按日取最后一事件结转
   （`synth_equity_curve` 口径）。
4. **fills 的 symbol 带 `.SIM` venue 后缀**，与 catalog closes key 不匹配时
   会**静默算出恒 0 持仓**——`_symbol_of(value, venue)` 剥后缀。
5. **Python 版本**：rc6 要求 `>=3.12,<3.15`（rc7 起移除 3.12）；侧车实测
   3.13.14。venv 用错版本会在 import 期就炸。
