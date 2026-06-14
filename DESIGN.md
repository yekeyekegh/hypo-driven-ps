# hypo-driven-ps — 设计稿 (Design Spec)

> 日期: 2026-06-13
> 状态: 已通过 brainstorm,待用户复核 → 进入 writing-plans
> 落地路径: `C:\Users\wenjie\.agents\skills\hypo-driven-ps\`

## 1. 目标与一句话定位

一个**用户显式调用**的、**hypothesis-driven & 结构化**的通用问题诊断与求解 skill。
当 agent 遇到卡住的、原因不明的、有多种可能性的难题时,用一套"广义差分诊断"流程:
提出多个症结候选(假设)→ 给每个假设配显式**可测量的确诊判据** → 按"信息增益 ÷ 验证成本"排序 →
逐轮验证(真/假/无法判断)→ 每轮把 learning 回灌、重排整盘假设 → 直到问题**经验证确实解决**。

**架构特点:多 subagent 协作,主 agent 只编排不亲自验证**(见 §4),目的是把读代码/跑测试/取证这些
吃上下文的活下放给 subagent,主 agent 的上下文只承载假设板的演进。

## 2. 范围 (Scope)

- **通用问题求解**:代码 bug、性能、架构选型、研究、数据异常、流程问题等皆适用。
- **拥有完整闭环**:假设 → 诊断 → 修复 → 验证。终点 = 问题经验证确实解决(不止于找到症结)。
- **触发**:仅用户显式调用(`/hypo-driven-ps` 或口头点名)。不做激进自动触发。

### 非目标 (YAGNI)
- **不做并行 hypo 验证**:循环本质串行——每轮必须靠上一轮 learning 重排整盘。**任一时刻只有一个
  Diagnostician 在领一轮**,不并发多个。
- **不做复杂打分公式**:概率/成本一律定性 高/中/低 即可。

## 3. 核心产物:两份文件

放在 `docs/hypo-driven-ps/<topic>-*.md`,**提交进 Git**,作为可追溯的正式诊断档案
(此处有意覆盖全局 `_tmp_` 临时文件规范——因为它是诊断记录而非临时脚本)。

### 3.1 假设板 `docs/hypo-driven-ps/<topic>-board.md` — 当前状态快照(每轮重写,主 agent 维护)
包含三块:
1. **问题框定**:观察到的症状 + "解决"的成功判据(怎样算解决)。
2. **假设集合表**(每个假设一行):

   | 字段 | 含义 |
   |---|---|
   | ID | H1, H2, … |
   | 描述 | 怀疑的症结陈述 |
   | **确诊判据** | 显式、**可测量**的条件;一旦成立,此假设即真正症结(假设→症结的"晋级条件") |
   | 先验概率 | 高/中/低 |
   | 验证成本 | 低/中/高 |
   | 优先级 | 由"信息增益 ÷ 成本"定性导出 |
   | 状态 | 待验证 / 已确诊 / 已排除 / 无法判断 / 无需验证 |

   状态语义:待验证=尚未检验;已确诊=判据成立确为症结;已排除=判据被证伪;
   无法判断=判据当前不可测量;无需验证=被已确诊项盖过或经重排已无关。
3. **当前轮计划**:本轮交给 Diagnostician 验哪个 Hypo · 为何选它(性价比理由)· 验证要点(对照哪条判据、建议手段)。

### 3.2 迭代日志 `docs/hypo-driven-ps/<topic>-log.md` — append-only,只追加不改写
两类来源都往这里追加(谁写、写什么见 §4):
- **Diagnostician 的轮次记录**:轮次 N、攻击的 Hypo(ID+描述)、做了什么(action)、测了什么
  (对照的判据+手段)、结果(成功/失败/无法判断,具体哪条判据测不了)、提交的证据摘要、Reviewer 结论。
- **主 agent 的板更新记录**:本轮据 learning 通读全量日志后得出的跨轮 insight、对整盘假设做了哪些
  增删/改判据/改状态/重排,以及下一轮计划。

## 4. 核心循环:多 subagent 协作 (The Loop)

**所有 subagent 用 opus 最高等级**——因为全程没有清晰的 spec/plan 约束,需要最强推理能力。
**任一时刻只有一个 Diagnostician、一个 Reviewer,串行进行。**

**第 0 步(首次,主 agent)框定问题**:写出症状与成功判据 → 生成初始假设集(每个带 描述/判据/概率/成本)
→ 建立两份文件 → 选出第一轮要验的 Hypo,写当前轮计划。

**每一轮:**
1. **主 agent — 编排**:在"待验证"中按"信息增益 ÷ 成本"性价比选出本轮 Hypo,在假设板写下当前轮计划
   (验哪条判据、建议手段)。
   - 冲突仲裁:优先"一测就能最大限度缩小不确定性(排除或确认一大片)÷ 验证成本"最高者,哪怕非最高概率。定性即可。
2. **Diagnostician(opus subagent)— 验证**:领本轮计划。
   - 先调 `test-driven-development`:为待验的判据写出会失败的测试(红),看着它正确失败。
   - 执行验证,得出该判据 真 / 假 / 无法判断。
   - 交付:结论 + **证据**(命令、输出、退出码、复现步骤等)。**此时尚不写 log。**
3. **Reviewer(opus subagent)— 复核**:核对 Diagnostician 提交的证据是否真支撑其结论。
   - 调 `verification-before-completion`:要求有当场新鲜的验证证据,无证据不予通过。
   - 产出:**批准通过** 或 **整改意见**。
   - 若整改 → 回第 2 步,Diagnostician 按意见重做,Reviewer 再核,直到通过。
4. **Diagnostician — 写 log**:通过后,把本轮轮次记录(action/测了什么/结果/证据摘要/Reviewer 结论)
   **追加**到迭代日志。
5. **主 agent — 回灌重排(硬门禁)**:据 Diagnostician 的反馈与 learning 系统更新假设板:
   a. **通读全量历史日志**(此步不可跳过),提炼跨轮的更深 insight。
   b. 据此复核**整盘**假设:逐个看 描述/判据/概率/成本/状态 是否要改;增删假设;重新排序。
   c. 把"板更新记录"(insight + 改动 + 下一轮计划)追加到迭代日志。
6. **终止判定**:
   - 本轮 Hypo 已确诊且**修复后经验证确实解决**(修复同样走 Diagnostician 写测试 + Reviewer 核验)→ **完成**。
   - 为真但问题未解(症结不完整/有并存症结)、为假、或无法判断 → 回第 1 步,由主 agent 选下一轮 Hypo,
     交下一个 Diagnostician 领。
   - 假设板打空 → 进 §5 升级协议。

> 上下文边界:主 agent **不读源码、不跑测试、不亲自取证**,这些只在 Diagnostician/Reviewer 的上下文里发生;
> 主 agent 只消费它们交付的结论与证据摘要来演进假设板。

## 5. 打空升级协议 (Exhaustion → Escalate)

当"待验证"全部清空(整盘只剩 已排除 + 无法判断),问题仍未解决:
**主 agent 停下,按 2 → 1 → 3 顺序**呈报用户:
1. (无法判断项)它们是否值得攻克?把它变可测的成本(加探针/instrumentation/造可复现样本)?
2. (新方向)还有哪些新假设/新方向值得探索?
3. (质疑框定)问题框定是否错了 / 架构是否根本不对?

→ 主 agent 综合三项 + 自身判断,**给出下一步推荐**。但**最终决策权交用户**,不擅自继续。

## 6. 写法刚性:重 / 门禁式

SKILL.md 按 `systematic-debugging` 同量级写,包含:
- **Iron Law**:如「主 agent 不得亲自验证/解题,必须下放给 Diagnostician」「无 Reviewer 批准不得写 log」
  「无『已确诊』假设不得动手修复」「每轮重排前必须先通读全量日志」「判据必须可测量,否则归入无法判断」。
- 强制两份文件产物。
- **Red Flags 清单**:如"主 agent 自己去读代码验证了""跳过 Reviewer 直接写 log""凭直觉先改改看"
  "只更新当前假设没重排整盘""日志改写而非追加"等。
- **常见借口对照表**(借口 vs 现实)。
- 升级协议作为门禁(打空时不擅自继续)。

## 7. 复用关系与对 systematic-debugging 的处理

| 对象 | 处理方式 | 用途 |
|---|---|---|
| `systematic-debugging` | **不直接 invoke**(其 Phase 1–3 与本设计重叠,直接调会冲突);**读取参考、内容自己重写**进 SKILL.md | 借鉴四阶段/铁律/红旗的写法 |
| `root-cause-tracing.md` | **复制**进本 skill `references/`(直接拿来用) | Diagnostician 取证时反向追深栈根因 |
| `condition-based-waiting.md` | **复制**进本 skill `references/`(直接拿来用) | 验证对象是 flaky/竞态/时序问题时 |
| `defense-in-depth.md` | **复制**进本 skill `references/`,但配**最窄披露门禁** | 见下方门禁——仅代码逻辑 bug 的加固阶段 |

**defense-in-depth 披露门禁**(写进 SKILL.md / diagnostician-brief):
> 仅当 ① 已确诊根因是「代码逻辑缺陷,由非法/畸形数据跨层传播引起」**且** ② 处于修复后的加固阶段时,
> 才披露并应用 defense-in-depth。**明确排除**:agent 行为/环境类、非代码类、研究/选型类问题——
> 即便它们被当作"debugging"在跑,也不得套用多层校验。
| `test-driven-development` | **Diagnostician invoke**(软引用) | 为待验判据/为修复写失败测试 |
| `verification-before-completion` | **Reviewer invoke**(软引用) | 核验证据/确认问题真解决 |

> 注:这三个被复用的 superpowers skill **自身都不 spawn subagent、不自动链式调用**——它们是行为指引型
> prompt,在哪个 agent 的上下文里被读到就约束哪个 agent。本设计是由我们自己的 Diagnostician/Reviewer
> 在各自上下文里 invoke 它们。

## 8. Skill 文件结构(预案,交 writing-plans 细化)

```
hypo-driven-ps/
  SKILL.md                    # 主流程(重/门禁式)+ 三角色编排
  references/
    board-template.md         # 假设板模板
    log-template.md           # 迭代日志模板
    diagnostician-brief.md    # Diagnostician subagent 的角色 prompt 模板
    reviewer-brief.md         # Reviewer subagent 的角色 prompt 模板
    root-cause-tracing.md     # 复制自 systematic-debugging
    condition-based-waiting.md # 复制自 systematic-debugging
    defense-in-depth.md       # 复制自 systematic-debugging,配最窄披露门禁(见 §7)
  DESIGN.md                   # 本设计稿
```

## 9. 待 writing-plans 阶段决定的细节(已在 PLAN.md 全部闭环)
- SKILL.md 各 section 的精确措辞与 Red Flags / 借口表条目。
- 主 agent 用什么 subagent_type + model=opus 来 spawn Diagnostician / Reviewer(Agent 工具的具体调用形态)。
- Diagnostician / Reviewer brief 模板的字段(领到什么、交付什么、证据格式)。
- 两个数据模板(board / log)的确切字段排版。
- description 字段的触发措辞(显式调用导向)。
- 是否需要一个最小示例(worked example)。

> 决议(2026-06-13):以上全部在 PLAN.md 落定。subagent spawn 形态 = `general-purpose` + `model: opus`,串行;
> worked example 保留(SKILL.md 末尾,CI ENOENT 两轮示例);其余措辞/字段/触发词均已在 PLAN 的 Task 1–5 全文嵌入。
>
> 补充决议(执行 Task 1 期间):**subagent 生命周期 = 每轮回收 + 轮内整改保持同一实例**。
> 每轮起全新一对 D/R(靠 board+log 重建状态,反锚定、上下文有界);轮内 D→R→整改→D 用 SendMessage
> 续同一实例;本轮 log 写完、board 更新后一起回收,下一轮全新一对。已写入 SKILL.md(架构/单轮协议/Red Flags)。
>
> 补充决议(执行 Task 1 期间):**每轮 spawn 前的用户检查点(人在环)**。主 agent 画好/更新好 board 并写下
> 当前轮计划后、spawn D 之前,默认须向用户简报(hypo 概况 / 前三排序缘由 / 后续轮的上轮发现及影响)并等
> go-ahead;仅当用户已授予自主权(`/loop`、`/goal`、明示自行决策)时改为只汇报不等待。
> 已写入 SKILL.md(Iron Law 8 / 单轮协议第0步+第1步 / 新增「用户检查点」章节 / Red Flags)。
>
> 补充决议(执行 Task 2 期间):**状态细分 + 修复成本闸门**。状态「已确诊」拆为「已确诊·已修复」「已确诊·未修复」;
> 全集 = 待验证 / 已确诊·已修复 / 已确诊·未修复 / 已排除 / 无法判断 / 无需验证。确诊后修复前评估成本,
> **多信号清单任一命中 = 成本过大**(大范围重构·跨模块 / 触及架构·接口·数据模型 / 数据迁移·不可逆 / 高回归风险·生产安全 /
> 新依赖·基建 / 工作量远超单轮)→ Diagnostician 不修、带评估上报 → 用户决策(立即修 / 暂缓先修小的);
> 都不命中 = 小修,本轮内直接修。**此闸门即便自主模式也生效**。已写入 SKILL.md(Iron Law 9 / 单轮协议第4步 /
> 新增「修复成本闸门」章节 / 升级协议含 parked 项 / Red Flags / 借口表),board/log 模板与 diagnostician-brief 同步。
>
> 补充决议(执行 Task 4–5 期间):**Reviewer 读懂测试 + 两评审节点 + 交付单 + 3 次修复上限 + 测试二分持久化 + 状态再细分**。
> ① 状态确诊子态改为三个:已确诊·已修复 / 已确诊·未启动修复(原"未修复",成本过大暂缓)/ 已确诊·修复失败(3 次未果)。
> ② 每轮两个评审节点:节点1 核诊断、节点2 核修复;Diagnostician 各交一张交付单(新增 `references/handoff-template.md`)。
> ③ Reviewer **必须读懂支撑本节点结论的那组 test**、查对错与完整性、重跑;节点2 加跑回归;test 有缺陷→打回("整改",不计入修复尝试)。
> ④ 修复**最多 3 次尝试**,经节点2 证实未解决/回归算一次;3 次未果→「已确诊·修复失败」+ learning 回灌。
> ⑤ 测试**二分持久化**:修复/回归测试进正式套件;诊断探针存 `docs/hypo-driven-ps/<topic>-probes/`,log 记路径,下一轮复用不从零写。
> 已写入 SKILL.md(用户检查点状态列表 / 单轮协议第2–5步与第7步 / 修复成本闸门 / 新增「测试与交付单」章节 / 升级协议 / Red Flags×3 / 借口表×2);
> board-template、log-template、diagnostician-brief 重写,新增 handoff-template、reviewer-brief。
>
> 补充决议(执行 Task 6 前):**运行时提交策略 + 修复失败回退**(目标有 git 才做,否则跳过仅靠 log)。
> ① 每轮两次提交:节点1 通过→Diagnostician 提交诊断探针(`hypo-driven-ps(diag): …`);节点2 通过→提交修复(`hypo-driven-ps(fix): …`);
> 板/日志更新→主 agent 在第6步提交(`hypo-driven-ps(board): …`)。
> ② 修复 3 次失败→**回退 + 存档**:失败 diff 存 `<topic>-probes/failed-fixes/第N轮-H?.patch`;`git checkout <节点1提交> -- <修复改动路径>`
> 回退工作树到修复前(不动探针与存档);提交回退;记「已确诊·修复失败」+ 三次失败 learning。下轮在干净态继续。
> 已写入 SKILL.md(Iron Law 10 / 新增「提交与回退」章节 / Red Flags)、diagnostician-brief、log-template。
>
> 补充决议(执行 Task 6 前,对照 systematic-debugging 补缺):**完整继承 sys-debug Phase 1–3 的考量**(因我们用自研流程替代它)。补:
> ① **基线 triage**(读报错/堆栈 1.1、稳定复现 1.2、查最近改动 1.3)——第0步证据稀薄时主 agent 先派 Diagnostician 取基线再建板(不破坏"主 agent 不下场"分工)。
> ② **诊断启发式**:多组件边界插桩先定位 WHERE(1.4,大半在 root-cause-tracing.md,加指针)、**模式对照**(Phase2,known-good 差分)。
> ③ **智识诚实**(3.4)= Iron Law 11:不懂别装,证据不足不下结论。
> ④ **跨轮质疑架构**(F):回灌重排时若"多确诊项修复失败/每修一处别处冒新问题"→架构信号,停下交用户(opt3)。
> ⑤ **无可修根因→缓解+监控**(E):确属外部/环境/时序时的合法终态(重试/超时/降级 + 监控),非硬修。
> 已写入 SKILL.md(第0步基线 triage / 单轮协议第6步c 架构检查 / 终止判定 E / Iron Law 11 / 「取证参考文件」扩为「诊断启发式与取证技术」/ Red Flags×2 / 借口表×2),diagnostician-brief 同步(triage 角色 + 模式对照/边界插桩 + 诚实)。
>
> 补充决议(首次实跑反馈):**简报两维全标注 + 术语统一**。用户简报里假设的两维必须写 `（可能性:高 · 验证难度:中）`,
> 禁止裸 `(高/中)`(用户无法分辨哪个是难度/可能性)。board 列名由「先验概率/验证成本」改为「可能性/验证难度」(语义不变)。
> 已写入 SKILL.md(用户检查点格式规则 + Red Flag)、board-template(列名+注)、log-template(术语)。

---

# v2 增量设计(2026-06-14,已过 brainstorm + 用户通过)

> 范围:worktree 隔离门禁 / verdicts 持久化层 / 跨 session 续跑 / 工作语言锚定 / 收尾合并 / board 头部字段。
> 触发动机:实跑中出现共享工作树串台(另一 session 的改动被 Reviewer 误判为 Diagnostician 所为、错误要求 rollback);
> 以及"主 agent 读完整 verdict 留上下文"信息量过大、跨 session 无法续跑等。

## v2.1 Worktree 隔离门禁(第0步最前,先于基线 triage)

- 第0步**最先**用 `using-git-worktrees` 确保隔离工作树:该 topic 的 worktree **已存在 → 进入续跑;否则新建**。
- 所有产物(board / log / verdicts / probes)落在 **worktree 内**——根除跨 session/跨任务的工作树串台。
- **无 git 退路**:默认**拒绝启动** + 引导(`git init` / 换到 git 项目);**仅当用户显式坚持**才降级"裸跑",
  并明确告知失去 隔离 / 提交 / 回退 / 续跑 四项保障。
- 新增 **Iron Law**:有 git 时**必须在隔离 worktree 中运行**。

## v2.2 verdicts 持久化层(细粒度证据,与 log 分层)

- 目录 `docs/hypo-driven-ps/<topic>-verdicts/`,**一轮一文件** `R<NN>-H<NN>.md`(round 号在前、H 号在后,各补零两位;
  极端超 99 再走三位,基本不会)。
- 文件内**子轮分节**:`## R01-H02.1 diagnostic` / `## R01-H02.1 diagnostic-review` / `## R01-H02.1 repair` /
  `## R01-H02.1 repair-review` / `## R01-H02.2 diagnostic` …(Reviewer 的核验 verdict 跟产出名 +`-review`)。
- **子轮号 y 的递增规则**:**仅"重新诊断"(同一轮内换角度/重诊)才 y+1**。整改打回(test 缺陷重做)、修复重试(第2/3次)
  **留在同一 y 内**,用 `attempt 2/3` 标记。→ 子轮 = 一次诊断主导的微循环,编号最稳。
- **写入与消费(双轨,省主 agent 上下文)**:产出 verdict 的 subagent(Diagnostician 写 diagnostic/repair;
  Reviewer 写 *-review)把**完整 verdict 写进当轮 verdicts 文件**,**只回 1–2 行 digest 给主 agent**。
  主 agent 上下文只承载 digest;完整 verdict 在盘,**按需才读**。
- **log 瘦身**:每轮条目缩成 `round · hypo · 最终结论 · 状态变化 · 一句 learning · → verdicts 文件路径`;
  原先塞在 log 的证据摘要/测试清单**下沉到 verdicts**。log 仍是"每轮通读全量"的决策/叙事层,必须保持精简。
- 新增模板 `references/verdict-template.md`(diagnostic / diagnostic-review / repair / repair-review 四类节的字段)。

## v2.3 跨 session 续跑

- 新 session **重进同一 topic worktree**(v2.1 已存在则进)。
- **主 agent 读最新 board(当前快照 = 整盘假设 + 状态 + 排序)+ log(跨轮 why)重建假设框架**;
  **不读全部 verdicts**(按需)。
- **round 号续号 = 现有最大 R + 1**(从 board 头部「最新轮号」/ verdicts 目录 / log 末条推),**不 reset 到 R01**;
  所有新文书接已有轮次往上。
- **可选派基线 Diagnostician 重核"代码现实"**(两次 session 间代码可能漂移:别的改动、上轮修复是否还在),再据此续框架。
- 无既有文档 → fresh start(R01)。
- 分工:**主 agent 读 board+log 重建决策层**;**基线 Diagnostician 重核代码证据层**。

## v2.4 工作语言锚定

- **首次启动**:主 agent 用当时(= 主 agent 当前对话)语言,定 **`工作语言: X` 写进 board 头部**。
- **全程沿用**(含续跑:读 board 头取语言)。spawn 时**注入每个 subagent prompt**:所有产出
  (verdict / 交付单 / log / 与主 agent 沟通)用工作语言;引用的英文 skill(TDD 等)**"理解用英文、产出用工作语言"**。

## v2.5 收尾合并

- 问题解决 / 用户结束 → 走 `finishing-a-development-branch`,把 **merge / 留 PR / 仅清理** 的选择**交用户**。
  **不自动 merge**(外向且难回滚)。

## v2.6 board 头部新增字段

- `工作语言: X`(v2.4)
- `最新轮号: R<NN>`(让续跑续号确定、零猜测)

## v2 影响的文件

- **SKILL.md**:第0步(worktree 门禁 → 续跑读 board+log/续号 → 基线 triage)、verdicts 与 log 分层、spawn 步注入工作语言、
  收尾走 finishing-a-development-branch、新增 Iron Law(worktree)、Red Flags(无隔离裸跑/未续号 reset/未瘦身 log 等)。
- **board-template.md**:头部加 `工作语言` + `最新轮号`。
- **log-template.md**:A 条目瘦身为 摘要 + verdicts 指针。
- **新增 references/verdict-template.md**:四类 verdict 节模板。
- **diagnostician-brief.md / reviewer-brief.md**:写完整 verdict + 回 digest、用工作语言、(diag)续跑重核代码。
- **handoff-template.md → 合并进 verdict-template,retire**:交付单(D→R 的当场材料)与落盘 verdict 高度重叠——
  D 给 R 的"诊断/修复交付单"本就是它写进 verdicts 文件的 diagnostic/repair 节,R 的批准/整改即 *-review 节。
  故**统一为单一 `verdict-template.md`**(交付单字段成为 diagnostic/repair 节的字段),**删除 handoff-template.md**,
  并改 SKILL/brief 里对「交付单」的引用为「verdict 节(按 verdict-template)」。避免两份重叠模板。
