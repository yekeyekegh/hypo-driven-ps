# hypo-driven-ps 课题文件夹收拢 + proofs 证据夹 + 子 agent 异步交互 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 hypo-driven-ps 的每课题产物收进单一 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/` 文件夹、新增 `<topic>-proofs/` 证据夹、并把子 agent 改为非阻塞 spawn 以支持运行期用户交互。

**Architecture:** 纯文档/prompt 改动——编辑 skill 源仓库 `~/.claude/skills/hypo-driven-ps` 的 `SKILL.md` 与 `references/*.md`。两类改动:(1) **路径迁移** = 对字面量 `docs/hypo-driven-ps/<topic>-` 做 `replace_all` 加日期文件夹层 + 对裸 sibling 引用做显式 Edit;(2) **结构性改动** = spawn 改 `run_in_background: true`、新增「运行期异步交互」小节、proofs 规则、续跑 glob 探测、Iron Law/Red Flag 同步。无代码测试套件,故每步「验证」用 `Grep`/通读充当红/绿。

**Tech Stack:** Markdown skill 文件;Edit(含 replace_all)、Grep、git。

---

## File Structure

| 文件 | 改动 |
|---|---|
| `SKILL.md` | 路径迁移 + 产物 section 重写(加 proofs)+ 续跑 glob + spawn background + 新增异步交互小节 + Iron/Red 同步 |
| `references/diagnostician-brief.md` | 路径迁移 + 裸引用 + proofs 写入规则 + buffer-relay 措辞 |
| `references/reviewer-brief.md` | 裸引用迁移 + proofs 查看说明 |
| `references/verdict-template.md` | 路径迁移 + 裸引用 + 证据字段加 proofs |
| `references/board-template.md` | 裸引用迁移 + proofs 提及 |
| `references/log-template.md` | 裸引用迁移 |
| `README.md` | (可选,超出 spec 6 文件清单) 同步布局与「串行」描述 |

**路径占位约定**:全程用占位符 `<yyyy-mm-dd>`(创建日)与 `<topic>`(课题名),与现有 `<topic>` 占位风格一致;运行时由 agent 替换为真实值。proofs 子层用 `R<NN>-H<NN>`(**连字符**,与 verdicts 一致)。

---

## Task 1: 在 skill 仓库建工作分支

**Files:** 无(git 操作)

- [ ] **Step 1: 切到 skill 仓库并建分支**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && git checkout -b feat/topic-folder-proofs-async && git status
```
Expected: `Switched to a new branch 'feat/topic-folder-proofs-async'`,工作树干净(spec 已在 main 提交)。

> 注:本仓库当前在 `main`。仅本地分支/提交,**不 push、不合并 main**(全局规范要求 push/合并须口头确认)。

---

## Task 2: SKILL.md 路径迁移

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: 迁移前确认有 5 处全路径占位**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c 'docs/hypo-driven-ps/<topic>-' SKILL.md
```
Expected: `5`

- [ ] **Step 2: replace_all 全路径加日期文件夹层**

用 Edit 工具,`replace_all: true`,对 `SKILL.md`:
- old_string: `docs/hypo-driven-ps/<topic>-`
- new_string: `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-`

- [ ] **Step 3: 验证已无无日期全路径,且新路径出现 5 处**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && echo "stale:"; grep -c 'docs/hypo-driven-ps/<topic>-' SKILL.md; echo "dated:"; grep -c 'docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-' SKILL.md
```
Expected: `stale: 0`、`dated: 5`

---

## Task 3: SKILL.md 产物 section 重写(加 proofs + 文件夹布局)

**Files:**
- Modify: `SKILL.md`(承 Task 2 后)

- [ ] **Step 1: 重写「产物」整节**

用 Edit:
- old_string:
```
## 产物:两份文件(入 docs/,提交进 Git)

- **假设板** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-board.md` —— 当前状态快照,主 agent 每轮重写。结构见 `references/board-template.md`。
- **迭代日志** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-log.md` —— append-only,只追加不改写。结构见 `references/log-template.md`。

> 注:此处有意覆盖"临时文件 `_tmp_` 前缀、不入库"的全局规范——这是可追溯的诊断档案,不是临时脚本。
```
- new_string:
```
## 产物:课题文件夹(入 docs/,提交进 Git)

每个课题的全部产物收进**单一文件夹** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/`(`<yyyy-mm-dd>` = 课题**创建日**,定死不改;`hypo-driven-ps/` 下可并存多个课题文件夹,不再散落):

- **假设板** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-board.md` —— 当前状态快照,主 agent 每轮重写。结构见 `references/board-template.md`。
- **迭代日志** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-log.md` —— append-only,只追加不改写。结构见 `references/log-template.md`。
- **verdict 目录** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/` —— 每轮 `R<NN>-H<NN>.md` 文字结论。
- **探针目录** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/` —— 诊断探针测试代码 + `failed-fixes/*.patch`。
- **证据目录** `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-proofs/` —— 截图/图片/抓取等**非测试视觉证据**。**铁律**:其下**不得直接放文件**,所有证据必须落 `R<NN>-H<NN>/` 子文件夹(连字符,与 verdicts 一致),该层下**建议**再按用途分子文件夹(如 `R02-H03/登录失败截图/step1.png`)。

> 注:此处有意覆盖"临时文件 `_tmp_` 前缀、不入库"的全局规范——这是可追溯的诊断档案,不是临时脚本。board/log/verdicts/probes/proofs 全部随课题文件夹提交进 Git。
```

- [ ] **Step 2: 验证 proofs 与布局已写入**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c '证据目录' SKILL.md; grep -c '不得.*直接放文件' SKILL.md
```
Expected: 两行均 `1`

---

## Task 4: SKILL.md 续跑 glob 探测 + 目录建立

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: worktree 门禁列出 proofs**

用 Edit:
- old_string: `走 `using-git-worktrees` 确保隔离工作树,**所有产物(board/log/verdicts/probes)落在 worktree 内**,根除跨 session/任务的工作树串台。`
- new_string: `走 `using-git-worktrees` 确保隔离工作树,**所有产物(board/log/verdicts/probes/proofs,均在课题文件夹 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/` 下)落在 worktree 内**,根除跨 session/任务的工作树串台。`

- [ ] **Step 2: 续跑探测改 glob + 多命中报警**

用 Edit:
- old_string: `   - worktree 里**已有** `<topic>-board.md` → **续跑**:主 agent 读**最新 board**(整盘假设+状态+排序)+ **log**(跨轮 why)重建框架;`工作语言` 读 board 头;**round 续号 = board 头「最新轮号」+1(不 reset)**;**可选派基线 Diagnostician 重核"代码现实"**(代码是否漂移、上轮修复是否还在)。`
- new_string: `   - **课题文件夹定位(按课题名 glob,与日期无关)**:在 worktree 的 `docs/hypo-driven-ps/` 下 glob `*-<topic>/<topic>-board.md` —— **唯一命中** → **续跑**该文件夹(日期前缀沿用、**不变**);**多命中**(同名课题多次创建)→ 停下报警、列出候选文件夹交用户选,**不擅自挑**;**无命中** → 全新,用**当天日期**建 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/`。续跑时主 agent 读**最新 board**(整盘假设+状态+排序)+ **log**(跨轮 why)重建框架;`工作语言` 读 board 头;**round 续号 = board 头「最新轮号」+1(不 reset)**;**可选派基线 Diagnostician 重核"代码现实"**(代码是否漂移、上轮修复是否还在)。`

- [ ] **Step 3: 第 3 步建目录加 probes/proofs**

用 Edit:
- old_string: `3. 用模板建立/更新 board、log、`<topic>-verdicts/` 目录 → 选本轮 Hypo,写「当前轮计划」→ **过【用户检查点】(见下节)**。`
- new_string: `3. 用模板在课题文件夹 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/` 下建立/更新 board、log、`<topic>-verdicts/`、`<topic>-probes/`、`<topic>-proofs/` → 选本轮 Hypo,写「当前轮计划」→ **过【用户检查点】(见下节)**。`

- [ ] **Step 4: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c '按课题名 glob' SKILL.md; grep -c 'board/log/verdicts/probes/proofs' SKILL.md
```
Expected: 两行均 `≥1`

---

## Task 5: SKILL.md 子 agent 非阻塞 spawn

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: 架构「串行」澄清(不阻塞主 agent)**

用 Edit:
- old_string: `任一时刻**只有一个 Diagnostician、一个 Reviewer,串行进行**。不并发多个假设。`
- new_string: `任一时刻**只有一个 Diagnostician、一个 Reviewer**——「串行」指**不并发多个假设**(每轮靠上一轮 learning 重排整盘),**不等于阻塞主 agent**:spawn 用 `run_in_background: true`,子 agent 工作期间主 agent 仍可与用户对话(见「运行期异步交互」)。`

- [ ] **Step 2: Diagnostician spawn 改 background**

用 Edit:
- old_string: `、**串行(不 run_in_background)**;prompt = `references/diagnostician-brief.md``
- new_string: `、**非阻塞 spawn(`run_in_background: true`)——仍单 subagent(不并发多假设),但主 agent 这一回合就此解阻塞、运行期可与用户对话(见「运行期异步交互」)**;prompt = `references/diagnostician-brief.md``

- [ ] **Step 3: Reviewer spawn 同样 background**

用 Edit:
- old_string: `**spawn Reviewer(opus,每轮全新)— 节点1 核诊断**:同样三参 + **`工作语言`**;`
- new_string: `**spawn Reviewer(opus,每轮全新)— 节点1 核诊断**:同样三参 + **`工作语言`**(同样 `run_in_background: true`);`

- [ ] **Step 4: 验证旧措辞已除**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && echo "old:"; grep -c '串行(不 run_in_background)' SKILL.md; echo "new:"; grep -c 'run_in_background: true' SKILL.md
```
Expected: `old: 0`、`new: ≥3`

---

## Task 6: SKILL.md 新增「运行期异步交互」小节

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: 在「用户检查点」节之后、「修复成本闸门」节之前插入新小节**

用 Edit(锚定「修复成本闸门」标题,在其前插入):
- old_string: `## 修复成本闸门(确诊后)`
- new_string:
```
## 运行期异步交互(子 agent 工作时)

子 agent 用 `run_in_background: true` 起,**主 agent 这一回合不被阻塞**——子 agent 跑测试/取证期间,用户随时可与主 agent 对话。规则:

1. **主 agent 运行期仍不下场**:不读源码、不跑测试、不亲自取证(Iron Law #1 不变)。它只回答用户关于假设板的问题、接收用户指令。
2. **buffer-and-relay(送达子 agent 的唯一通道)**:
   - **纯状态提问**(如"现在第几轮""H3 验得怎样")→ 主 agent **直接答**,不入队。
   - **与当前子 agent 任务相关的可执行指令** → 主 agent **即时记下(buffer)**,**待该子 agent 返回 digest 时,先用 `SendMessage` 续同一实例转达,再走下一协议步**。
   - **改排序 / 增删假设 / 改判据类** → 并入下一轮「回灌重排(第 6 步)」。
   - **不中途打断**:本轮原子完成,无中途 `TaskStop`。已知代价:若用户指令是"方向错了/该换题",当前轮仍跑到完成,redirect 在其返回后才生效。
3. **完成通知 + 区分激活来源**:子 agent 完成即重新激活主 agent。主 agent 须分清本次激活是「子 agent 完成」(→ 走推进协议:核诊断/转修复/写板)还是「用户插话」(→ 走上面的答复/buffer),**不可把用户输入误当子 agent 结论,或反之**。

> pre-spawn 的【用户检查点】门禁**不受影响**——它发生在 spawn *之前*;background 改变的只是 spawn *之后*、子 agent 运行*期间*主 agent 的可用性。

## 修复成本闸门(确诊后)
```

- [ ] **Step 2: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c '运行期异步交互' SKILL.md; grep -c 'buffer-and-relay' SKILL.md
```
Expected: `运行期异步交互` ≥ `2`(标题 + 交叉引用)、`buffer-and-relay` ≥ `1`

---

## Task 7: SKILL.md 测试持久化加 proofs + Red Flags 同步

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: 测试持久化区加视觉证据落点**

用 Edit:
- old_string: `- **诊断验证测试(探针)** → 存到 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/`,在 log 证据里记路径。下一轮全新 Diagnostician 通过 log 找到、复用/扩展,**不从零写**;Reviewer 可重跑。`
- new_string: `- **诊断验证测试(探针)** → 存到 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/`,在 log 证据里记路径。下一轮全新 Diagnostician 通过 log 找到、复用/扩展,**不从零写**;Reviewer 可重跑。
- **视觉/二进制证据(截图、图片、抓取产物)** → 存到 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-proofs/R<NN>-H<NN>/<用途>/`(**不得**直接放 proofs 根),在 verdict 证据里记路径。`

- [ ] **Step 2: 追加 Red Flags**

用 Edit(锚定现有最后一条 Red Flag,追加新条):
- old_string: `- 把完整 verdict 堆进主 agent 上下文 / 塞进 log(应:subagent 写 verdict 文件 + 只回 digest;log 只留摘要+指针)。`
- new_string: `- 把完整 verdict 堆进主 agent 上下文 / 塞进 log(应:subagent 写 verdict 文件 + 只回 digest;log 只留摘要+指针)。
- 产物散落在 `docs/hypo-driven-ps/` 根,而非课题文件夹 `<yyyy-mm-dd>-<topic>/` 下;或 proofs 目录下直接放文件,没归到 `R<NN>-H<NN>/` 子文件夹。
- 子 agent 运行期主 agent 自己下场读码/跑测试/取证(应只答疑 + buffer,不下场)。
- 用户运行期给的可执行指令没 buffer、丢了;或没等子 agent 返回就硬中途打断(本设计不做中途 TaskStop)。
- 把子 agent「完成激活」与「用户插话激活」搞混(误把用户输入当子 agent 结论,或反之)。`

- [ ] **Step 3: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c '视觉/二进制证据' SKILL.md; grep -c '完成激活.*用户插话激活' SKILL.md
```
Expected: 两行均 `≥1`

- [ ] **Step 4: 通读 SKILL.md 一致性自检**

Read `SKILL.md` 全文,确认:路径全部带 `<yyyy-mm-dd>-<topic>/`;proofs 规则、异步小节、Red Flags 自洽,无残留「串行(不 run_in_background)」。

- [ ] **Step 5: 提交 SKILL.md**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && git add SKILL.md && git commit -m "feat(skill): 课题文件夹收拢 + proofs 证据夹 + 子agent非阻塞spawn

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```
Expected: 1 file changed。

---

## Task 8: diagnostician-brief.md

**Files:**
- Modify: `references/diagnostician-brief.md`

- [ ] **Step 1: 路径迁移(replace_all,3 处全路径)**

用 Edit,`replace_all: true`,对 `references/diagnostician-brief.md`:
- old_string: `docs/hypo-driven-ps/<topic>-`
- new_string: `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-`

- [ ] **Step 2: 裸 sibling 引用改全路径**

用 Edit:
- old_string: `- 两份文件路径:`<topic>-board.md`(只读参考)、`<topic>-log.md`、当轮 `<topic>-verdicts/R<NN>-H<NN>.md`。`
- new_string: `- 课题文件夹路径 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/`,内含:`<topic>-board.md`(只读参考)、`<topic>-log.md`、当轮 `<topic>-verdicts/R<NN>-H<NN>.md`。`

- [ ] **Step 3: buffer-and-relay 措辞**

用 Edit:
- old_string: `轮内若收到 Reviewer 整改意见,你会被续聊继续(那时你记得自己刚做了什么)。`
- new_string: `轮内若收到 Reviewer 整改意见,你会被续聊继续(那时你记得自己刚做了什么);主 agent 也可能在你返回后用 `SendMessage` 转达用户运行期给的指令——照常续做。`

- [ ] **Step 4: 节点1 加 proofs 写入规则**

用 Edit:
- old_string: `   - 诊断验证测试(探针)**持久化**到 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/`,路径写进 verdict。`
- new_string: `   - 诊断验证测试(探针)**持久化**到 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/`,路径写进 verdict。
   - **视觉/二进制证据**(截图、图片、抓取产物)**持久化**到 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-proofs/R<NN>-H<NN>/<用途>/`(**不得**直接放 proofs 根;`R<NN>-H<NN>` 用连字符;路径写进 verdict 证据)。`

- [ ] **Step 5: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && echo "stale:"; grep -c 'docs/hypo-driven-ps/<topic>-' references/diagnostician-brief.md; echo "bare:"; grep -c '`<topic>-board.md`(只读' references/diagnostician-brief.md; echo "proofs:"; grep -c 'proofs/R<NN>-H<NN>' references/diagnostician-brief.md
```
Expected: `stale: 0`、`bare: 0`、`proofs: ≥1`

---

## Task 9: reviewer-brief.md

**Files:**
- Modify: `references/reviewer-brief.md`

- [ ] **Step 1: 裸 verdicts 引用改全路径**

用 Edit:
- old_string: `- Diagnostician 写好的 **verdict 节**(在当轮 `<topic>-verdicts/R<NN>-H<NN>.md`:节点1=`diagnostic` / 节点2=`repair`;格式见 `references/verdict-template.md`)。`
- new_string: `- Diagnostician 写好的 **verdict 节**(在当轮 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/R<NN>-H<NN>.md`:节点1=`diagnostic` / 节点2=`repair`;格式见 `references/verdict-template.md`)。`

- [ ] **Step 2: 加 proofs 查看说明**

用 Edit:
- old_string: `- **工作语言**:用主 agent 注入的「工作语言」产出你的 verdict 与沟通;引用的英文 skill **理解用英文、产出用工作语言**。`
- new_string: `- **工作语言**:用主 agent 注入的「工作语言」产出你的 verdict 与沟通;引用的英文 skill **理解用英文、产出用工作语言**。
- verdict 若引用 `<topic>-proofs/R<NN>-H<NN>/…` 下的视觉证据(截图等),**按需打开核对**,不可只看文字声称。`

- [ ] **Step 3: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && echo "bare:"; grep -c '在当轮 `<topic>-verdicts/' references/reviewer-brief.md; echo "dated:"; grep -c 'docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/' references/reviewer-brief.md; echo "proofs:"; grep -c 'proofs/R<NN>-H<NN>' references/reviewer-brief.md
```
Expected: `bare: 0`、`dated: ≥1`、`proofs: ≥1`

---

## Task 10: verdict-template.md

**Files:**
- Modify: `references/verdict-template.md`

- [ ] **Step 1: 路径迁移(replace_all,2 处全路径)**

用 Edit,`replace_all: true`:
- old_string: `docs/hypo-driven-ps/<topic>-`
- new_string: `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-`

- [ ] **Step 2: 裸 failed-fixes 引用改全路径**

用 Edit:
- old_string: `第 3 次失败 → 回退到修复前 + 存档 patch 路径(`<topic>-probes/failed-fixes/R<NN>-H<NN>.patch`)。`
- new_string: `第 3 次失败 → 回退到修复前 + 存档 patch 路径(`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-probes/failed-fixes/R<NN>-H<NN>.patch`)。`

- [ ] **Step 3: diagnostic 证据字段加 proofs**

用 Edit:
- old_string: `- **证据**:命令 / 完整输出 / 退出码 / 复现步骤。`
- new_string: `- **证据**:命令 / 完整输出 / 退出码 / 复现步骤;视觉证据(截图等)写其 `<topic>-proofs/R<NN>-H<NN>/<用途>/…` 路径。`

- [ ] **Step 4: repair 证据字段加 proofs**

用 Edit:
- old_string: `- **证据**:问题不再复现的命令+输出+退出码;全套测试绿、无回归。`
- new_string: `- **证据**:问题不再复现的命令+输出+退出码;全套测试绿、无回归;视觉证据(截图等)写其 `<topic>-proofs/R<NN>-H<NN>/<用途>/…` 路径。`

- [ ] **Step 5: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && echo "stale:"; grep -c 'docs/hypo-driven-ps/<topic>-' references/verdict-template.md; echo "bare-patch:"; grep -c '`<topic>-probes/failed-fixes' references/verdict-template.md; echo "proofs:"; grep -c 'proofs/R<NN>-H<NN>' references/verdict-template.md
```
Expected: `stale: 0`、`bare-patch: 0`、`proofs: ≥2`

---

## Task 11: board-template.md

**Files:**
- Modify: `references/board-template.md`

- [ ] **Step 1: 裸 sibling 引用改全路径 + 加 proofs**

用 Edit:
- old_string: `> 配套日志:`<topic>-log.md`(append-only);细粒度证据:`<topic>-verdicts/R<NN>-H<NN>.md`。`
- new_string: `> 课题文件夹:`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/`;配套日志:`<topic>-log.md`(append-only);细粒度证据:`<topic>-verdicts/R<NN>-H<NN>.md`;视觉证据:`<topic>-proofs/R<NN>-H<NN>/`。`

- [ ] **Step 2: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c '课题文件夹:`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/`' references/board-template.md; grep -c '视觉证据:`<topic>-proofs' references/board-template.md
```
Expected: 两行均 `1`

---

## Task 12: log-template.md

**Files:**
- Modify: `references/log-template.md`

- [ ] **Step 1: 裸 verdicts 指针改全路径**

用 Edit:
- old_string: `- **→ verdicts 文件**:`<topic>-verdicts/R<NN>-H<NN>.md`(完整证据/测试清单/各子轮 D/R verdict 在此)`
- new_string: `- **→ verdicts 文件**:`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/R<NN>-H<NN>.md`(完整证据/测试清单/各子轮 D/R verdict 在此)`

- [ ] **Step 2: 验证**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c 'docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/R<NN>-H<NN>.md' references/log-template.md
```
Expected: `1`

- [ ] **Step 3: 提交全部 references 改动**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && git add references/ && git commit -m "feat(skill): references 同步课题文件夹路径 + proofs 规则 + buffer-relay

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```
Expected: 5 files changed。

---

## Task 13: 全仓库一致性终检

**Files:** 无(只读校验)

- [ ] **Step 1: 运行时文件无残留无日期路径**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -rn 'docs/hypo-driven-ps/<topic>-' SKILL.md references/
```
Expected: 无输出(0 命中)。

- [ ] **Step 2: 运行时文件无残留裸 sibling 引用**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -rnE '[^/]`<topic>-(board|log|verdicts|probes|proofs)' SKILL.md references/ | grep -v 'yyyy-mm-dd'
```
Expected: 无输出(所有 `<topic>-xxx` 引用都带 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/` 前缀;board-template 的同行 sibling 简写因已带「课题文件夹:」前缀说明,可接受——若此处有命中,逐条确认是否在已声明课题文件夹的上下文内)。

- [ ] **Step 3: 旧 spawn 措辞清零**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -rn '串行(不 run_in_background)' SKILL.md references/
```
Expected: 无输出。

---

## Task 14(可选,超出 spec 6 文件清单): README.md 同步

> spec 显式只列 6 个运行时文件;README.md 是面向用户的说明,若不同步会与新行为矛盾。DESIGN.md / PLAN.md 为历史设计快照,**保持不动**。**执行前向用户确认是否纳入。**

**Files:**
- Modify: `README.md`

- [ ] **Step 1: 读 README 找出布局/串行/路径描述段**

Read `README.md`,定位:目录布局说明、第 70 行「任一时刻只有一个 Diagnostician、一个 Reviewer，串行」、以及任何 `<topic>-board.md` 等路径引用(grep 第二条命中过 3 处)。

- [ ] **Step 2: 同步布局为课题文件夹 + proofs**

将路径描述更新为 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/` 课题文件夹布局(含 verdicts/probes/proofs 及 proofs 的 `R<NN>-H<NN>/` 归档铁律)。具体 old/new 依 Step 1 实读内容确定(本任务可选,执行者按实读文本逐段替换,保持与 SKILL.md 一致措辞)。

- [ ] **Step 3: 同步「串行」描述**

将「串行，不并发多个假设」补一句:`run_in_background: true` 非阻塞 spawn、运行期主 agent 可与用户交互,与 SKILL.md 「运行期异步交互」一致。

- [ ] **Step 4: 验证 + 提交**

Run:
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && grep -c 'docs/hypo-driven-ps/<topic>-' README.md; grep -c 'proofs' README.md
```
Expected: 第一行 `0`(无无日期路径)、第二行 `≥1`。
```bash
cd /c/Users/wenjie/.claude/skills/hypo-driven-ps && git add README.md && git commit -m "docs(readme): 同步课题文件夹布局 + proofs + 异步交互

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Self-Review(plan 作者已核)

- **Spec 覆盖**:变更1(收拢)= Task 2/3/4/8-12;变更2(proofs)= Task 3/7/8/9/10/11;变更3(异步)= Task 5/6 + brief 措辞(Task 8/9);续跑 glob = Task 4;worktree 名不变 = 未动(spec 要求);终检 = Task 13。全部 spec 节有对应任务。
- **占位一致性**:全程 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-xxx`;proofs 子层 `R<NN>-H<NN>/`(连字符)统一;`run_in_background: true` 措辞统一。
- **无 placeholder**:每个编辑步给出确切 old/new 字符串;唯 Task 14(可选、超 spec)的 README 因需实读内容,Step 2 留给执行者按实读逐段替换——已显式标注为可选且需用户确认。
- **范围**:DESIGN.md/PLAN.md 历史快照不动;README 可选并需确认。符合 spec「其它未触及保持原样」。
