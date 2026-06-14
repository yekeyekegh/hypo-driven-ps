# hypo-driven-ps

> **假设驱动的结构化诊断与求解** — 一个用于攻克「卡住的、原因不明的、多可能性」难题的 Claude / Agent Skill。
>
> *Hypothesis-driven, structured diagnosis & problem solving for hard, stuck, ambiguous problems.*

<p>
  <img alt="type: agent skill" src="https://img.shields.io/badge/type-agent%20skill-6f42c1">
  <img alt="scope: general problem solving" src="https://img.shields.io/badge/scope-general%20problem%20solving-0969da">
  <img alt="trigger: explicit only" src="https://img.shields.io/badge/trigger-explicit%20only-d29922">
  <img alt="language: 中文 / English" src="https://img.shields.io/badge/lang-中文%20%2F%20English-2da44e">
</p>

---

## 这是什么

`hypo-driven-ps` 把「调试式直觉」升级成一套**可审计、可续跑、反锚定**的诊断流程。它不只针对代码 bug——**性能、架构选型、研究、数据异常、agent 行为 / 环境问题**等任何「多个可能根因、一时定不下来」的难题都适用。

核心机制：

```
提出多个症结候选(假设)
        ↓  每个配「可测量的确诊判据」
按  信息增益 ÷ 验证成本  排序
        ↓  逐轮验证（真 / 假 / 无法判断）
每轮把 learning 回灌、重排整盘假设
        ↓
直到问题「经验证确实解决」
```

> ⚠️ **仅在用户显式调用时启动**（如 `/hypo-driven-ps`、「摆个假设板」、「用假设驱动诊断」），不做自动触发。

---

## 解决什么痛点

| 没有它 | 有了它 |
|---|---|
| 凭直觉乱试，改一处坏一处 | 每个怀疑点都有**可测量判据**，验证后才晋级为「确诊」 |
| 上下文被源码 / 日志塞爆，越查越乱 | 主 agent **不下场取证**，只维护假设板；脏活下放给一次性 subagent |
| 反复纠结同一个高概率假设 | 按**信息增益 ÷ 成本**排序，先用便宜的验证切小可能性空间 |
| 「我觉得八成是它」就开修 | **无确诊不动修复**；判据测不了只能记「无法判断」，不许装懂 |
| 诊断过程随对话蒸发 | 假设板 + append-only 日志 + verdict 文件，**全程可追溯** |

---

## 工作原理：三角色，主 agent 不亲自下场

```
┌─────────────────────────────────────────────────────────┐
│  主 agent（编排者）                                        │
│  提假设框架 · 排序 · 选每轮验哪个 · 据反馈重排假设板        │
│  ✗ 不读源码  ✗ 不跑测试  ✗ 不亲自取证  ✗ 不亲自修复        │
└───────────────┬─────────────────────────────────────────┘
                │  每轮起「全新」一对，靠 board + log 重建状态
        ┌───────┴────────┐
        ▼                ▼
┌───────────────┐  ┌───────────────┐
│ Diagnostician │  │   Reviewer    │
│ (opus)        │→ │ (opus)        │
│ 写测试·验判据 │  │ 读懂 test·重跑 │
│ ·取证·修复    │  │ ·批准/退回整改 │
└───────────────┘  └───────────────┘
   test-driven-      verification-
   development       before-completion
```

- **任一时刻只有一个 Diagnostician、一个 Reviewer，串行**，不并发多个假设。
- **每轮回收**：下一轮起全新一对，靠两份文件（board + log）重建状态，不靠跨轮记忆 → 反锚定、上下文有界。
- **轮内交接**用 `SendMessage` 续同一实例（让它记得刚做了什么）；跨轮才换新。

---

## 单轮协议（The Loop）

1. **worktree 门禁** — 先经 `using-git-worktrees` 进隔离工作树（优先原生 `EnterWorktree`，落项目内 `.claude/worktrees/`），所有产物落在 worktree 内。无 git 默认拒启。
2. **续跑 or 全新** — 有既有 board → 读最新 board + log 重建框架、轮号续号；否则框定问题、基线 triage、生成初始假设集。
3. **用户检查点** — 画好假设板 + 写好当前轮计划后，向用户简报并等 go-ahead（除非已获自主授权）。
4. **诊断（节点1）** — Diagnostician 写测试验证判据 → verdict 文件 → Reviewer 读懂 test、重跑、批准或退回。
5. **修复成本闸门 + 修复（节点2）** — 确诊后先评估修复成本；成本过大上报用户决策，小修则当轮修复 + Reviewer 核验。**修复封顶 3 次**。
6. **回灌重排** — 通读全量 log，复核整盘假设、重新排序、检查跨轮架构信号，重写 board。
7. **终止判定** — 确诊且修复经验证解决 → 完成；否则选下一轮，或打空走升级协议。

---

## Iron Laws（违反字面即违反本 skill 精神）

1. 主 agent **不亲自验证 / 修复** —— 必须下放。
2. **判据必须可测量** —— 不可测只能记「无法判断」，不许假装验证。
3. **无『已确诊』不动修复**。
4. **无 Reviewer 批准，不写 log、不进下一轮**。
5. **每轮重排前必须通读全量日志**。
6. **日志只追加，不改写**。
7. **打空不擅自继续** —— 走升级协议交用户。
8. **每轮 spawn 前过用户检查点**（除非已获自主授权）。
9. **成本过大的修复不得擅自执行** —— 上报用户，即便自主模式也要上报。
10. **每节点通过须提交，修复失败须回退**（目标有 git 时）。
11. **智识诚实，不懂别装**。
12. **必须在隔离 worktree 中运行**（有 git 时）。

---

## 产物：两份可追溯档案（入 `docs/hypo-driven-ps/`，提交进 Git）

- **假设板** `<topic>-board.md` —— 当前状态快照，主 agent 每轮重写。
- **迭代日志** `<topic>-log.md` —— append-only，只追加不改写。
- **verdict 文件** `<topic>-verdicts/R<NN>-H<NN>.md` —— 细粒度证据；主 agent 只看 1–2 行 digest，省上下文。

> 这是可追溯的诊断档案，有意覆盖「临时文件不入库」的惯例。

---

## 使用

在 Claude Code（或兼容的 Agent 环境）中，对一个**卡住的、原因不明的**难题显式调用：

```
/hypo-driven-ps 这个接口在 CI 上偶发 500，本地从不复现，帮我查
```

或自然语言触发：「**摆个假设板**」「**用假设驱动诊断**」「**用 hypo-driven-ps**」。

主 agent 会建好假设板、向你简报排序缘由、等你 go-ahead，再逐轮把验证 / 修复下放给 subagent。

---

## 最小示例

> 场景：某 CLI 在 CI 上偶发 `ENOENT`，本地从不复现。

- **框定**：症状 = CI 偶发 ENOENT；成功判据 = 连续 20 次 CI 绿。初始假设 H1 工作目录差异（中/低）、H2 异步写未等完成就读（中/中）、H3 依赖缓存（低/中）。
- **第 1 轮**：选 H1（成本低、一测切一大片）。Diagnostician 在 CI 打印 `cwd` 与目标路径 → 一致 → **H1 为假**。Reviewer 核对日志来自本次 run → 批准。主 agent 通读 log，把 H2 概率上调为高、重排。
- **第 2 轮**：验 H2。Diagnostician 读 `references/condition-based-waiting.md`，把 `setTimeout` 换成条件轮询复现，确认读发生在写完成前 → **H2 为真**。改为 await 写完成 → 失败测试 → 最小实现 → Reviewer 核验连续绿 → **问题解决**。

---

## 仓库结构

```
hypo-driven-ps/
├── SKILL.md      # 完整 skill 定义（主 agent 读这份执行）
├── DESIGN.md     # 设计决策与取舍
├── PLAN.md       # 初版实施计划
├── PLAN-v2.md    # v2 迭代计划
└── references/   # 按需 Read 的附属文件与模板
    ├── board-template.md           # 假设板模板
    ├── log-template.md             # 迭代日志模板
    ├── verdict-template.md         # verdict（诊断/修复结论）模板
    ├── diagnostician-brief.md      # Diagnostician 职责说明
    ├── reviewer-brief.md           # Reviewer 职责说明
    ├── root-cause-tracing.md       # 反向追踪根因（边界插桩）
    ├── condition-based-waiting.md  # flaky / 竞态 / 时序问题
    ├── condition-based-waiting-example.ts
    ├── defense-in-depth.md         # 修复加固阶段的最窄门禁
    └── find-polluter.sh
```

---

## 复用的 skill

本 skill 复用的所有外部 skill 均来自 **[superpowers](https://github.com/obra/superpowers)**（by [@obra](https://github.com/obra)）。它们在各自的上下文里被 invoke，互不污染主 agent：

| 复用的 skill | 由谁 invoke | 用途 |
|---|---|---|
| `using-git-worktrees` | 主 agent（第 0 步门禁） | 进隔离工作树后再开工，根除跨 session 串台 |
| `test-driven-development` | Diagnostician | 为判据 / 修复先写失败测试再实现 |
| `verification-before-completion` | Reviewer | 核验证据、确认问题真解决 |
| `finishing-a-development-branch` | 主 agent（收尾） | 把 merge / 留 PR / 仅清理的选择交用户 |

> 这些 skill 是行为指引型 prompt，自身不 spawn subagent，在哪个上下文被读到就约束哪个 agent。
>
> 此外，`references/` 下的 `root-cause-tracing.md`、`condition-based-waiting.md`、`defense-in-depth.md` 等取证技法，改编自 superpowers 的 `systematic-debugging`（本 skill 有意**不直接 invoke** 它，以免四阶段流程与本流程冲突）。

---

## License

详见仓库 LICENSE（如未提供，默认保留所有权利，使用前请联系作者）。
