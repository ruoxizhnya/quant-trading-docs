# quant-trading-docs

Quant Lab 的**跨领域 / 高层 / 整体文档**。从 monorepo `quant-trading` 拆出（2026-10-09）。

## 定位与边界

本仓只放**不属于任何单一领域**的文档——高层叙事、架构决策记录（ADR/ODR）、
项目历史档案、运维指南、跨领域功能规格。

切分判据是 **DDD**，落在「领域边界」这条线上：

| 文档类型 | 归处 |
|---|---|
| **活跃台账**（`TASKS.md`，每次任务都改） | 留在 core（`quant-trading`）——它与 commit 同生命周期 |
| **架构决策记录**（ADR / ODR，Accepted 后只读档案） | 本仓 |
| **高层叙事**（愿景 / 产品 / 架构 / 路线图） | 本仓 |
| **项目历史档案**（archive） | 本仓 |
| **运维指南**（本地开发 / 部署 / 数据依赖） | 本仓 |
| **单一领域的设计文档**（内核蓝图 → core，前端设计系统 → ui，agent 设计 → agent） | 各自领域仓 |

其它三仓：

- [`quant-trading`](https://github.com/ruoxizhnya/quant-trading) — 内核（仪器）
- [`quant-trading-agent`](https://github.com/ruoxizhnya/quant-trading-agent) — AI 实验员
- [`quant-trading-ui`](https://github.com/ruoxizhnya/quant-trading-ui) — 前端

## 目录

| 路径 | 内容 |
|---|---|
| `docs/ADR.md` + `docs/adr/` | 架构决策记录（跨领域治理档案） |
| `docs/ARCHITECTURE.md` / `VISION.md` / `PRODUCT.md` / `ROADMAP.md` | 高层叙事 |
| `docs/guides/` | 本地开发 / 部署 / 数据依赖 |
| `docs/design/equitydeep/` | EquityDeep 功能规格（跨领域） |
| `docs/archive/` | 项目历史档案（ODR / 计划 / 报告 / 研究 / 被取代的 ADR） |

## ⚠️ 跨仓契约

`docs/openapi.yaml`（后端 API 契约）**不在这里**——它留在 core 仓，因为被 Go 的
`//go:embed` 嵌进二进制。本仓引用它时用 core 仓路径，不复制。

## 文档治理

活跃层文档的坏链检查随 core 仓 CI（`tools/check_doc_links.py`）。本仓拆出后，
跨仓链接需改为绝对 URL（GitHub 路径）或相对各自仓内的路径。
