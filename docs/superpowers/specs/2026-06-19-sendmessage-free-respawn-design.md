# 设计：移除 SendMessage 依赖，改「重起交接」+ 角色重划

> 日期：2026-06-19 · 课题：sendmessage-free-respawn · 状态：待用户审阅

## 问题

skill 的轮内交接全部依赖 `SendMessage`「续同一实例」。但当前 harness **不提供 `SendMessage`**（ToolSearch 检索不到；Agent 工具描述里的"续聊"是 UI 能力，未作为可调用工具暴露）。一旦某个子 agent 返回 digest 进入 idle，主 agent 就**无法再激活它**。

### 根因：所有"批准后才做的动作"都需要二次激活 idle 的实例

子 agent 交完 digest 即 idle；凡是设计成"夹在 Reviewer 步骤之后"由原实例继续做的动作，都要靠 `SendMessage` 二次激活：

| 触点 | 原设计归属 | 触发条件 |
|---|---|---|
| ① 整改回路（节点1/2 打回重做） | 续同一 D、续同一 R | 有条件（Reviewer 打回时） |
| ② 节点1 批准→转修复 / 成本评估 | 续同一 D | 有条件（结论=真） |
| ③ 用户运行期指令转达 | 续同一实例 | 有条件（用户插指令） |
| ④ 写 log A 条目（批准后） | 续同一 D | **每轮必现** |
| ⑤ commit diag/fix（批准后）/ 3 次失败回退 | 续同一 D | 有条件（有 git） |

### 真实证据（D:\51_test\...\docs\hypo-driven-ps，11 轮）

- ④ **11/11 轮**全部注明"因无 SendMessage，由主 agent 据双 digest 代写 log"——持续性失效、非偶发。
- ① 唯一擦边（06-14 R07）：Reviewer 给了 2 条必改修正，被主 agent 在 log 散文里"盖章"绕过，**整改未被执行者重做**——这是有损降级。
- ⑤ 两课题均 no-commit，commit 路径从未跑过，病被掩盖。
- verdict 文件（如 R04-H1.md）**自足完整**：diagnostic + diagnostic-review + 测试清单 + 证据齐全 → 证明"靠文件重建状态"成立。

## 决策

1. **范围**：改运行期行为文件 + 当前面向人的文档：`SKILL.md`、`references/diagnostician-brief.md`、`references/reviewer-brief.md`、`references/log-template.md`、`README.md`、`DESIGN.md`。dated `docs/.../plans|specs` 作为历史快照**不动**。
2. **实现**：**md-only**。重起交接本质是 agent 动作（主 agent 调 Agent 工具 spawn），无代码杠杆可强制/校验 spawn 的 prompt；新增脚本不能减少 agent-compliance 依赖，净收益为负。可靠性建在"verdict 文件已是单一状态源"这一既有事实上。
3. **DRY**：「重起交接」机制在 `SKILL.md` **只定义一次**；①②③ 三个触发点只写一句指针 + 本次注入什么。`SKILL.md`(主 agent) 与 brief(子 agent) 是不同读者，各留自己一侧不算重复。

## 新模型：三角色，无 post-approval 二次激活

### 重起交接（轮内每次 handoff 的统一做法）

上一个实例返回 digest 即回收；下一步**重新 spawn 全新实例**，prompt 注入三样：① 当轮 verdict 文件路径（上次产出/证据/对方意见都在其中）；② 角色·节点（诊断/核诊断/修复/核修复）+ attempt 号；③ 本次具体诉求。子 agent **先读 verdict 文件恢复上下文**再接着干。

用于 ①整改（诉求=Reviewer 整改意见，同一 H、attempt+1）、②转修复/成本评估（诉求=转修复 + 确诊指针）、③用户指令转达（诉求=用户指令，注入下一个**同角色**实例）。

### 角色归属（关键变化）

- **Diagnostician = 纯生产者**：诊断 / 成本评估 / 修复 / 整改重做，写 verdict、返回 digest。**不写 log、不碰 git**。每次都是经重起交接起的全新实例。
- **Reviewer = 本轮收尾者**：核验后，作为**仍活着**的最后一个执行者完成 post-approval 动作——**无需 SendMessage**：
  - **写 log A 条目**：每确认一个节点（节点1 得出 真/假/无法判断；节点2 修复 解决/失败），就**追加**该节点的 A 片段（它本就为复核读了完整 verdict，上下文最全、无损）。被打回的复核不写 A。
  - **git**：节点1 批准→commit diag 探针/verdict（**此 commit 即修复失败的回滚锚点**，故必须由此刻活着的 Reviewer 做，不能攒）；节点2 批准→commit fix；3 次失败→`git checkout <节点1 commit>` 回退 + 存档 patch + 提交回退。
- **主 agent = 编排者**：选 hypo、写 board「当前轮计划」、过用户检查点、spawn；**第 6 步（本轮尾、Reviewer 返回后、仍同一轮）**回灌重排，写 board 快照 + log **B 条目**，commit board/log。**只消费 1–2 行 digest，全程不读 verdict、不碰诊断代码** → 上下文恒瘦。

> 时序保证：board 更新是本轮第 6 步、主 agent 此刻活着 → board commit 落在**本轮尾**，不跨轮。每轮通常 2 个 commit：Reviewer 提交"发现+回滚锚点"、主 agent 提交"板演进"。

### log A/B 分工（解决"哪个 Reviewer 写"与终态边界）

- **A（本轮实质发现）**：Reviewer 按节点**增量追加**，永不需预测自己是否"最后一个"。
- **B（板演进/状态变化/处置）**：主 agent 写。成本闸门暂缓、打空、修复失败等**靠主 agent/用户决策收尾**的处置进 B —— 这类轮没有"最终 Reviewer"也不留缺口。

### Iron Law 影响

- #1（主 agent 不下场）：不变。主 agent commit 自己写的 board/log = 记账，非取证/验证/修复；回退代码由 Reviewer 做，主 agent 不碰代码工作树。
- #4（无 Reviewer 批准不写 log）：不变，且更顺——现在就是 Reviewer 在批准时亲手写 A。
- #10（每节点通过须 commit / 失败须回退）：执行者明确为 **Reviewer**。

### verdict 文件：attempt 追加（不覆盖）

整改/修复重试的多次 attempt 必须**追加保留**（同一 y、标 attempt K），使重起实例看得到"上次试了什么、为何被打回"，避免重犯。模板已支持 `attempt K`，brief 与模板点明 append 语义。

## 逐文件落点

- **SKILL.md**：§24 立「重起交接」权威定义（含"实例每次全新、从 verdict 重建"）；§80/§83/§121 改为指针 + 注入物；§85 第 5 步删"Diagnostician 写 log"，改 Reviewer 写 A；§86 第 6 步明确主 agent 写 B + commit board/log；§91 回收措辞改为"每次 handoff 即回收，本步为本轮最后回收"；Iron Law #10 点明 commit/回退执行者=Reviewer。
- **diagnostician-brief.md**：§12 删"续聊/SendMessage"，改"每次全新实例、先读 verdict 恢复、做哪个节点以 prompt 注入为准"；§25 点明 attempt 追加；§26/§37/§38 删 commit/回退（移交 Reviewer）；§40–42 删"写 log"段。
- **reviewer-brief.md**：§42 反转"不写 log"；新增"收尾者"职责块——增量追加 A、commit diag/fix、3 次失败回退。
- **log-template.md**：A 条目允许按节点增量追加；写明 A=Reviewer、B=主 agent。
- **README.md** §70/§72、**DESIGN.md** §164：去 SendMessage 机制描述，降到行为层，指向 SKILL.md「重起交接」为权威。

## 不做（YAGNI）

- 不新增任何脚本/hook。
- 不改 board/verdict/probe 目录结构与命名。
- 不改 dated 历史 plans/specs。
- 不动 H 编号格式不一致等既有瑕疵（超范围）。
