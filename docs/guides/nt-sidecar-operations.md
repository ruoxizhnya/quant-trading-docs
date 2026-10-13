---
status: evergreen
type: how-to
last-verified: 2026-10-13
verified-by: K16 ③ 交付时逐条对代码核实——§1 三条纪律与 API 盯防清单来自
  core 仓 `docs/design/kernel/kernel-pluggable-nt-ready.md` §8.5（2026-10-13 裁决）；
  §2 fail-closed 语义经 dualrun 停侧车实测（exit=1 点名 kernel "nautilus" unreachable）；
  §3 命令与 nt-sidecar/README.md §2 逐字核对；§4 指标名与 `pkg/observability/metrics.go`
  核对；§5 微坑全部来自 K15-R1 / K16 ② 实测踩坑记录
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
2. **升级预案即交付物**：升级完成时本文档 §1 与 §5 必须同步更新
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
   §5 新坑；升级工单以「对齐回归绿」为验收门关闭。

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
3. 停侧车进程（按 §5 的端口清单确认真停了）。
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

## 4. 监控口径

| 盯什么 | 指标 | 说明 |
|---|---|---|
| 侧车延迟 | `nt_sidecar_call_duration_seconds{outcome}` | Histogram |
| 侧车失败率 | `nt_sidecar_calls_total{outcome}` | Counter；失败 = `outcome!="ok"` 占比 |
| 回退次数 | **无此指标，也永远不会有** | Runner 无回退分支，恒 0 是结构属性，由测试执法（no-silent-fallback 断言） |

`outcome` 封闭词表：`ok / unavailable / timeout / internal`（未知码 fail-closed
折叠为 internal）——告警规则直接按 outcome 键控。每次 round trip 恰好一条观测
（成功/失败/超时/解码失败全覆盖）。

---

## 5. 已知微坑（排查时先排除这些）

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
