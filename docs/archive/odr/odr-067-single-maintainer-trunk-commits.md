# ODR-067: 单人维护期允许直接提交 main（分支约定改判）

> **Status**: Accepted
> **Date**: 2026-10-14
> **Category**: Process
> **Related ADRs**: [ADR-023](../../adr/adr-023-ai-experimenter-lab.md)（单人自托管实验室 —— 「多租户 / RBAC / 计费 / 高可用都是纯负担，可砍」是同一条判断的前例）, [ADR-027](../../adr/adr-027-modular-decomposition-to-independent-services.md)（AI 拆仓：一次改动横跨 core / ui / agent 三个仓，PR 流的成本被放大）

## Context

A-07（状态三值全站收口）收口时，三个仓的活都在**从未推送过**的本地 feature 分支上：
`quant-trading-ui` 的 `feat/design-tokens-v2-palette-sweep` 领先 `origin/main` **44 条**，
是那次改动的唯一副本。主任（若曦）随后把三条分支推送、合并进 `main`，并把四个仓的
active branch 都设回 `main`，给出裁决：**「目前就我一个人在维护做这些，你如果下一步要
继续可以直接 commit 在 main 上。」**

问题不在授权本身，在于**授权与文档相反**。`AGENTS.md` 有两处写着旧约定：

| 位置 | 原文 |
|---|---|
| §8.5 规范 3（原子提交 → 提交前检查清单） | 「分支策略: 不直接提交到 `main`，使用 feature branch」 |
| §9 行为边界 → Never Do | 「绝不直接提交到 `main` 分支 — 使用 PR 工作流」 |

只要这两行不改，下一个会话（或换一个 agent）读到的是硬约束，就会**按文档拒绝直推 main**，
或者在「用户口头授权」与「文档红线」之间自行择一 —— 后者最坏，因为它把选择权悄悄交给了
执行者。本项目的文档自维护协议（Rule 4：「是否改变了项目约定？→ 更新 AGENTS.md」）要求的
正是把裁决落到文档，而不是让它停在对话里。

## Decision

**改判：单人自托管维护期，本仓允许直接提交 `main`。**

三条边界，防止这条授权被读成「什么都可以直推」：

1. **他人贡献仍走 PR**。改判的是"自己给自己开 PR 并自批"这段 ceremony，不是评审通道本身。
2. **上游仓不套用**。`nautilus_trader` 等外部仓有自己的 `AGENTS.md`（明确要求 base on
   `develop`、target `develop`、非经要求不 push），本授权对它们无效。
3. **补偿门禁写进 §8.5 同一行**，不留抽象口号：受影响包的测试 + `go vet` + `gofmt -l`
   **当场跑过**；一个 commit 只做一件事；**不得把新红混进既有红**。

第 3 条是这次改判真正的风险对冲。失去 PR 这道外部检查点后，唯一能区分「这条提交弄坏了」
与「本来就坏的」的，是**红必须逐条登记在案** —— 即 ui 仓 `docs/TASKS.md` 的 AUD-61 登记表
机制（每条红改 `test.fixme` 并登记一行，破坏验证要求「去掉 fixme 必须重新变红」）。
2026-10-14 实测全量 e2e 是 **61 failed / 116 passed / 2 skipped**，所以「套件绿了才准提交」
这条门**现在物理上不存在**，AUD-61 阶段一因此在 trunk-based 下优先级上升。

**为什么选「改文档」而不是「口头授权、文档不动」**：文档不动 = 约定处于两套真相并存状态，
而这两套的优先级没写清。本轮（A-07）刚为同一族问题收过一次口 —— `BacktestEngine.vue`
自带一份与全站逐字相同但会各改各的 status switch，`BacktestCompare.vue` 贴原始 Code 而
别处显示人话。分支约定写在**两处**，所以两处**一起改**，并在新措辞里点名那条前例。

## Consequences

**正面**

- 去掉一次自说自话的仪式：A-05~A-07 期间 PR #1 的描述是我写、合并是主任点，中间没有任何
  独立信息增益 —— 单人仓的"PR 工作流"实际形态就是「同一人审两次」。
- 跨仓改动（core + ui + agent 一次任务动三个仓）不再需要三条 feature branch + 三个 PR。
- 本地唯一副本的风险消失：直推 main 意味着**每次提交都有远端副本**。本次盘点发现 ui 那 44 条
  只存在于那台机器上，是这条改判最硬的即时理由。

**负面 / 代价**

- 没有外部检查点。`main` 上的错误提交对所有拉取者立即可见，回滚从「PR 不合并」变成
  「revert 一条已进主干的提交」。
- 门禁强度从「CI + 人审」降为「agent 自审 + 提交前当场跑测试」。**这把纪律负担全部转给
  提交前动作**，而本仓 CI 历史上不含 `-race` 与前端门禁（见 ODR-065 的 High 项）。
- 与"红登记"耦合：如果不做 AUD-61 阶段一，直推 main 会让红清单与提交历史无法对齐。
  这是本 ODR 明确记下的**依赖**，不是已完成状态。

## Artifacts

| 文件 | 改动 |
|---|---|
| `quant-trading/AGENTS.md` §8.5 规范 3 | 「分支策略」行改判 + 补偿门禁写成可执行清单 |
| `quant-trading/AGENTS.md` §9 Never Do | 该条从「绝不直接提交 main」改为「绝不绕过评审直接推**他人贡献的分支**」，并写明适用范围与上游仓例外 |
| `quant-trading/AGENTS.md` 尾注 | 新增改判记录；顺带登记一处**未修**的既有漂移：本文件引用的 `docs/PRODUCT.md`、`docs/ADR.md`、`docs/VISION.md`、`docs/adr/`、`docs/archive/odr/` 在 core 仓已不存在（文档拆仓后归 `quant-trading-docs`，`docs/archive/odr/` 现存 0 个文件），批量改链不属本次改动 |
| `quant-trading/docs/TASKS.md` | 登记 AUD-61 优先级上升的理由 |
| `quant-trading-docs/docs/ADR.md` | ODR Index 新增 ODR-067 行 + 版本号与尾注 |

## Metrics

- 改动 **1 个约定 / 3 处文字**（§8.5 一行、§9 一行、尾注一段），跨 **2 个仓**（core 正文 + docs 索引）。
- 改判的直接动因可量化：ui 44 条、core 6 条、agent 2 条提交当时**全部只存在于本地**。
- 现状基线（供后续核对，非本 ODR 的成果）：全量 e2e `61 failed / 116 passed / 2 skipped`；
  单测 `vitest 354 passed / 28 files`、`vue-tsc` 0 错。

## Lessons Learned

1. **口头授权不落文档就等于没给**。它的失效方式是"下个会话看不见"，而表现是 agent 拒执行或
   自行择一 —— 两种都比改一行文档贵。以后凡「改变工作流」的裁决，同一轮就落 AGENTS.md + ODR。
2. **同一约定写在几处就要同时改几处**。这次两条线（§8.5 与 §9）之间隔了 90 行，很容易只改
   看起来"更正式"的那条 Never Do。
3. **改判的代价要用可执行的补偿控制抵**，不能用「注意质量」这种词。「绿了才提交」在当前
   61 红的套件下是一句空话，所以真解是先把红登记在案（AUD-61 阶段一），而不是宣布门禁存在。
