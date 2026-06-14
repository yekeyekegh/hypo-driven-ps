# hypo-driven-ps Skill 实施计划 (Implementation Plan)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把已通过的设计稿(DESIGN.md)落成一个可用的 skill——产出 SKILL.md + 4 个 references 模板/简报 + 从 systematic-debugging 复制 3 份取证技术文件(含 2 个配套依赖)。

**Architecture:** 这是一个「行为指引型 + 多 subagent 编排」的 skill。SKILL.md 是重/门禁式主流程,定义三角色(主 agent 编排、Diagnostician 验证、Reviewer 复核)与单轮循环;references/ 放两份数据模板、两份 subagent 角色简报、三份复制来的取证技术文件。产物全是 markdown / 脚本,无运行时代码,故"测试"= 结构校验(frontmatter 合法、关键章节齐全、内部引用不悬空)。

**Tech Stack:** Markdown(SKILL.md + references)、Claude Code Skill 约定(frontmatter: name/description)、Agent 工具(spawn opus subagent)、复用 superpowers 的 test-driven-development / verification-before-completion。

**前置说明:**
- ⚠️ **执行期迭代**:Tasks 1–5 的交付内容在 inline 执行中经多轮用户反馈迭代(状态三态化、用户检查点、修复成本闸门、subagent 每轮回收、两评审节点 + 交付单、3 次修复上限、测试二分持久化、新增 handoff-template.md)。**已写文件为权威内容**(`SKILL.md` 与 `references/*.md`),完整决策见 `DESIGN.md` 的「§9 决议」累积记录。本 PLAN 下方 Task 1/4/5 的嵌入正文为早期版本,可能滞后——以实际文件为准。
- 落地根目录:`C:\Users\wenjie\.agents\skills\hypo-driven-ps\`(下称 `<ROOT>`)。
- 复制源目录:`C:\Users\wenjie\.agents\skills\.cache\superpowers\skills\systematic-debugging\`(下称 `<SRC>`)。
- 提交步骤:仅当 `<ROOT>` 所在目录处于 Git 版本控制下才执行;当前 `C:\Users\wenjie` 不是 git 仓库,若 `.agents` 也不是,则**跳过所有 `git commit` 步骤**,改为在任务末尾用 `ls`/`grep` 核对文件落盘即可。

---

## File Structure

```
<ROOT>/
  SKILL.md                          # Task 1 — 主流程(重/门禁式)+ 三角色编排
  references/
    board-template.md               # Task 2 — 假设板模板
    log-template.md                 # Task 3 — 迭代日志模板(append-only)
    diagnostician-brief.md          # Task 4 — Diagnostician subagent 角色简报
    reviewer-brief.md               # Task 5 — Reviewer subagent 角色简报
    handoff-template.md             # Task 5 — 两张交付单(诊断/修复)模板
    root-cause-tracing.md           # Task 6 — 复制自 <SRC>
    find-polluter.sh                # Task 6 — root-cause-tracing 的配套脚本
    condition-based-waiting.md      # Task 6 — 复制自 <SRC>
    condition-based-waiting-example.ts # Task 6 — condition-based-waiting 的配套示例
    defense-in-depth.md             # Task 6 — 复制自 <SRC>,配最窄披露门禁(门禁文字在 SKILL.md / diagnostician-brief)
  DESIGN.md                         # 已存在
  PLAN.md                           # 本文件
```

---

## Task 1: SKILL.md(主流程)

**Files:**
- Create: `<ROOT>/SKILL.md`

- [ ] **Step 1: 写 SKILL.md,内容如下(完整,勿删减)**

````markdown
---
name: hypo-driven-ps
description: Hypothesis-driven, structured diagnosis and problem solving. Use ONLY when the user explicitly invokes it (e.g. "/hypo-driven-ps", "用 hypo-driven-ps", "摆个假设板", "用假设驱动诊断") for a hard / stuck / ambiguous problem that has multiple possible root causes — general problem solving (code bugs, performance, architecture choices, research, data anomalies, agent-behavior or environment issues), not limited to debugging. The main agent only orchestrates a hypothesis board and delegates all verification and fixing to Diagnostician + Reviewer subagents.
---

# Hypo-Driven Problem Solving (hypo-driven-ps)

## 概述

通用难题的 hypothesis-driven、结构化诊断与求解。机制:提出多个症结候选(假设)→ 每个配**可测量的确诊判据**(判据成立则该假设晋级为真正症结)→ 按**信息增益 ÷ 验证成本**排序 → 逐轮验证(真/假/无法判断)→ 每轮把 learning 回灌、重排整盘假设 → 直到问题**经验证确实解决**。

**适用**:卡住的、原因不明的、多可能性的难题——代码 bug、性能、架构选型、研究、数据异常、agent 行为/环境问题。
**仅在用户显式调用时启动**,不做自动触发。

## 架构:三角色,主 agent 不亲自下场

- **主 agent(编排者)**:提假设框架、建计划、选每轮验哪个 Hypo、据反馈重排假设板。**不读源码、不跑测试、不亲自取证、不亲自修复**——把吃上下文的活下放给 subagent,自己的上下文只承载假设板的演进。
- **Diagnostician(opus subagent)**:领当前轮,调 `test-driven-development` 写测试、验证判据、交付结论+证据;经 Reviewer 通过后写 log。职责见 `references/diagnostician-brief.md`。
- **Reviewer(opus subagent)**:核验 Diagnostician 的证据,调 `verification-before-completion`,批准或退回整改。职责见 `references/reviewer-brief.md`。

任一时刻**只有一个 Diagnostician、一个 Reviewer,串行进行**。不并发多个假设。

**subagent 生命周期(每轮回收)**:每一轮起**全新**的一对 Diagnostician/Reviewer——它们靠 board + log 两份文件重建所需状态,不靠跨轮记忆,以保证反锚定、上下文有界。**轮内**的「Diagnostician → Reviewer → 整改 → Diagnostician」交接用 `SendMessage` **续同一个实例**(让它记得自己刚做了什么、Reviewer 提了什么),**不要**中途起新的。本轮 log 写完、board 更新后,这一对**一起回收**;下一轮再起全新一对。

## Iron Laws(违反字面即违反本 skill 精神)

1. **主 agent 不亲自验证/修复** —— 必须下放给 Diagnostician/Reviewer。
2. **判据必须可测量** —— 不可测的判据,状态只能是「无法判断」,不得假装验证。
3. **无『已确诊』不动修复** —— 没有判据成立且 Reviewer 认可的确诊,不得进入修复。
4. **无 Reviewer 批准,不写 log、不进下一轮**。
5. **每轮重排前必须通读全量日志** —— 不读历史就重排,等于丢掉迭代价值。
6. **日志只追加,不改写**。
7. **打空不擅自继续** —— 走升级协议交用户决策。
8. **每轮 spawn 前过用户检查点** —— board 更新 + 当前轮计划写好后,除非已获**自主授权**,必须向用户简报并等 go-ahead 再 spawn;获授权则只汇报、不等待。
9. **成本过大的修复不得擅自执行** —— 确诊后须先评估修复成本;命中「成本过大」任一信号(见「修复成本闸门」)时,Diagnostician 不修、带评估上报,**由用户决策**——**即便处于自主模式也要上报**(高影响、难回滚)。

## 产物:两份文件(入 docs/,提交进 Git)

- **假设板** `docs/hypo-driven-ps/<topic>-board.md` —— 当前状态快照,主 agent 每轮重写。结构见 `references/board-template.md`。
- **迭代日志** `docs/hypo-driven-ps/<topic>-log.md` —— append-only,只追加不改写。结构见 `references/log-template.md`。

> 注:此处有意覆盖"临时文件 `_tmp_` 前缀、不入库"的全局规范——这是可追溯的诊断档案,不是临时脚本。

## 优先级规则:信息增益 ÷ 成本

在「待验证」中,优先选「一测就能最大限度缩小不确定性(排除或确认一大片)÷ 验证成本」最高者——即便它不是最高概率。定性判断(高/中/低)即可,不必算公式。

- 易验证 + 高信息量 → 最先做。
- 高概率但极难验证 → 往后排,先用便宜的验证把可能性空间切小。

## 单轮协议(The Loop)

**第 0 步(首次,主 agent 做)**:框定问题(症状 + "怎样算解决"的成功判据)→ 生成初始假设集(每个带 描述/判据/概率/成本)→ 用模板建立 board 与 log 两份文件 → 选第一轮 Hypo,写「当前轮计划」→ **过【用户检查点】(见下节)**。

**每一轮:**

1. **主 agent 编排**:按优先级规则在「待验证」中选本轮 Hypo,在 board 写「当前轮计划」(验哪条判据、建议手段)→ **过【用户检查点】**。
2. **spawn Diagnostician(opus,每轮全新)**:用 Agent 工具,`subagent_type: general-purpose`、`model: opus`、**串行(不 run_in_background)**;prompt = `references/diagnostician-brief.md` 的全文 + 当前轮计划 + 两份文件的绝对路径。它写测试、验证、交付 **结论 + 证据**(此时不写 log)。
3. **spawn Reviewer(opus,每轮全新)**:同样 `subagent_type: general-purpose`、`model: opus`、串行;prompt = `references/reviewer-brief.md` 的全文 + Diagnostician 的结论与证据 + 当前轮计划。它调 `verification-before-completion` 核验 → **批准** 或 **整改意见**。
   - 整改 → 用 `SendMessage` 续**同一个** Diagnostician 按意见重做(不起新实例),再用 `SendMessage` 续**同一个** Reviewer 复核,直到通过。
4. **确诊后:修复成本闸门**(仅当步骤 3 结论为「真」且 Reviewer 已批准验证时走;详见「修复成本闸门」章节。结论为 假 / 无法判断 → 跳过本步):
   - **小修**(局部·低风险·可逆·单轮内·不动架构接口)→ 本轮内由**同一个** Diagnostician 修复(`test-driven-development` 写失败测试 → 最小实现),**同一个** Reviewer 核验(`verification-before-completion`)。
   - **成本过大**(命中任一信号)→ Diagnostician **不修**,带成本评估上报主 agent → 主 agent 告知用户 → **用户决策**:立即修(走上面同样的修+核流程)/ 暂缓先查别的(记状态「已确诊·未修复」)。
5. **Diagnostician 写 log**:经 Reviewer 通过后,把本轮「Diagnostician 轮次记录」(验证结论 + 证据;若确诊则含 修复成本评估 + 修复结果或暂缓)**追加**到 log。
6. **主 agent 回灌重排(硬门禁)**:
   a. **通读全量 log**(不可跳过),提炼跨轮的更深 insight。
   b. 复核**整盘**假设:逐个看 描述/判据/概率/成本/状态 是否要改;增删假设;重新排序。
   c. 把「主 agent 板更新记录」**追加**到 log,并重写 board 快照。
   d. **回收本轮的 Diagnostician 与 Reviewer**(此后不再向它们发消息)。
7. **终止判定**:
   - 本轮 Hypo 已确诊**且修复后经验证确实解决** → 状态「已确诊·已修复」,**完成**。
   - 为真但问题未解(症结不完整/有并存症结)、为假、无法判断、或确诊但用户暂缓修复(已确诊·未修复)→ 回第 1 步,选下一轮 Hypo。
   - 假设板打空 → 进「升级协议」。

> **上下文边界**:主 agent 不读源码、不跑测试、不亲自取证;这些只发生在 Diagnostician/Reviewer 的上下文里。主 agent 只消费它们交付的结论与证据摘要来演进假设板。

## 用户检查点(每轮 spawn 前的人在环)

主 agent 每当**画好/更新好假设板、写下当前轮计划之后、spawn Diagnostician 之前**,向用户**简报**(简略,别长篇):

1. **hypo 概况**:当前有哪些假设、各自状态(待验证/已确诊·已修复/已确诊·未修复/已排除/无法判断/无需验证)。
2. **前三 hypo 的排序缘由**:为什么这三个排在前、本轮为何先验选中的那个(信息增益÷成本)。
3. **(后续轮)上一轮发现 + 对本轮的影响**:上轮验出了什么、据此整盘做了哪些调整。

**默认**:简报后**等待用户的 go-ahead 信号或其它输入**,收到后再 spawn。

**自主授权例外**:当用户已明确表示可自主推进(例如使用 `/loop`、`/goal`,或直接要求主 agent 自行决策),则主 agent **只汇报上述内容、不等 go-ahead**,汇报完直接进入 spawn。

## 修复成本闸门(确诊后)

某 Hypo 经 Reviewer 批准为「真」后,修复前先评估成本。**命中以下任一信号 = 成本过大**:

- 需大范围重构 / 跨多个模块改动。
- 触及架构 / 公共接口 / 数据模型。
- 需数据迁移,或包含不可逆操作。
- 高回归风险,或触及生产安全敏感区(如生产 DB)。
- 需引入新依赖 / 新基础设施。
- 估计工作量远超单轮可完成。

**都不命中 = 小修**(局部·低风险·可逆·单轮内·不动架构接口):本轮内直接修(同一个 Diagnostician 写测试+实现,同一个 Reviewer 核验),修复后经验证解决 → 状态「已确诊·已修复」。

**成本过大**:Diagnostician **不动手修**,把「确诊结论 + 修复方案 + 命中了哪条成本信号 + 工作量/风险估计」上报主 agent;主 agent 转达用户,**由用户决策**:

- **立即修** → 走小修同样的 修+核 流程。
- **暂缓** → 记状态「已确诊·未修复」,继续查 / 先修别的小症结(多个小修叠加可能就缓解了问题)。

> 此闸门**即便在自主模式(`/loop`、`/goal`)下也生效**——成本过大的修复高影响、难回滚,必须回到用户决策。

## 升级协议(假设板打空)

当「待验证」全部清空(整盘只剩 已排除 / 无法判断 / 已确诊·未修复),问题仍未解决:主 agent 停下。**若存在「已确诊·未修复」项**,先就这些大修项请用户决策是否执行(它们路径明确,只是成本大被暂缓);若无此类项、或用户对其仍选暂缓而问题未解,再**按 2 → 1 → 3 顺序**呈报用户:

1. **(无法判断项)** 它们是否值得攻克?把它变可测的成本(加探针/instrumentation/造可复现样本)?
2. **(新方向)** 还有哪些新假设/新方向值得探索?
3. **(质疑框定)** 问题框定是否错了 / 架构是否根本不对?

→ 主 agent 综合三项 + 自身判断,**给出下一步推荐**。但**最终决策权交用户**,不擅自继续。

## 复用的 skill(由 subagent 在各自上下文里 invoke)

- **Diagnostician** → `test-driven-development`(为判据/修复写失败测试再实现)。
- **Reviewer** → `verification-before-completion`(核验证据、确认问题真解决)。

> 这些 skill 自身**不 spawn subagent、不自动链式调用**——它们是行为指引型 prompt,在哪个 agent 上下文里被读到就约束哪个 agent。

## 取证参考文件(同目录 `references/`,按需 Read,命中才用)

- `root-cause-tracing.md` —— 深栈 / 坏数据来源不明时,反向追踪到源头。
- `condition-based-waiting.md` —— 验证对象是 flaky / 竞态 / 时序问题(测试里有 setTimeout/sleep)。
- `defense-in-depth.md` —— **最窄门禁**:仅当 ① 已确诊根因是「代码逻辑缺陷,由非法/畸形数据跨层传播引起」**且** ② 处于修复后的加固阶段,才 Read 并应用。**明确排除**:agent 行为/环境类、非代码类、研究/选型类问题——即便被当作 debugging 在跑,也不得套用多层校验。

> 本 skill 刻意**不直接 invoke `systematic-debugging`**(其 Phase 1–3 与本流程重叠会冲突)。上面三份文件是从它复制来的独立参考,被动 Read,不会触发它的四阶段。

## Red Flags —— 出现就停,回到流程

- 主 agent 自己去读代码 / 跑测试 / 取证了。
- 跳过 Diagnostician 或 Reviewer 直接下结论 / 写 log。
- 判据说不清"怎么测",却没归入「无法判断」。
- 没看着测试失败,就声称判据成立。
- 只改了当前假设,没重排整盘 / 没读全量日志。
- 日志被改写,而非追加。
- 没确诊就开始修;或"顺手"改了无关的东西。
- 打空了还自己硬往下凑,没交用户决策。
- 对 agent 行为/环境类问题套用 defense-in-depth 多层校验。
- 跨轮还续用同一对 subagent(没每轮回收);或轮内整改时另起新实例(丢了轮内连续性)。
- board 画完/更新完没向用户简报就直接 spawn(且未获自主授权);或已获自主授权却还在傻等 go-ahead。
- 确诊后没评估修复成本就开修;或成本过大却擅自动了大修(需重构/动架构/数据迁移/不可逆),没上报用户决策。

## 常见借口 vs 现实

| 借口 | 现实 |
|---|---|
| "这问题简单,主 agent 自己看一眼就行" | 自己下场就丢了省上下文的意义,且无 Reviewer 把关。仍要下放。 |
| "判据测不了,但我觉得八成是它" | 觉得 ≠ 确诊。测不了就是「无法判断」,别假装验证。 |
| "Diagnostician 说对了,直接写 log 吧" | 无 Reviewer 批准不写 log。证据要被独立核验。 |
| "只更新这条假设就够了" | 每轮必须读全量日志、重排整盘,否则迭代无增益。 |
| "先改改看,不行再说" | 无确诊不动修复。乱改反而污染判断。 |
| "打空了我再硬想几个继续试" | 打空走升级协议,决策交用户。 |
| "确诊了就赶紧把它修好" | 先评估成本。成本过大要上报用户:也许先修几个小的就缓解,或大修需用户拍板。 |

## 最小示例(worked example)

> 场景:某 CLI 在 CI 上偶发 `ENOENT: no such file or directory`,本地从不复现。用户显式调用 hypo-driven-ps。

- **第 0 步(主 agent)**:框定——症状=CI 偶发 ENOENT;成功判据=连续 20 次 CI 绿。初始假设:
  - H1 工作目录在 CI 与本地不同(概率中,成本低)
  - H2 异步写文件未等完成就读(flaky,概率中,成本中)
  - H3 依赖缓存导致文件缺失(概率低,成本中)
- **第 1 轮**:按 信息增益÷成本,选 H1(成本低、一测能切一大片)。主 agent 写当前轮计划:判据=「在 CI 打印 `process.cwd()` 与目标路径,若二者不一致则 H1 成立」。
  - Diagnostician(opus):调 TDD 写一个会失败的断言测试,在 CI 注入打印,取证。结论:cwd 一致 → **H1 为假**,证据=两段 CI 日志。
  - Reviewer(opus):调 verification-before-completion,核对日志确实来自本次 CI run、退出码一致 → **批准**。
  - Diagnostician 写 log。
  - 主 agent:通读 log,insight=「路径正确,问题更可能在时序」→ 把 H2 概率上调为高,重排,下一轮验 H2。
- **第 2 轮**:验 H2。Diagnostician Read `references/condition-based-waiting.md`,把测试里的 `setTimeout` 换成条件轮询复现,确认读发生在写完成前 → **H2 为真**。修复:改为 await 写完成;再写失败测试→最小实现→Reviewer 核验连续绿 → **问题解决,完成**。
````

- [ ] **Step 2: 校验 frontmatter 与关键章节齐全**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
head -3 "$ROOT/SKILL.md"
grep -c -E "^## (概述|架构|Iron Laws|产物|优先级规则|单轮协议|升级协议|复用的 skill|取证参考文件|Red Flags|常见借口|最小示例)" "$ROOT/SKILL.md"
```
Expected: 第一行是 `---`,第二行 `name: hypo-driven-ps`;grep 计数 = **11**(11 个二级章节全在)。

- [ ] **Step 3: 提交(若在 Git 下)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add SKILL.md && git commit -m "feat(hypo-driven-ps): add main SKILL.md with gated 3-role loop"
```
(若该目录非 Git 仓库,跳过本步。)

---

## Task 2: references/board-template.md(假设板模板)

**Files:**
- Create: `<ROOT>/references/board-template.md`

- [ ] **Step 1: 写文件,内容如下**

````markdown
# 假设板:<topic>

> 维护者:主 agent(编排者)。每轮重写为最新快照。
> 配套日志:`<topic>-log.md`(append-only)。

## 1. 问题框定

- **症状**:<观察到的现象,尽量是可复现的客观事实>
- **成功判据(怎样算解决)**:<可验证的、具体的完成条件>

## 2. 假设集合

| ID | 描述(怀疑的症结) | 确诊判据(可测量) | 可能性 | 验证难度 | 优先级 | 状态 |
|----|------------------|------------------|:------:|:--------:|:------:|------|
| H1 | <症结候选陈述>   | 当 <判据> 成立则确诊 | 高/中/低 | 高/中/低 | <信息增益÷成本 排名> | 待验证 |
| H2 | …                | …                | …      | …        | …      | 待验证 |

> 列义:可能性=先验概率;验证难度=验证成本。向用户简报时两维**全标注**:`（可能性:高 · 验证难度:中）`,不用裸 `(高/中)`。

> 状态取值:待验证 / 已确诊·已修复 / 已确诊·未启动修复 / 已确诊·修复失败 / 已排除 / 无法判断 / 无需验证
> 语义:待验证=尚未检验;已确诊·已修复=判据成立且修复后经验证解决;已确诊·未启动修复=判据成立确为症结,但修复成本过大、经用户决策暂缓(未动手);已确诊·修复失败=判据成立,修复连试 3 次仍未解决(learning 已回灌);已排除=判据被证伪;无法判断=判据当前不可测量;无需验证=被已确诊项盖过或经重排已无关。

## 3. 当前轮计划(第 N 轮)

- **本轮 Hypo**:H?
- **选它的理由**:信息增益 ÷ 成本 最高,因为 <一测能排除/确认一大片 而成本低>
- **要验证的判据**:<具体哪一条>
- **建议手段**:<怎么测>
  - 若深栈 / 坏数据来源不明 → 提示 Diagnostician 读 `root-cause-tracing.md`
  - 若 flaky / 竞态 / 时序 → 提示读 `condition-based-waiting.md`
````

- [ ] **Step 2: 校验**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
grep -c -E "^## (1\. 问题框定|2\. 假设集合|3\. 当前轮计划)" "$ROOT/references/board-template.md"
```
Expected: **3**。

- [ ] **Step 3: 提交(若在 Git 下)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/board-template.md && git commit -m "feat(hypo-driven-ps): add board template"
```

---

## Task 3: references/log-template.md(迭代日志模板)

**Files:**
- Create: `<ROOT>/references/log-template.md`

- [ ] **Step 1: 写文件,内容如下**

````markdown
# 迭代日志:<topic>  (append-only,只追加不改写)

<!--
  规则:本文件只能在末尾追加新条目,禁止修改/删除既有条目。
  每轮产生两类条目:
    A. Diagnostician 轮次记录(由 Diagnostician 在 Reviewer 批准后写)
    B. 主 agent 板更新记录(由主 agent 在回灌重排后写)
-->

---
## 第 N 轮 — A. Diagnostician 轮次记录

- **攻击的 Hypo**:H? — <描述>
- **做了什么(action)**:<执行了哪些步骤>
- **测了什么**:对照判据 <…>,手段 <命令 / 插桩 / 复现脚本>
- **结果**:判据成立(真)/ 判据证伪(假)/ 无法判断
  - 若「无法判断」:哪条判据测不了、为何
- **(若确诊)修复成本评估**:小修 / 成本过大(命中信号:<哪条>)
- **(若确诊)修复结果**:已修复并经验证解决(已确诊·已修复)/ 用户决策暂缓(已确诊·未启动修复)/ 修复失败(连试 N≤3 次未果,已确诊·修复失败)
- **(若尝试修复)修复尝试次数**:第 N 次 / 共失败 N 次;若失败,各次试了什么、为何失败(learning)
- **测试清单**:<节点1 诊断 test + 节点2 修复/回归 test 的路径(探针存 `<topic>-probes/`,修复回归 test 进正式套件)>
- **证据摘要**:<命令 / 完整输出要点 / 退出码 / 复现步骤>
- **提交(有 git)**:节点1 diag commit <hash/N/A> · 节点2 fix commit <hash/N/A> · 若回退:fix-failed commit + patch 路径
- **Reviewer 结论(节点1 / 节点2)**:批准通过(若曾整改:整改了什么,第几次通过)

## 第 N 轮 — B. 主 agent 板更新记录

- **通读全量日志得到的跨轮 insight**:<把本轮与历史轮连起来看到的更深结论>
- **对整盘假设的改动**:新增 H? / 排除 H? / 改判据 / 改概率·成本 / 改状态 / 重排
- **下一轮计划**:验 H?,理由 <…>
---
````

- [ ] **Step 2: 校验**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
grep -c -E "Diagnostician 轮次记录|主 agent 板更新记录" "$ROOT/references/log-template.md"
```
Expected: **2**。

- [ ] **Step 3: 提交(若在 Git 下)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/log-template.md && git commit -m "feat(hypo-driven-ps): add append-only log template"
```

---

## Task 4: references/diagnostician-brief.md(Diagnostician 角色简报)

**Files:**
- Create: `<ROOT>/references/diagnostician-brief.md`

- [ ] **Step 1: 写文件,内容如下**

````markdown
# Diagnostician 角色简报(opus subagent)

你是 hypo-driven-ps 流程里的 **Diagnostician**。任一时刻只有你一个在领这一轮,串行进行。

## 你领到的输入

- 假设板「当前轮计划」:要验证的 Hypo、它的确诊判据、建议手段。
- 两份文件路径:`<topic>-board.md`(只读参考)与 `<topic>-log.md`(通过后追加)。

> 你是**本轮全新起**的实例:不假设记得之前轮的工作;需要历史背景就读 log。轮内若收到 Reviewer 整改意见,你会被续聊继续(那时你记得自己刚做了什么)。

## 你必须做的(铁律)

1. **先写测试,再验证**:调用 `test-driven-development` skill,为待验证的判据写出会失败的测试,运行并**看着它正确地失败(红)**,再据此判定。没看着测试失败,不得声称判据可测/成立。
2. **判定**:严格对照该判据,给出三选一结论:
   - **真** = 判据成立,确为症结。
   - **假** = 判据被证伪。
   - **无法判断** = 判据当前不可测量(写明哪条、为何)。
3. **交付**:把 **结论 + 证据**(命令、完整输出、退出码、复现步骤)交回主 agent。**此时不要写 log**——必须先过 Reviewer。
4. **(若结论为「真」且 Reviewer 已批准验证)修复成本闸门**:对照下面的成本信号清单评估修复成本。
   - **成本过大**(命中任一)→ **不要动手修**。把「确诊结论 + 修复方案 + 命中了哪条成本信号 + 工作量/风险估计」上报主 agent,由用户决策。用户若暂缓 → 本轮到此(写 log 时记「用户决策暂缓」)。
   - **小修(都不命中)或 用户决策立即修** → 先写失败的复现测试,再写最小实现使其通过(绿),不夹带无关改动;交 Reviewer 核验修复。
   - **成本信号清单(任一命中 = 成本过大)**:需大范围重构 / 跨多模块;触及架构·公共接口·数据模型;需数据迁移或不可逆操作;高回归风险 / 触及生产安全敏感区;需引入新依赖 / 基建;工作量远超单轮。
5. **写 log**:Reviewer 全部通过后,把本轮「Diagnostician 轮次记录」按 `log-template.md` 的 A 条目格式**追加**到 log(只追加,不改既有内容;若确诊则含修复成本评估 + 修复结果/暂缓)。
   - 收到**整改意见** → 按意见重做,重新交付给 Reviewer,直到通过。

## 取证技术(按需 Read,命中才用)

- 深栈 / 坏数据来源不明 → Read `root-cause-tracing.md`(反向追踪 + 插桩 + `find-polluter.sh`)。
- flaky / 竞态 / 时序(测试里有 setTimeout/sleep)→ Read `condition-based-waiting.md`(用条件轮询替代死等)。
- **defense-in-depth 最窄门禁**:仅当 ① 已确诊根因是「代码逻辑缺陷,由非法/畸形数据跨层传播引起」**且** ② 处于修复后的加固阶段,才 Read `defense-in-depth.md` 并加多层校验。**明确排除**:agent 行为/环境类、非代码类、研究/选型类问题——即便被当作 debugging,也不得套用。

## 你不做

- 不更新假设板(那是主 agent 的事)。
- 不擅自扩大范围:只验当前轮的判据 / 只修当前确诊的根因,禁止"顺手改"。
- 不在没有 Reviewer 批准时写 log。
````

- [ ] **Step 2: 校验**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
grep -c -E "test-driven-development|defense-in-depth 最窄门禁|不要写 log" "$ROOT/references/diagnostician-brief.md"
```
Expected: **3**(三项关键约束都在)。

- [ ] **Step 3: 提交(若在 Git 下)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/diagnostician-brief.md && git commit -m "feat(hypo-driven-ps): add diagnostician brief"
```

---

## Task 5: references/reviewer-brief.md + references/handoff-template.md

> ⚠️ 执行期已迭代,实际内容以写入文件为准(见顶部「执行期迭代」说明)。本 Task 列出两文件的**必备要点**与校验。

**Files:**
- Create: `<ROOT>/references/reviewer-brief.md`
- Create: `<ROOT>/references/handoff-template.md`

- [ ] **Step 1: 写 reviewer-brief.md** —— 必备要点:
  - 两个评审节点:**节点1 核诊断 / 节点2 核修复**。
  - **铁律:必须读懂 test**——读懂支撑本节点结论的那组 test、查对错与完整性、重跑;调 `verification-before-completion`,无当场新鲜证据**不予通过**。
  - 节点1:结论是否被 test+证据支撑;test 有漏洞/不完整 → 打回(**整改**)。
  - 节点2:复现测试 修复前红→后绿;**额外重跑回归/既有套件**;问题是否真不再复现。判修复未解决/回归 → 计入 3 次尝试;判 test 缺陷 → 整改(**不计入**)。
  - 产出:批准 / 整改意见。不放水;不替做、不更新 board、不写 log。

- [ ] **Step 2: 写 handoff-template.md** —— 两张交付单:
  - **诊断交付单(节点1)**:轮次 / Hypo / 判据 / 结论(真·假·无法判断)/ 测试清单(路径·测什么·命令·红绿)/ **完整性论证** / 证据。
  - **修复交付单(节点2)**:轮次 + 第 K 次尝试(K≤3)/ 确诊 Hypo / 成本评估 / 修复方案+改了哪些文件 / 测试清单(复现·回归,进正式套件)/ 回归核查 / 证据 / (若失败)失败说明。

- [ ] **Step 3: 校验**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
echo "reviewer:"; grep -c -E "verification-before-completion|不予通过|整改意见" "$ROOT/references/reviewer-brief.md"   # 应=3
grep -q "节点1" "$ROOT/references/reviewer-brief.md" && grep -q "节点2" "$ROOT/references/reviewer-brief.md" && echo "两节点 OK"
echo "handoff:"; grep -c -E "诊断交付单|修复交付单" "$ROOT/references/handoff-template.md"   # 应=2
```
Expected: reviewer=3、两节点 OK、handoff=2。

- [ ] **Step 4: 提交(若在 Git 下)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/reviewer-brief.md references/handoff-template.md && git commit -m "feat(hypo-driven-ps): add reviewer brief + handoff templates"
```

---

## Task 6: 复制三份取证技术文件 + 两个配套依赖

**Files:**
- Create: `<ROOT>/references/root-cause-tracing.md`(copy from `<SRC>`)
- Create: `<ROOT>/references/find-polluter.sh`(copy,root-cause-tracing 内部引用)
- Create: `<ROOT>/references/condition-based-waiting.md`(copy from `<SRC>`)
- Create: `<ROOT>/references/condition-based-waiting-example.ts`(copy,condition-based-waiting 内部引用)
- Create: `<ROOT>/references/defense-in-depth.md`(copy from `<SRC>`)

- [ ] **Step 1: 执行复制**

Run:
```bash
SRC="$HOME/.agents/skills/.cache/superpowers/skills/systematic-debugging"
DST="$HOME/.agents/skills/hypo-driven-ps/references"
cp "$SRC/root-cause-tracing.md"            "$DST/root-cause-tracing.md"
cp "$SRC/find-polluter.sh"                 "$DST/find-polluter.sh"
cp "$SRC/condition-based-waiting.md"       "$DST/condition-based-waiting.md"
cp "$SRC/condition-based-waiting-example.ts" "$DST/condition-based-waiting-example.ts"
cp "$SRC/defense-in-depth.md"              "$DST/defense-in-depth.md"
```

- [ ] **Step 2: 校验 5 个文件都到位且非空,且内部引用不悬空**

Run:
```bash
DST="$HOME/.agents/skills/hypo-driven-ps/references"
for f in root-cause-tracing.md find-polluter.sh condition-based-waiting.md condition-based-waiting-example.ts defense-in-depth.md; do
  if [ -s "$DST/$f" ]; then echo "OK  $f"; else echo "MISSING/EMPTY  $f"; fi
done
echo "--- 引用自检(应能在同目录找到被引用的配套文件)---"
grep -l "find-polluter.sh" "$DST/root-cause-tracing.md" && ls "$DST/find-polluter.sh" >/dev/null && echo "find-polluter ref OK"
grep -l "condition-based-waiting-example" "$DST/condition-based-waiting.md" && ls "$DST/condition-based-waiting-example.ts" >/dev/null && echo "example ref OK"
```
Expected: 5 行 `OK`;两条 `ref OK`(若某 .md 实际未引用配套文件,grep 无输出也可接受——配套文件已在即可)。

- [ ] **Step 3: 提交(若在 Git 下)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/root-cause-tracing.md references/find-polluter.sh references/condition-based-waiting.md references/condition-based-waiting-example.ts references/defense-in-depth.md && git commit -m "chore(hypo-driven-ps): vendor root-cause-tracing / condition-based-waiting / defense-in-depth from systematic-debugging"
```

---

## Task 7: 全量结构校验(收尾)

**Files:** 只读校验,不改文件。

- [ ] **Step 1: 列目录,确认最终结构完整**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
find "$ROOT" -type f | sed "s#$ROOT/##" | sort
```
Expected(至少包含):
```
DESIGN.md
PLAN.md
SKILL.md
references/board-template.md
references/condition-based-waiting-example.ts
references/condition-based-waiting.md
references/defense-in-depth.md
references/diagnostician-brief.md
references/find-polluter.sh
references/handoff-template.md
references/log-template.md
references/reviewer-brief.md
references/root-cause-tracing.md
```

- [ ] **Step 2: 校验 SKILL.md 里指向的 references 文件全部真实存在(引用不悬空)**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
for f in board-template.md log-template.md diagnostician-brief.md reviewer-brief.md handoff-template.md root-cause-tracing.md condition-based-waiting.md defense-in-depth.md; do
  grep -q "$f" "$ROOT/SKILL.md" && ([ -f "$ROOT/references/$f" ] && echo "OK ref: $f" || echo "DANGLING ref: $f")
done
```
Expected: 8 行全部 `OK ref:`,无 `DANGLING`。

- [ ] **Step 3: 确认 skill 可被发现(frontmatter name 唯一且合法)**

Run:
```bash
head -1 "$HOME/.agents/skills/hypo-driven-ps/SKILL.md"   # 应为 ---
grep -m1 "^name:" "$HOME/.agents/skills/hypo-driven-ps/SKILL.md"  # 应为 name: hypo-driven-ps
```
Expected: 第一行 `---`;第二条输出 `name: hypo-driven-ps`。

- [ ] **Step 4: (可选)新会话里验证触发**:在一个新对话说"用 hypo-driven-ps 帮我诊断 X",确认该 skill 被列出/可被 Skill 工具调用。此步为人工验证,非脚本。

---

## Self-Review(规划者自检结果)

- **Spec 覆盖**:DESIGN 的 §2 范围/触发 → SKILL frontmatter+概述;§3 两份文件 → Task 2/3 模板 + SKILL「产物」;§4 三角色循环 → SKILL「架构/单轮协议」+ Task 4/5 简报;§5 升级协议 → SKILL「升级协议」;§6 重/门禁式 → SKILL Iron Laws/Red Flags/借口表;§7 systematic-debugging 处理 + 3 份复制(defense-in-depth 窄门禁)→ Task 6 + SKILL「取证参考文件」+ diagnostician-brief 门禁。全部有任务承接,无缺口。
- **占位符扫描**:无 TBD/TODO;模板里的 `<...>` 是模板占位符(预期保留,供使用时填),非计划占位。
- **命名一致性**:三角色名(主 agent / Diagnostician / Reviewer)、文件名(`<topic>-board.md` / `<topic>-log.md` / `<topic>-probes/`)、状态取值(待验证/已确诊·已修复/已确诊·未启动修复/已确诊·修复失败/已排除/无法判断/无需验证)、两评审节点(节点1/节点2)、交付单(诊断/修复)、复用 skill 名(test-driven-development / verification-before-completion)在 SKILL 与各 brief / 模板中一致。
