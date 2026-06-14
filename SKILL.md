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
10. **每节点通过须提交,修复失败须回退**(目标有 git 时)—— 节点1/节点2 各 Reviewer 通过后各一次 commit;修复 3 次失败 → 工作树回退到修复前 + 失败 diff 存档。
11. **智识诚实,不懂别装** —— 判据测不了→「无法判断」;证据不足→不下结论;不清楚就如实说"我不理解 X"、去研究或回报主 agent,绝不臆造信心。

## 产物:两份文件(入 docs/,提交进 Git)

- **假设板** `docs/hypo-driven-ps/<topic>-board.md` —— 当前状态快照,主 agent 每轮重写。结构见 `references/board-template.md`。
- **迭代日志** `docs/hypo-driven-ps/<topic>-log.md` —— append-only,只追加不改写。结构见 `references/log-template.md`。

> 注:此处有意覆盖"临时文件 `_tmp_` 前缀、不入库"的全局规范——这是可追溯的诊断档案,不是临时脚本。

## 优先级规则:信息增益 ÷ 成本

在「待验证」中,优先选「一测就能最大限度缩小不确定性(排除或确认一大片)÷ 验证成本」最高者——即便它不是最高概率。定性判断(高/中/低)即可,不必算公式。

- 易验证 + 高信息量 → 最先做。
- 高概率但极难验证 → 往后排,先用便宜的验证把可能性空间切小。

## 单轮协议(The Loop)

**第 0 步(首次,主 agent 做)**:框定问题(症状 + "怎样算解决"的成功判据)。

> **基线 triage(证据稀薄时先做,尤其代码 bug)**:主 agent 不亲自读代码/跑命令,先派一个 Diagnostician 做基线取证再建板——① **仔细读报错/堆栈/症状**(报错常含答案,记下行号/错误码);② **建立稳定复现**(能否可靠触发?不能→先取数据,别急着假设);③ **查最近改动**(`git diff`/最近提交/新依赖/配置/环境差异——最廉价高产的假设来源)。Diagnostician 回报基线证据,主 agent 据此生成更靠谱的初始假设。非代码问题(agent 行为/研究)同理:先确认可复现、收集基线观测。

→ 生成初始假设集(每个带 描述/判据/概率/成本)→ 用模板建立 board 与 log 两份文件 → 选第一轮 Hypo,写「当前轮计划」→ **过【用户检查点】(见下节)**。

**每一轮:**

1. **主 agent 编排**:按优先级规则在「待验证」中选本轮 Hypo,在 board 写「当前轮计划」(验哪条判据、建议手段)→ **过【用户检查点】**。
2. **spawn Diagnostician(opus,每轮全新)— 诊断**:用 Agent 工具,`subagent_type: general-purpose`、`model: opus`、**串行(不 run_in_background)**;prompt = `references/diagnostician-brief.md` 的全文 + 当前轮计划 + 两份文件的绝对路径。它写测试、验证,交付 **「诊断交付单」**(结论 + 证据 + 测试清单;见 `references/handoff-template.md`)。此时不写 log。
3. **spawn Reviewer(opus,每轮全新)— 节点1 核诊断**:同样 `subagent_type: general-purpose`、`model: opus`、串行;prompt = `references/reviewer-brief.md` 的全文 + 诊断交付单 + 当前轮计划。它**读懂**交付单里支撑结论的那组 test(查对错与完整性)、重跑,调 `verification-before-completion` → **批准** 或 **整改意见**。
   - 整改(含 test 写错/组合不完整)→ 用 `SendMessage` 续**同一个** Diagnostician 补齐/重做(不起新实例),再用 `SendMessage` 续**同一个** Reviewer 复核,直到通过。
4. **确诊后(节点1 批准为「真」):修复成本闸门 + 修复**(结论为 假 / 无法判断 → 跳过本步;详见「修复成本闸门」章节):
   - **成本过大**(命中任一信号)→ Diagnostician **不修**,带成本评估上报主 agent → 主 agent 告知用户 → **用户决策**:立即修 / 暂缓(记「已确诊·未启动修复」)。
   - **小修(都不命中)或 用户决策立即修** → **同一个** Diagnostician 修(`test-driven-development` 写失败复现测试 → 最小实现),交 **「修复交付单」** → **同一个** Reviewer **节点2 核修复**(`verification-before-completion`;读懂修复 test + 重跑 + 跑回归)。
     - **修复尝试上限 3 次**:经节点2 证实未解决/有回归算一次失败(因 test 缺陷被打回的"整改"不计入);3 次仍未果 → 记「已确诊·修复失败」,把"各次试了什么、为何失败"作为 learning 返回主 agent。
5. **Diagnostician 写 log**:经 Reviewer 通过后,把本轮「Diagnostician 轮次记录」(验证结论 + 证据 + 测试清单;若确诊则含 修复成本评估 + 修复结果/尝试次数)**追加**到 log。
6. **主 agent 回灌重排(硬门禁)**:
   a. **通读全量 log**(不可跳过),提炼跨轮的更深 insight。
   b. 复核**整盘**假设:逐个看 描述/判据/概率/成本/状态 是否要改;增删假设;重新排序。
   c. **跨轮架构信号检查**:若出现「多个确诊项都修复失败」或「每修一处别处就冒新问题」的模式 → 这是**架构问题**而非单个假设失败,停下交用户(走升级协议 opt3),不要继续打补丁。
   d. 把「主 agent 板更新记录」**追加**到 log,并重写 board 快照。
   e. **回收本轮的 Diagnostician 与 Reviewer**(此后不再向它们发消息)。
7. **终止判定**:
   - 本轮 Hypo 已确诊**且修复后经验证确实解决** → 状态「已确诊·已修复」,**完成**。
   - 为真但问题未解(症结不完整/有并存症结)、为假、无法判断、确诊但用户暂缓修复(已确诊·未启动修复)、或确诊但修复 3 次失败(已确诊·修复失败)→ 回第 1 步,选下一轮 Hypo。
   - **确属外部/环境/时序且无可修代码根因**(经充分调查,非偷懒下的结论)→ 这是合法终态:不硬修,改做**缓解**(重试/超时/降级/明确报错)+ 加**监控/日志**便于复发时再查;交用户确认后**完成**。
   - 假设板打空 → 进「升级协议」。

> **上下文边界**:主 agent 不读源码、不跑测试、不亲自取证;这些只发生在 Diagnostician/Reviewer 的上下文里。主 agent 只消费它们交付的结论与证据摘要来演进假设板。

## 用户检查点(每轮 spawn 前的人在环)

主 agent 每当**画好/更新好假设板、写下当前轮计划之后、spawn Diagnostician 之前**,向用户**简报**(简略,别长篇):

1. **hypo 概况**:当前有哪些假设、各自状态(待验证/已确诊·已修复/已确诊·未启动修复/已确诊·修复失败/已排除/无法判断/无需验证)。
2. **前三 hypo 的排序缘由**:为什么这三个排在前、本轮为何先验选中的那个(信息增益÷成本)。
3. **(后续轮)上一轮发现 + 对本轮的影响**:上轮验出了什么、据此整盘做了哪些调整。

> **呈现格式(避免歧义,必守)**:每条假设的两个维度必须**全标注**,写成 `（可能性:高 · 验证难度:中）`。**禁止**用裸 `(高/中)` 让用户猜哪个是可能性、哪个是难度。(可能性 = 先验概率;验证难度 = 验证成本。)

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
- **暂缓** → 记状态「已确诊·未启动修复」,继续查 / 先修别的小症结(多个小修叠加可能就缓解了问题)。

> 此闸门**即便在自主模式(`/loop`、`/goal`)下也生效**——成本过大的修复高影响、难回滚,必须回到用户决策。

**修复尝试上限 3 次**:无论小修还是用户决策立即修,Diagnostician 修复最多试 3 次。经 Reviewer 节点2 证实「未解决 / 引入回归」算一次失败(因 test 写错或组合不完整被打回的"整改"**不计入**)。3 次仍未果 → 状态「已确诊·修复失败」,把"各次试了什么、为何失败"作为 learning 返回主 agent(它很可能提示该症结不完整或属架构问题,供重排/升级用)。修复成功并经验证解决 → 「已确诊·已修复」。

## 测试与交付单(两个评审节点)

每轮有**两个评审节点**,Diagnostician 各提交一张交付单(字段见 `references/handoff-template.md`),Reviewer 照单核:

- **节点1 — 诊断交付**:Diagnostician 提交「诊断交付单」(结论真/假/无法判断 + 证据 + 测试清单)。
- **节点2 — 修复交付**:确诊并决定修复后,提交「修复交付单」(修复方案 + 复现/回归测试 + 第几次尝试 + 证据)。

**Reviewer 必须读懂 test**:不懂 test 测了什么就无法判断该不该过。两节点都要**读懂支撑本节点结论的那组 test**、查对错与完整性、重跑;节点2 额外重跑回归/既有套件。**test 有逻辑漏洞或组合不完整 → 本轮打回**让 Diagnostician 补齐(属"整改",**不计入**修复尝试次数)。

**测试持久化(二分)**:

- **修复/回归测试** → 进项目**正式测试套件**(永久护航)。
- **诊断验证测试(探针)** → 存到 `docs/hypo-driven-ps/<topic>-probes/`,在 log 证据里记路径。下一轮全新 Diagnostician 通过 log 找到、复用/扩展,**不从零写**;Reviewer 可重跑。

## 提交与回退(运行时,有 git 才做)

> 以下是 skill 在**目标项目仓库**中的运行时行为。目标无 git → 跳过提交/回退,仅靠 log 留痕。

**每轮两次提交(Reviewer 通过后):**

- **节点1 通过 → 提交诊断产物**(由 Diagnostician):探针 test(`docs/hypo-driven-ps/<topic>-probes/`)等。讯息如 `hypo-driven-ps(diag): 第N轮 H? <真/假/无法判断>`。
- **节点2 通过 → 提交修复**(由 Diagnostician):修复代码 + 复现/回归 test。讯息如 `hypo-driven-ps(fix): 第N轮 H? <摘要>`。
- **板/日志更新 → 由主 agent 在回灌重排(第6步)提交**:讯息如 `hypo-driven-ps(board): 第N轮`。

**修复 3 次失败 → 回退 + 存档:**

1. 把失败尝试的 diff 存档为 patch:`docs/hypo-driven-ps/<topic>-probes/failed-fixes/第N轮-H?.patch`(`git diff <节点1提交> -- <修复改动路径>`)。
2. 工作树**回退到修复前**(= 节点1 提交):`git checkout <节点1提交> -- <本次修复改动的代码与 test 路径>`——只回退本次修复的改动,**不动**已存档 patch 与节点1 的诊断探针。
3. 状态记「已确诊·修复失败」,log 写明三次各试了什么、为何失败;提交此回退 + 存档(`hypo-driven-ps(fix-failed): 第N轮 H? 回退并存档`)。
4. 下一轮在干净状态上继续。

## 升级协议(假设板打空)

当「待验证」全部清空(整盘只剩 已排除 / 无法判断 / 已确诊·未启动修复 / 已确诊·修复失败),问题仍未解决:主 agent 停下。**若存在「已确诊·未启动修复」(大修待决)或「已确诊·修复失败」(试 3 次未果)项**,先把这些连同其 learning 呈给用户决策(执行大修 / 换思路 / 接受现状);若无此类项、或用户仍不推进而问题未解,再**按 2 → 1 → 3 顺序**呈报用户:

1. **(无法判断项)** 它们是否值得攻克?把它变可测的成本(加探针/instrumentation/造可复现样本)?
2. **(新方向)** 还有哪些新假设/新方向值得探索?
3. **(质疑框定)** 问题框定是否错了 / 架构是否根本不对?

→ 主 agent 综合三项 + 自身判断,**给出下一步推荐**。但**最终决策权交用户**,不擅自继续。

## 复用的 skill(由 subagent 在各自上下文里 invoke)

- **Diagnostician** → `test-driven-development`(为判据/修复写失败测试再实现)。
- **Reviewer** → `verification-before-completion`(核验证据、确认问题真解决)。

> 这些 skill 自身**不 spawn subagent、不自动链式调用**——它们是行为指引型 prompt,在哪个 agent 上下文里被读到就约束哪个 agent。

## 诊断启发式与取证技术(Diagnostician 按需用)

**通用启发式(验证判据/找根因时常用):**

- **多组件系统:边界插桩,先定位 WHERE 再查 WHY** —— 在各组件/层的边界打印进出数据,跑一次看是哪一层断,再深挖那层(详见 `root-cause-tracing.md`)。
- **模式对照(差分)** —— 找一个 known-good 的可工作对照(同库里相似的能跑的代码、参考实现),逐处 diff,**列出每一个差异**,别假设"这点不可能有影响"。

**参考文件(同目录 `references/`,命中才 Read):**

- `root-cause-tracing.md` —— 深栈 / 坏数据来源不明时,反向追踪到源头(含边界插桩技法)。
- `condition-based-waiting.md` —— 验证对象是 flaky / 竞态 / 时序问题(测试里有 setTimeout/sleep)。
- `defense-in-depth.md` —— **最窄门禁**:仅当 ① 已确诊根因是「代码逻辑缺陷,由非法/畸形数据跨层传播引起」**且** ② 处于修复后的加固阶段,才 Read 并应用。**明确排除**:agent 行为/环境类、非代码类、研究/选型类问题——即便被当作 debugging 在跑,也不得套用多层校验。

> 本 skill 刻意**不直接 invoke `systematic-debugging`**(其 Phase 1–3 与本流程重叠会冲突)。其 Phase 1–3 的考量(读报错、稳定复现、查最近改动、边界插桩、模式对照)已分别吸收进上面的「基线 triage」与本节启发式;三份附属文件从它复制为独立参考,被动 Read,不触发它的四阶段。

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
- Reviewer 没读懂 test 就放行;或 test 有漏洞/组合不完整却放过。
- 修复无限重试,没在 3 次封顶;或把 test 缺陷的"整改"也算进 3 次。
- 探针 test 用完即弃没持久化;或下一轮没读 log 复用、又从零写 test。
- 节点通过了没提交(目标有 git);或修复 3 次失败没回退、把不工作的半成品留给下轮。
- 证据稀薄却跳过基线 triage(没读报错/没确认能复现/没查最近改动)就硬凑假设。
- 证据不足却下了结论,或装懂(本该「无法判断」或"我不理解 X"却含糊带过)。
- 简报里用裸 `(高/中)` 让用户猜哪个是可能性、哪个是难度(必须全标注)。

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
| "Reviewer 大致扫下证据就行,不必懂 test" | 不懂 test 测什么就无法判断真伪。两节点都要读懂该组 test 并重跑。 |
| "这个修复再多试几次总能成" | 修复封顶 3 次。3 次未果是 learning(症结可能不完整/架构问题),回灌重排,不无限试。 |
| "懒得查最近改了啥,直接想原因" | `git diff`/新依赖/配置/环境差异 是最廉价高产的假设来源,先查。 |
| "差不多就是它了,先当确诊" | 证据不足就别下结论。该「无法判断」就如实标,别臆造信心。 |

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
