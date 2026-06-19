# 移除 SendMessage 依赖 + 重起交接 + 角色重划 — 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 hypo-driven-ps 的轮内交接从依赖 `SendMessage`「续同一实例」改为「重起交接」(全新 spawn + 靠 verdict 文件重建),并重划角色:Diagnostician 纯生产者、Reviewer 本轮收尾者(写 log A + commit)、主 agent 写 B + commit board/log。

**Architecture:** md-only。改 6 个文件:`SKILL.md`(立单一权威定义 +3 指针 + step5/6 + Iron Law10)、`references/diagnostician-brief.md`(去 log/git/SendMessage)、`references/reviewer-brief.md`(加收尾者职责)、`references/log-template.md`(A 由 Reviewer 按节点追加 / 状态归 B)、`README.md`、`DESIGN.md`(追加超代决议,不改历史)。「重起交接」机制只在 SKILL.md 定义一次,余处引用。

**Tech Stack:** Markdown only。验证用 `grep`。仓:`C:\Users\wenjie\.agents\skills\hypo-driven-ps`,分支 `sendmessage-free-respawn`。

**约束:** md-only;新文字精确简短;同一事件禁跨文件/多处重复(SKILL.md=主 agent、brief=子 agent 不同读者各留一侧除外)。`docs/superpowers/` 被 gitignore,spec/plan 本地不提交。

**任务顺序(有依赖):** Task 1 SKILL.md 先立权威定义 → Task 2/3 briefs 依赖该角色模型 → Task 4 模板 → Task 5/6 文档。每个 Task 自成一次 commit。

> **MD 计划的"测试"约定:** 无单元测试。每个 Task 的验证 = 对改后文件跑 `grep`:① 禁用词(`SendMessage`/`续同一实例`/`续聊`)计数=0;② 新锚点存在。命令在仓根运行(`cd "C:/Users/wenjie/.agents/skills/hypo-driven-ps"` 或用 `git -C`)。

---

## Task 1: SKILL.md — 立「重起交接」权威定义 + 3 指针 + step5/6 + Iron Law10

**Files:**
- Modify: `SKILL.md`(§24 生命周期、§37 Iron Law10、§80 整改、§83 转修复、§85 step5、§90/§91 step6、§121 指令转达)

- [ ] **Step 1: 改 §24 生命周期段 —— 立「重起交接」权威定义**

old_string(整段):
```
**subagent 生命周期(每轮回收)**:每一轮起**全新**的一对 Diagnostician/Reviewer——它们靠 board + log 两份文件重建所需状态,不靠跨轮记忆,以保证反锚定、上下文有界。**轮内**的「Diagnostician → Reviewer → 整改 → Diagnostician」交接用 `SendMessage` **续同一个实例**(让它记得自己刚做了什么、Reviewer 提了什么),**不要**中途起新的。本轮 log 写完、board 更新后,这一对**一起回收**;下一轮再起全新一对。
```
new_string:
```
**subagent 生命周期**:Diagnostician/Reviewer **每次都是全新实例**,不靠对话记忆——状态全部从课题文件(board/log + 当轮 `R<NN>-H<NN>.md` verdict)重建,以保证反锚定、上下文有界。

**重起交接(轮内每次 handoff 的统一做法)**:上一个实例返回 digest 即回收;下一步**重新 spawn 全新实例**,prompt 注入三样——① 当轮 verdict 文件路径(它的上次产出/证据/对方意见都已写在其中);② 角色·节点(诊断/核诊断/修复/核修复)+ attempt 号;③ 本次具体诉求。子 agent **先读 verdict 文件恢复上下文**再接着干。

**谁写、谁 commit**:Diagnostician 只写 verdict、返回(不写 log、不碰 git);**Reviewer 是本轮收尾者**——核验后(仍活着,无需续聊)按节点**追加 log A 条目**、并按 Iron Law #10 commit diag/fix 或回退;主 agent 在第 6 步写 log **B 条目** + commit board/log。
```

- [ ] **Step 2: 改 §37 Iron Law #10 —— 点明执行者 = Reviewer**

old_string:
```
10. **每节点通过须提交,修复失败须回退**(目标有 git 时)—— 节点1/节点2 各 Reviewer 通过后各一次 commit;修复 3 次失败 → 工作树回退到修复前 + 失败 diff 存档。
```
new_string:
```
10. **每节点通过须提交,修复失败须回退**(目标有 git 时)—— 由收尾的 **Reviewer** 执行:节点1/节点2 各通过后各一次 commit(**节点1 的 commit 即修复失败的回滚锚点**,故必须由此刻活着的 Reviewer 当场做);修复 3 次失败 → Reviewer 把工作树回退到修复前 + 存档失败 diff。
```

- [ ] **Step 3: 改 §80 整改 —— 指针**

old_string:
```
   - 整改(含 test 写错/组合不完整)→ 用 `SendMessage` 续**同一个** Diagnostician 补齐/重做(同一 y,标 attempt;不起新实例),再用 `SendMessage` 续**同一个** Reviewer 复核,直到通过。
```
new_string:
```
   - 整改(含 test 写错/组合不完整)→ 走**重起交接**起 Diagnostician 补齐/重做(诉求 = Reviewer 整改意见;同一 y、attempt+1),再**重起交接**起 Reviewer 复核,直到通过。
```

- [ ] **Step 4: 改 §83 转修复 —— 指针**

old_string:
```
   - **小修(都不命中)或 用户决策立即修** → **同一个** Diagnostician 修(`test-driven-development` 写失败复现测试 → 最小实现),**把 repair verdict 写进当轮 verdict 文件** → **同一个** Reviewer **节点2 核修复**(`verification-before-completion`;读懂修复 test + 重跑 + 跑回归,**写 repair-review verdict**)。
```
new_string:
```
   - **小修(都不命中)或 用户决策立即修** → 走**重起交接**起 Diagnostician 修(诉求 = 转修复 + 确诊指针;`test-driven-development` 写失败复现测试 → 最小实现),repair verdict 写进当轮 verdict 文件 → 再**重起交接**起 Reviewer **节点2 核修复**(`verification-before-completion`;读懂修复 test + 重跑 + 跑回归,写 repair-review verdict)。
```

- [ ] **Step 5: 改 §85 step5 —— Diagnostician 写 log → Reviewer 写 A 条目**

old_string:
```
5. **Diagnostician 写 log(摘要)**:经 Reviewer 通过后,把本轮「Diagnostician 轮次记录(摘要)」按 `log-template.md` 的 A 条目**追加**到 log——只放 结论/状态变化/一句 learning/提交/**→ verdicts 文件指针**;完整证据已在 verdict 文件。
```
new_string:
```
5. **Reviewer 写 log A 条目**:Reviewer 每确认一个节点(节点1 = 真/假/无法判断;节点2 = 修复解决/失败),返回 digest 前按 `log-template.md` A 条目**追加**该节点记录——结论 + 一句 learning + **→ verdict 文件指针**(完整证据在 verdict);被打回的复核不写。**状态变化/重排/处置归 B 条目(主 agent),不进 A。**
```

- [ ] **Step 6: 改 §90 step6.d —— B 条目 + commit board/log**

old_string:
```
   d. 把「主 agent 板更新记录」**追加**到 log,并重写 board 快照(**含更新 board 头「最新轮号」= 本轮 R<NN>**)。
```
new_string:
```
   d. 把「主 agent 板更新记录(B 条目:跨轮 insight / 状态变化 / 重排 / 处置)」**追加**到 log,重写 board 快照(**含更新 board 头「最新轮号」= 本轮 R<NN>**),并(有 git)commit board/log。
```

- [ ] **Step 7: 改 §91 step6.e —— 回收措辞**

old_string:
```
   e. **回收本轮的 Diagnostician 与 Reviewer**(此后不再向它们发消息)。
```
new_string:
```
   e. 本轮实例已随每次重起交接回收,无遗留;下一轮全新起。
```

- [ ] **Step 8: 改 §121 用户指令转达 —— 指针**

old_string:
```
   - **与当前子 agent 任务相关的可执行指令** → 主 agent **即时记下(buffer)**,**待该子 agent 返回 digest 时,先用 `SendMessage` 续同一实例转达,再走下一协议步**。
```
new_string:
```
   - **与当前子 agent 任务相关的可执行指令** → 主 agent **即时记下(buffer)**;待该子 agent 返回后,走**重起交接**把该指令注入下一个**同角色**实例的 prompt(诉求 = 用户指令),再走下一协议步。
```

- [ ] **Step 9: 验证 SKILL.md**

Run: `grep -nE "SendMessage|续同一|续聊" SKILL.md`
Expected: 无输出(0 行)。

Run: `grep -c "重起交接" SKILL.md`
Expected: ≥4(1 处定义 + 3 处指针;§24 含 2 次→实际 ≥5)。

- [ ] **Step 10: Commit**

```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" add SKILL.md
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" commit -m "$(printf 'refactor(SKILL): 重起交接替代 SendMessage + 角色重划\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 2: diagnostician-brief.md — 去 log/git/SendMessage,纯生产者

**Files:**
- Modify: `references/diagnostician-brief.md`(§12、§25–26、§37、§38、§40–42、§56)

- [ ] **Step 1: 改 §12 —— 全新实例 + 先读 verdict + 节点以注入为准**

old_string:
```
> 你是**本轮全新起**的实例:不假设记得之前轮的工作;需要历史背景就读 log/verdicts。**log/verdicts 里记了往轮探针 test 的路径**——能复用就复用/扩展,不从零写。轮内若收到 Reviewer 整改意见,你会被续聊继续(那时你记得自己刚做了什么);主 agent 也可能在你返回后用 `SendMessage` 转达用户运行期给的指令——照常续做。
```
new_string:
```
> 你**每次都是全新实例**(诊断/成本评估/修复/整改重做/转达用户指令,每一次都是):不靠记忆。**你的上次产出、证据、Reviewer 的意见都已写在当轮 verdict 文件**——接着干前先读它恢复上下文;**本次做哪个节点以 prompt 注入为准**,已确认的节点结论在 verdict、不重做。往轮探针 test 路径在 log/verdicts 里,能复用就复用/扩展,不从零写。
```

- [ ] **Step 2: 改 §25–26 —— attempt 追加不覆盖 + 删 commit 行**

old_string:
```
   - 收到整改意见(含 test 写错 / 组合不完整)→ 补齐/重做,**写进同一 y、标 attempt**,重新交付,直到节点1 通过。**整改不计入修复尝试次数。**
   - **节点1 通过后(目标有 git)**:提交诊断探针等产物 —— `hypo-driven-ps(diag): 第N轮 H? <真/假/无法判断>`。
```
new_string:
```
   - 收到整改意见(含 test 写错 / 组合不完整)→ 补齐/重做,在 verdict 文件**追加**新 attempt 节(同一 y、标 attempt K,**不覆盖上次 attempt**,使后续实例看得到历次尝试),重新交付,直到节点1 通过。**整改不计入修复尝试次数。**(节点1 通过后的 commit 由收尾的 Reviewer 做,你不提交。)
```

- [ ] **Step 3: 改 §37 —— 删 fix commit**

old_string:
```
   - **成功并经验证解决** → 节点2 通过后(有 git)提交修复 `hypo-driven-ps(fix): 第N轮 H? <摘要>`;状态「已确诊·已修复」。
```
new_string:
```
   - **成功并经验证解决** → repair verdict 记「已修复并验证」;commit 由收尾的 Reviewer 做。
```

- [ ] **Step 4: 改 §38 —— 回退/存档移交 Reviewer**

old_string:
```
   - **3 次仍未果** → (有 git)把失败 diff 存档为 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/failed-fixes/第N轮-H?.patch`;`git checkout <节点1提交> -- <本次修复改动的代码与 test 路径>` **回退到修复前**(不动探针与存档),提交此回退;上报主 agent,记「已确诊·修复失败」,附"三次各试了什么、为何失败"。
```
new_string:
```
   - **3 次仍未果** → 在 repair verdict 写清「三次各试了什么、为何失败」+ 列出本次修复改动的代码与 test 路径(供 Reviewer 回退);返回主 agent 记「已确诊·修复失败」。**存档失败 diff + 回退到节点1 commit 由收尾的 Reviewer 执行**(见 reviewer-brief 收尾者职责)。
```

- [ ] **Step 5: 删 §40–42「写 log(摘要)」整段**

old_string:
```
## 写 log(摘要)

- 两节点都过后,把本轮「Diagnostician 轮次记录(摘要)」按 `log-template.md` 的 A 条目格式**追加**(只追加,不改既有内容;只放 结论/状态变化/一句 learning/提交/→ verdicts 文件指针)。**完整证据已在 verdict 文件**,别往 log 里塞。

## 取证技术(按需用)
```
new_string:
```
## 取证技术(按需用)
```

- [ ] **Step 6: 改 §56「你不做」—— 不写 log、不碰 git**

old_string:
```
- 不在没有 Reviewer 批准时写 log;不擅自执行成本过大的修复;不超过 3 次修复尝试。
```
new_string:
```
- **不写 log、不 commit/回退**(log A 由 Reviewer 写、git 由收尾的 Reviewer 做);不擅自执行成本过大的修复;不超过 3 次修复尝试。
```

- [ ] **Step 7: 验证 diagnostician-brief.md**

Run: `grep -nE "SendMessage|续聊|续同一" references/diagnostician-brief.md`
Expected: 无输出。

Run: `grep -nE "提交诊断探针|提交修复|## 写 log" references/diagnostician-brief.md`
Expected: 无输出(commit/写 log 职责已移除)。

- [ ] **Step 8: Commit**

```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" add references/diagnostician-brief.md
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" commit -m "$(printf 'refactor(diagnostician-brief): 纯生产者,去 log/git/SendMessage\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 3: reviewer-brief.md — 收尾者职责(写 log A + commit/回退)

**Files:**
- Modify: `references/reviewer-brief.md`(节点2 段后插入新章节;改 §42「你不做」)

- [ ] **Step 1: 在「## 你不做」之前插入「收尾者职责」章节**

old_string:
```
## 你不做

- 不替 Diagnostician 重写 test 或重做修复,只核验。
```
new_string:
```
## 收尾者职责(每次复核后、你仍活着时做——无需 SendMessage)

确认一个节点后(节点1 = 真/假/无法判断;节点2 = 修复解决/失败),在返回 digest 前:

1. **写 log A 条目**:按 `log-template.md` A 条目格式,在 log 末尾**追加**该节点记录(结论 + 一句 learning + → verdict 文件指针)。**状态变化/重排/处置不写**(归主 agent 的 B 条目)。被你打回整改的复核**不写**,等重做通过再写。
2. **git(目标有 git,Iron Law #10)**:
   - 节点1 通过 → commit diag 探针/verdict:`hypo-driven-ps(diag): 第N轮 H? <真/假/无法判断>`。**此 commit 是修复失败的回滚锚点**,必须此刻当场做。
   - 节点2 通过 → commit fix:`hypo-driven-ps(fix): 第N轮 H? <摘要>`。
   - 判修复 3 次失败 → 按 verdict 列出的修复改动路径,存档失败 diff 到 `<topic>-probes/failed-fixes/R<NN>-H<NN>.patch` + `git checkout <节点1 commit> -- <修复改动的代码与 test>` 回退到修复前(不动探针与存档)+ 提交此回退。

## 你不做

- 不替 Diagnostician 重写 test 或重做修复,只核验。
```

- [ ] **Step 2: 改 §42「你不做」最后一条 —— 反转"不写 log"**

old_string:
```
- 不更新假设板;不写 log(log 由 Diagnostician 通过后写、由主 agent 写板更新记录)。
```
new_string:
```
- 不更新假设板;不写 log **B 条目**(状态/重排/处置由主 agent 写)。log **A 条目**与本轮 commit/回退由你按上「收尾者职责」做。
```

- [ ] **Step 3: 验证 reviewer-brief.md**

Run: `grep -nE "收尾者职责|回滚锚点|log A" references/reviewer-brief.md`
Expected: 至少各 1 处命中。

Run: `grep -n "不写 log(log 由 Diagnostician" references/reviewer-brief.md`
Expected: 无输出(旧的"不写 log"已反转)。

- [ ] **Step 4: Commit**

```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" add references/reviewer-brief.md
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" commit -m "$(printf 'refactor(reviewer-brief): 加收尾者职责(写 log A + commit/回退)\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 4: log-template.md — A 由 Reviewer 按节点追加,状态归 B

**Files:**
- Modify: `references/log-template.md`(§5–7 头注释、§11–18 A 模板)

- [ ] **Step 1: 改头注释 §5–7 —— A/B 作者与时机**

old_string:
```
  每轮产生两类条目:
    A. Diagnostician 轮次记录(由 Diagnostician 在 Reviewer 批准后写)
    B. 主 agent 板更新记录(由主 agent 在回灌重排后写)
```
new_string:
```
  每轮产生两类条目:
    A. 轮次记录(由 **Reviewer** 确认每个节点后追加:节点1 一片段、节点2 一片段;只写 结论/learning/提交/verdict 指针)
    B. 主 agent 板更新记录(由主 agent 在回灌重排后写:跨轮 insight / 状态变化 / 重排 / 处置)
```

- [ ] **Step 2: 改 A 模板 §11–18 —— 标题 + 按节点 + 移除状态变化**

old_string:
```
## 第 N 轮 — A. Diagnostician 轮次记录（摘要,细节在 verdicts）

- **轮次/假设**:R<NN> · H<NN> — <描述>
- **最终结论**:判据成立(真)/ 证伪(假)/ 无法判断
- **状态变化**:H<NN> → <已确诊·已修复 / 已确诊·未启动修复 / 已确诊·修复失败 / 已排除 / 无法判断>
- **一句 learning**:<本轮最关键的一条收获>
- **提交(有 git)**:diag <hash/N/A> · fix <hash/N/A> · 若回退 fix-failed <hash> + patch 路径
- **→ verdicts 文件**:`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/R<NN>-H<NN>.md`(完整证据/测试清单/各子轮 D/R verdict 在此)
```
new_string:
```
## 第 N 轮 — A. 轮次记录(Reviewer 按节点追加;细节在 verdicts）

- **轮次/假设**:R<NN> · H<NN> — <描述>
- **节点1 结论**:判据成立(真)/ 证伪(假)/ 无法判断 — <一句 learning>
- **节点2 结论(若有修复)**:修复解决 / 失败(原因) — <一句 learning>
- **提交(有 git)**:diag <hash/N/A> · fix <hash/N/A> · 若回退 fix-failed <hash> + patch 路径
- **→ verdict 文件**:`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/R<NN>-H<NN>.md`(完整证据/测试清单/各子轮 D/R verdict 在此)
```

- [ ] **Step 3: 验证 log-template.md**

Run: `grep -nE "由 Diagnostician 在 Reviewer 批准后写|状态变化.*H<NN> →" references/log-template.md`
Expected: 无输出(旧 A 作者与 A 内状态变化已移除)。

Run: `grep -n "由 \*\*Reviewer\*\* 确认每个节点后追加" references/log-template.md`
Expected: 1 处命中。

- [ ] **Step 4: Commit**

```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" add references/log-template.md
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" commit -m "$(printf 'refactor(log-template): A 由 Reviewer 按节点追加,状态归 B\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 5: README.md — 行为层,去机制,指向 SKILL.md

**Files:**
- Modify: `README.md`(§70、§72)

- [ ] **Step 1: 改 §70 —— 指令转达去 SendMessage**

old_string:
```
- **任一时刻只有一个 Diagnostician、一个 Reviewer**：「串行」指不并发多个假设，**不等于阻塞主 agent**——子 agent 用 `run_in_background: true` 起，工作期间主 agent 仍可与你对话（状态提问直接答；可执行指令 buffer 到子 agent 返回时经 `SendMessage` 转达，本轮不中途打断）。
```
new_string:
```
- **任一时刻只有一个 Diagnostician、一个 Reviewer**：「串行」指不并发多个假设，**不等于阻塞主 agent**——子 agent 用 `run_in_background: true` 起，工作期间主 agent 仍可与你对话（状态提问直接答；可执行指令 buffer 到子 agent 返回后并入下一次重起，本轮不中途打断）。
```

- [ ] **Step 2: 改 §72 —— 轮内交接去 SendMessage,指向 SKILL.md**

old_string:
```
- **轮内交接**用 `SendMessage` 续同一实例（让它记得刚做了什么）；跨轮才换新。
```
new_string:
```
- **轮内交接**：每次都重起全新实例、靠当轮 verdict 文件重建上下文；Reviewer 收尾写 log A + commit（机制权威定义见 SKILL.md「重起交接」）。
```

- [ ] **Step 3: 验证 README.md**

Run: `grep -nE "SendMessage|续同一实例" README.md`
Expected: 无输出。

- [ ] **Step 4: Commit**

```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" add README.md
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" commit -m "$(printf 'docs(README): 轮内交接去 SendMessage,指向 SKILL.md\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 6: DESIGN.md — 追加 2026-06-19 超代决议(不改历史)

**Files:**
- Modify: `DESIGN.md`(§163–165 旧决议加超代标记;§9 末追加新决议)

> 理由:DESIGN.md §9 是**带日期的决议日志**。直接改写 2026-06-13 那条会篡改历史。做法 = 旧条加一行超代标记 + 末尾追加 2026-06-19 新条。

- [ ] **Step 1: 在 2026-06-13 SendMessage 决议块末加超代标记**

old_string:
```
> 补充决议(执行 Task 1 期间):**subagent 生命周期 = 每轮回收 + 轮内整改保持同一实例**。
> 每轮起全新一对 D/R(靠 board+log 重建状态,反锚定、上下文有界);轮内 D→R→整改→D 用 SendMessage
> 续同一实例;本轮 log 写完、board 更新后一起回收,下一轮全新一对。已写入 SKILL.md(架构/单轮协议/Red Flags)。
```
new_string:
```
> 补充决议(执行 Task 1 期间):**subagent 生命周期 = 每轮回收 + 轮内整改保持同一实例**。
> 每轮起全新一对 D/R(靠 board+log 重建状态,反锚定、上下文有界);轮内 D→R→整改→D 用 SendMessage
> 续同一实例;本轮 log 写完、board 更新后一起回收,下一轮全新一对。已写入 SKILL.md(架构/单轮协议/Red Flags)。
> **⚠️ 已被 2026-06-19 决议取代(本环境无 SendMessage):改「重起交接」+ 角色重划,见本节末。**
```

- [ ] **Step 2: 在 §9 决议块末尾(文件相应位置)追加 2026-06-19 决议**

old_string:
```
> 已写入 SKILL.md(Iron Law 8 / 单轮协议第0步+第1步 / 新增「用户检查点」章节 / Red Flags)。
>
```
new_string:
```
> 已写入 SKILL.md(Iron Law 8 / 单轮协议第0步+第1步 / 新增「用户检查点」章节 / Red Flags)。
>
> 补充决议(2026-06-19,取代上「轮内保持同一实例」):**本 harness 不提供 SendMessage**,idle 实例无法二次激活。改为——轮内每次 handoff 走**「重起交接」**(全新 spawn + 注入 verdict 路径/角色节点/诉求,子 agent 先读 verdict 恢复);**角色重划**:Diagnostician 纯生产者(只写 verdict),Reviewer 本轮收尾者(核验后写 log A + commit diag/fix/回退),主 agent 写 log B + commit board/log。详见 specs/2026-06-19-sendmessage-free-respawn-design.md。已写入 SKILL.md(架构「重起交接」/ §80·§83·§121 指针 / step5·6 / Iron Law10)+ briefs + log-template + README。
>
```

- [ ] **Step 3: 验证 DESIGN.md**

Run: `grep -nE "2026-06-19|重起交接|已被 2026-06-19 决议取代" DESIGN.md`
Expected: 至少 3 处命中(超代标记 + 新决议含日期与机制名)。

- [ ] **Step 4: Commit**

```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" add DESIGN.md
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" commit -m "$(printf 'docs(DESIGN): 追加 2026-06-19 超代决议(重起交接)\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 7: 全局验证 + DRY 复核

**Files:** 无修改,仅校验。

- [ ] **Step 1: 范围内文件无残留 SendMessage/续实例**

Run:
```bash
grep -rnE "SendMessage|续同一实例|续聊" SKILL.md README.md DESIGN.md references/
```
Expected: 唯一允许命中 = DESIGN.md 里 2026-06-13 历史决议原文(含"用 SendMessage 续同一实例")及其超代标记上下文。其余文件 0 命中。逐条核对命中行确属该历史块。

- [ ] **Step 2: DRY —— 「重起交接」机制只定义一次**

Run: `grep -rn "prompt 注入三样" SKILL.md README.md references/`
Expected: 仅 SKILL.md §24 命中 1 次(机制定义体)。其余文件只「引用」名词「重起交接」,不复述注入细节。

Run: `grep -rcn "重起交接" SKILL.md README.md references/diagnostician-brief.md references/reviewer-brief.md`
Expected: SKILL.md 多次(定义+指针);README ≤1(指针);briefs 各 0 或仅名词引用——确认无机制复述。

- [ ] **Step 3: 角色一致性 —— commit/写 log 归属单一**

Run: `grep -rnE "commit|提交|写 log|log A|log B" SKILL.md references/`
人工核对:commit/回退只出现在「Reviewer 收尾者」语境;log A 只 Reviewer 写、log B 只主 agent 写;Diagnostician 处只有「不写 log、不碰 git」。无相互矛盾。

- [ ] **Step 4: 整体诊断终态自查(若仓启用了任何 lint/skill 校验则跑;否则跳过并说明)**

Run(若存在):`ls .github/ 2>/dev/null; ls *.json 2>/dev/null | grep -i skill`
Expected: 本 skill 无自动化测试;Step 1–3 的 grep 即验收标准。记录"无 lint,grep 验收通过"。

- [ ] **Step 5: 分支状态确认(不 push、不合 main)**

Run:
```bash
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" log --oneline main..HEAD
git -C "C:/Users/wenjie/.agents/skills/hypo-driven-ps" status --short
```
Expected: 6 个改动 commit(Task1–6)在 `sendmessage-free-respawn` 分支领先 main;工作树干净。**不 push、不合并到 main**——按用户 git 规范须口头确认。

---

## 自检(writing-plans 收尾)

- **Spec 覆盖**:① 重起交接定义→T1S1;② 三指针→T1S3/S4/S8;③ step5 改派→T1S5;④ step6 B+commit→T1S6;⑤ Iron Law10 执行者→T1S2;⑥ Diagnostician 去 log/git→T2;⑦ Reviewer 收尾者→T3;⑧ log A/B→T4;⑨ README→T5;⑩ DESIGN 不改历史只追加→T6;⑪ attempt 追加→T2S2;⑫ 全局去残留 + DRY→T7。无遗漏。
- **占位符**:无 TBD/TODO;每步给出确切 old/new 全文与 grep 命令。
- **一致性**:commit 信息函数名/路径一致;「重起交接」「收尾者」「log A/B」全程同名。
