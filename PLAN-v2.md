# hypo-driven-ps v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给已建好的 hypo-driven-ps 加 v2 增量——worktree 隔离门禁、verdicts 细粒度持久化层(与 log 分层)、跨 session 续跑、工作语言锚定、收尾合并;并把 handoff-template 合并进新的 verdict-template。

**Architecture:** 全 markdown 改动(SKILL.md + references/),无运行时代码,"测试"= 结构校验(grep 章节/字段、引用不悬空、无残留旧引用)。设计权威源 = `DESIGN.md`「v2 增量设计」节。

**Tech Stack:** Markdown、Claude Code Skill 约定、Agent 工具(spawn opus subagent + 注入工作语言)、复用 superpowers 的 using-git-worktrees / finishing-a-development-branch / test-driven-development / verification-before-completion。

**前置约定:**
- 根目录 `C:\Users\wenjie\.agents\skills\hypo-driven-ps\`(下称 `<ROOT>`)。
- 仓库在 **main**,**每个 Task 末尾 commit 在 main、不推送**(用户已定)。commit 讯息末尾加 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。
- 这些是 skill 的**运行时**约定(worktree/commit/语言),改的是 skill 文本本身;本 plan 自己的执行就在 `<ROOT>` 仓库做。

---

## File Structure(v2 后)

```
<ROOT>/
  SKILL.md                          # 改:第0步(worktree门禁+续跑+triage)、verdicts/log 分层、spawn 注入语言、收尾、Iron Law 12、Red Flags、交付单→verdict 节
  references/
    board-template.md               # 改:头部加 工作语言 + 最新轮号
    log-template.md                 # 改:A 条目瘦身为 摘要 + verdicts 指针
    verdict-template.md             # 新增:R<NN>-H<NN>.md 文件模板(4 类节)
    diagnostician-brief.md          # 改:写全文 verdict+回 digest、工作语言、续跑重核、交付单→verdict 节
    reviewer-brief.md               # 改:写 *-review verdict+digest、工作语言、交付单→verdict 节
    handoff-template.md             # 删除(并入 verdict-template)
    (root-cause-tracing / condition-based-waiting / defense-in-depth + 2 配套:不变)
  DESIGN.md / PLAN.md / PLAN-v2.md
```

运行时新增产物(在目标 worktree 内,非本仓库):`docs/hypo-driven-ps/<topic>-verdicts/R<NN>-H<NN>.md`、`<topic>-probes/`。

---

## Task 1: 新增 references/verdict-template.md

**Files:**
- Create: `<ROOT>/references/verdict-template.md`

- [ ] **Step 1: 写文件,内容如下**

````markdown
# verdict 文件模板:`R<NN>-H<NN>.md`

> 位置:`docs/hypo-driven-ps/<topic>-verdicts/`,**一轮一文件**(round 号在前、H 号在后,各补零两位,如 `R01-H02.md`)。
> 由**产出 verdict 的 subagent 写全文**(Diagnostician 写 diagnostic/repair 节,Reviewer 写 *-review 节);同时**只回 1–2 行 digest 给主 agent**。
> **子轮号 y**:仅"重新诊断"(同一轮内换角度/重诊)才 y+1;整改重做、修复重试(第2/3次)**留同一 y**,用 `attempt K` 标记。
> 所有文字用 **board 头部「工作语言」**。

## 文件头

- **轮次**:R<NN> · **假设**:H<NN> — <描述>
- **验证的判据**:<本轮要验的那条>

---

## R<NN>-H<NN>.<y> diagnostic  〔attempt K — 仅整改重做时标〕

- **做了什么 / 测了什么**:对照判据 <…>,手段 <命令 / 插桩 / 复现脚本>
- **测试清单**:

  | test 路径 | 测什么 | 运行命令 | 红/绿 |
  |---|---|---|---|
  | docs/hypo-driven-ps/<topic>-probes/… | … | … | … |
- **完整性论证**:为何这组 test 足以判定该判据(覆盖哪些情形、为何没漏)。
- **证据**:命令 / 完整输出 / 退出码 / 复现步骤。
- **结论**:判据成立(真) / 证伪(假) / 无法判断(写明哪条判据测不了、为何)。
- **digest(回主 agent,1–2 行)**:<结论 + 关键证据一句话>

## R<NN>-H<NN>.<y> diagnostic-review

- **读懂的 test + 重跑结果**:<逐个 test 实际断言了什么、重跑是否与上面一致>
- **判定**:批准 / 整改意见(具体缺什么、哪里对不上、要补跑什么)。
- **digest**:<批准/整改 + 一句话理由>

## R<NN>-H<NN>.<y> repair  〔attempt K(K≤3)〕

- **修复成本评估**:小修 / 成本过大(命中信号:<哪条>;成本过大本不应到这步,除非用户决策立即修)。
- **修复方案 + 改了哪些文件**:<摘要 + 文件清单>
- **测试清单**:

  | test 路径 | 类型(复现/回归) | 运行命令 | 结果 |
  |---|---|---|---|
  | <进正式测试套件的路径> | 复现 | … | 修复前红 → 后绿 |
  | … | 回归 | … | 绿 |
- **证据**:问题不再复现的命令+输出+退出码;全套测试绿、无回归。
- **结果**:已修复并经验证解决 / 本次失败(原因) / 第 3 次失败 → 回退到修复前 + 存档 patch 路径(`<topic>-probes/failed-fixes/R<NN>-H<NN>.patch`)。
- **digest**:<结果 + 一句话>

## R<NN>-H<NN>.<y> repair-review

- **读懂修复 test + 重跑 + 跑回归**:<结果>
- **判定**:批准 / 整改(test 缺陷,不计入 3 次) / 判未解决(计入 3 次)。
- **digest**:<判定 + 一句话>
````

- [ ] **Step 2: 校验**

Run:
```bash
R="$HOME/.agents/skills/hypo-driven-ps/references"
grep -c -E "^## R<NN>-H<NN>\.<y> (diagnostic|diagnostic-review|repair|repair-review)" "$R/verdict-template.md"   # 应=4
grep -q "只回 1–2 行 digest" "$R/verdict-template.md" && grep -q '仅"重新诊断"' "$R/verdict-template.md" && echo "要点 OK"
```
Expected: 4;要点 OK。

- [ ] **Step 3: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/verdict-template.md && git commit -m "$(printf 'v2: add verdict-template (merges handoff)\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 2: board-template.md 头部加字段

**Files:**
- Modify: `<ROOT>/references/board-template.md`

- [ ] **Step 1: 在文件最上方的引用块后插入头部字段**

把开头:
```markdown
# 假设板:<topic>

> 维护者:主 agent(编排者)。每轮重写为最新快照。
> 配套日志:`<topic>-log.md`(append-only)。
```
改为:
```markdown
# 假设板:<topic>

> 维护者:主 agent(编排者)。每轮重写为最新快照。
> 配套日志:`<topic>-log.md`(append-only);细粒度证据:`<topic>-verdicts/R<NN>-H<NN>.md`。

## 0. 元信息（续跑读这里）

- **工作语言**:<首次启动时主 agent 当时语言,如 中文;全程沿用,spawn 时注入 subagent>
- **最新轮号**:R<NN>（续跑时下一轮 = 本号 +1，不 reset）
- **worktree**:<隔离工作树路径/分支>
```

- [ ] **Step 2: 校验**

Run:
```bash
R="$HOME/.agents/skills/hypo-driven-ps/references"
grep -q "## 0. 元信息" "$R/board-template.md" && grep -q "工作语言" "$R/board-template.md" && grep -q "最新轮号" "$R/board-template.md" && echo OK
```
Expected: OK。

- [ ] **Step 3: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/board-template.md && git commit -m "$(printf 'v2: board header (working language + latest round)\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 3: log-template.md A 条目瘦身

**Files:**
- Modify: `<ROOT>/references/log-template.md`

- [ ] **Step 1: 把「A. Diagnostician 轮次记录」整段替换为瘦身版**

替换为:
```markdown
## 第 N 轮 — A. Diagnostician 轮次记录（摘要,细节在 verdicts）

- **轮次/假设**:R<NN> · H<NN> — <描述>
- **最终结论**:判据成立(真) / 证伪(假) / 无法判断
- **状态变化**:H<NN> → <已确诊·已修复 / 已确诊·未启动修复 / 已确诊·修复失败 / 已排除 / 无法判断>
- **一句 learning**:<本轮最关键的一条收获>
- **提交(有 git)**:diag <hash/N/A> · fix <hash/N/A> · 若回退 fix-failed <hash> + patch 路径
- **→ verdicts 文件**:`<topic>-verdicts/R<NN>-H<NN>.md`(完整证据/测试清单/各子轮 D/R verdict 在此)
```

> 说明:原来塞在此处的「证据摘要 / 测试清单 / 各子轮细节」**下沉到 verdicts 文件**;log 只保留摘要 + 指针,保持"每轮通读全量 log"轻量。

- [ ] **Step 2: 校验**

Run:
```bash
R="$HOME/.agents/skills/hypo-driven-ps/references"
grep -q "→ verdicts 文件" "$R/log-template.md" && grep -q "细节在 verdicts" "$R/log-template.md" && echo OK
grep -q "完整输出要点 / 退出码 / 复现步骤" "$R/log-template.md" && echo "WARN: 旧证据字段仍在(应已下沉)" || echo "瘦身 OK"
```
Expected: OK;瘦身 OK。

- [ ] **Step 3: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/log-template.md && git commit -m "$(printf 'v2: slim log A-entry to summary + verdicts pointer\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 4: SKILL.md 主流程改动(v2 核心)

**Files:**
- Modify: `<ROOT>/SKILL.md`

> 共 7 处编辑(E1–E7)。每处给出锚点与新文本。

- [ ] **Step 1 (E1): 第0步重写为「worktree门禁 → 续跑/全新分流 → 基线triage」**

把现有「**第 0 步(首次,主 agent 做)**:框定问题…」整段(含其后的「基线 triage」引用块,直到「→ 生成初始假设集… → 过【用户检查点】」)替换为:

```markdown
**第 0 步(主 agent 做,首次或续跑):**

1. **worktree 门禁(最先,先于一切取证)**:用 `using-git-worktrees` 确保隔离工作树——该 topic 的 worktree **已存在 → 进入续跑;否则新建**。所有产物(board/log/verdicts/probes)落在 worktree 内。
   - **无 git → 默认拒绝启动** + 引导用户(`git init` / 换到 git 项目);**仅当用户显式坚持**才降级"裸跑",并明确告知失去 隔离 / 提交 / 回退 / 续跑 四项保障。
2. **续跑 or 全新**:
   - worktree 里**已有** `<topic>-board.md` → **续跑**:主 agent 读**最新 board**(整盘假设+状态+排序)+ **log**(跨轮 why)重建框架;`工作语言` 读 board 头;**round 续号 = board 头「最新轮号」+1(不 reset)**;**可选派基线 Diagnostician 重核"代码现实"**(代码是否漂移、上轮修复是否还在)。
   - **无既有文档 → 全新**:框定问题(症状 + 成功判据)→ 定 `工作语言`(= 主 agent 当前语言)写进 board 头 → **基线 triage**(证据稀薄时派 Diagnostician 读报错/堆栈、确认稳定复现、查最近改动)→ 生成初始假设集(R01 起)。
3. 用模板建立/更新 board、log、`<topic>-verdicts/` 目录 → 选本轮 Hypo,写「当前轮计划」→ **过【用户检查点】(见下节)**。
```

- [ ] **Step 2 (E2): spawn 步注入工作语言 + verdict 写法**

把单轮协议第 2、3 步里 `prompt = …` 的描述,各追加一句工作语言与 verdict 落盘约定。将第 2 步改为:
```markdown
2. **spawn Diagnostician(opus,每轮全新)— 诊断**:Agent 工具,`subagent_type: general-purpose`、`model: opus`、串行;prompt = `references/diagnostician-brief.md` 全文 + 当前轮计划 + 两份文件路径 + **`工作语言: <board 头取值>`**。它写测试、验证,**把完整 diagnostic verdict 写进 `<topic>-verdicts/R<NN>-H<NN>.md`(按 `verdict-template.md`),只回 1–2 行 digest 给主 agent**。不写 log。
```
将第 3 步改为:
```markdown
3. **spawn Reviewer(opus,每轮全新)— 节点1 核诊断**:同样三参 + **`工作语言`**;prompt = `references/reviewer-brief.md` 全文 + 该 verdict 文件路径 + 当前轮计划。它读懂该组 test、重跑,调 `verification-before-completion` → **把 diagnostic-review verdict 写进同一文件**,回 digest:**批准** 或 **整改意见**。
   - 整改(含 test 写错/不完整)→ `SendMessage` 续**同一个** Diagnostician 补齐/重做(同一 y,标 attempt),再续**同一个** Reviewer 复核,直到通过。
```

- [ ] **Step 3 (E3): 第4步修复 + 第5步写 log 调整为 verdict 落盘 + log 摘要**

把第 4 步的「交 **「修复交付单」** → … Reviewer **节点2 核修复**」一句改为:
```markdown
   - **小修(都不命中)或 用户决策立即修** → **同一个** Diagnostician 修(`test-driven-development` 写失败复现测试 → 最小实现),**把 repair verdict 写进当轮 verdict 文件** → **同一个** Reviewer **节点2 核修复**(`verification-before-completion`;读懂修复 test + 重跑 + 跑回归,**写 repair-review verdict**)。
```
把第 5 步改为:
```markdown
5. **Diagnostician 写 log(摘要)**:经 Reviewer 通过后,把本轮「Diagnostician 轮次记录(摘要)」按 `log-template.md` 的 A 条目**追加**到 log——只放 结论/状态变化/一句 learning/提交/**→ verdicts 文件指针**;完整证据已在 verdict 文件。
```

- [ ] **Step 4 (E4): 第6步回灌重排里,更新 board 头「最新轮号」**

把第 6 步 c 子项「把「主 agent 板更新记录」**追加**到 log,并重写 board 快照。」改为:
```markdown
   c. 把「主 agent 板更新记录」**追加**到 log,并重写 board 快照(**含更新 board 头「最新轮号」= 本轮 R<NN>**)。
```

- [ ] **Step 5 (E5): 新增 Iron Law 12(worktree)**

在 Iron Laws 末尾(第 11 条后)加:
```markdown
12. **必须在隔离 worktree 中运行**(有 git 时)—— 第0步先经 `using-git-worktrees`;无 git 默认拒启,仅用户显式坚持才降级裸跑。根除跨 session/任务的工作树串台。
```

- [ ] **Step 6 (E6): 收尾段 + Red Flags**

在「升级协议」节之后(或「最小示例」前)新增一节:
```markdown
## 收尾(问题解决 / 用户结束)

走 `finishing-a-development-branch`,把 **merge / 留 PR / 仅清理** 的选择**交用户**;**不自动 merge**(外向、难回滚)。board/log/verdicts 作为诊断档案随之处置。
```
在 Red Flags 末尾加三条:
```markdown
- 没经 worktree 门禁就在共享工作树(如 main)上跑(无 git 又没走显式降级确认)。
- 续跑时把 round 号 reset 回 R01,或没读最新 board+log 就另起炉灶。
- 把完整 verdict 堆进主 agent 上下文 / 塞进 log(应:subagent 写 verdict 文件 + 只回 digest;log 只留摘要+指针)。
```

- [ ] **Step 7 (E7): 「交付单」措辞全改为「verdict 节」**

把 SKILL.md 里「测试与交付单(两个评审节点)」节及其它处对「交付单 / handoff-template」的引用,改为指向 verdict 文件/verdict-template。具体:
- 节标题 `## 测试与交付单(两个评审节点)` → `## 测试与 verdict(两个评审节点)`。
- 节内「Diagnostician 各提交一张交付单(字段见 `references/handoff-template.md`)」→「Diagnostician/Reviewer 各写一节 verdict(字段见 `references/verdict-template.md`),写进当轮 `R<NN>-H<NN>.md`」。
- 「节点1 — 诊断交付:…提交「诊断交付单」」→「节点1 — 诊断:…写 diagnostic verdict + diagnostic-review」。
- 「节点2 — 修复交付:…提交「修复交付单」」→「节点2 — 修复:…写 repair verdict + repair-review」。
- 全文搜索 `handoff-template`,确保 SKILL.md 内无残留(除非在"已合并/删除"的说明语境)。

- [ ] **Step 8: 校验**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
echo "worktree门禁:"; grep -q "worktree 门禁(最先" "$ROOT/SKILL.md" && echo OK
echo "续跑续号:"; grep -q "round 续号 = board 头「最新轮号」+1" "$ROOT/SKILL.md" && echo OK
echo "工作语言注入:"; grep -q '工作语言: <board 头取值>' "$ROOT/SKILL.md" && echo OK
echo "verdict 落盘:"; grep -q "diagnostic verdict 写进" "$ROOT/SKILL.md" && echo OK
echo "Iron Law 12:"; grep -q "必须在隔离 worktree 中运行" "$ROOT/SKILL.md" && echo OK
echo "收尾节:"; grep -q "^## 收尾(问题解决" "$ROOT/SKILL.md" && echo OK
echo "无 handoff-template 残留引用:"; grep -c "handoff-template" "$ROOT/SKILL.md"   # 期望 0
```
Expected: 6 个 OK;handoff-template 计数 = 0。

- [ ] **Step 9: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add SKILL.md && git commit -m "$(printf 'v2: SKILL worktree gate, resume, verdicts/log layering, language injection, finish\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 5: diagnostician-brief.md 改动

**Files:**
- Modify: `<ROOT>/references/diagnostician-brief.md`

- [ ] **Step 1: 改动清单(逐条应用)**

1. 顶部输入块加一行:`- **工作语言**:用主 agent 注入的「工作语言」产出所有内容(verdict / 与主 agent 沟通);引用的英文 skill 内容**理解用英文、产出用工作语言**。`
2. 「若本轮是基线 triage」段补一句:续跑场景下可能被派来**重核代码现实**(代码是否漂移、上轮修复是否还在),回报给主 agent。
3. 节点1 step 3「交「诊断交付单」」→ 改为「**写 diagnostic verdict**(进 `<topic>-verdicts/R<NN>-H<NN>.md`,按 `verdict-template.md`),**只回 1–2 行 digest** 给主 agent;不写 log」。
4. 节点2 step 5「交「修复交付单」」→「**写 repair verdict** + 回 digest」。
5. 「写 log」节改为:仅在两节点都过后,追加 **log 摘要条目**(结论/状态/一句 learning/指针),**完整证据已在 verdict 文件**。
6. 全文 `交付单` → `verdict 节`;确保无 `handoff-template` 残留。

- [ ] **Step 2: 校验**

Run:
```bash
R="$HOME/.agents/skills/hypo-driven-ps/references"
grep -q "工作语言" "$R/diagnostician-brief.md" && grep -q "diagnostic verdict" "$R/diagnostician-brief.md" && grep -q "只回 1–2 行 digest" "$R/diagnostician-brief.md" && echo OK
grep -c -E "交付单|handoff-template" "$R/diagnostician-brief.md"   # 期望 0
```
Expected: OK;计数 0。

- [ ] **Step 3: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/diagnostician-brief.md && git commit -m "$(printf 'v2: diagnostician writes verdict + digest, working language, resume re-grounding\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 6: reviewer-brief.md 改动

**Files:**
- Modify: `<ROOT>/references/reviewer-brief.md`

- [ ] **Step 1: 改动清单**

1. 输入块加一行工作语言(同 Task 5.1)。
2. 输入由「Diagnostician 提交的交付单」→「Diagnostician 写好的 verdict 节(在 `R<NN>-H<NN>.md`)」。
3. 节点1/节点2 产出:**把 diagnostic-review / repair-review verdict 写进同一文件 + 回 digest**(批准/整改/判未解决)。
4. 全文 `交付单` → `verdict 节`;确保无 `handoff-template` 残留。

- [ ] **Step 2: 校验**

Run:
```bash
R="$HOME/.agents/skills/hypo-driven-ps/references"
grep -q "工作语言" "$R/reviewer-brief.md" && grep -q -E "diagnostic-review|repair-review" "$R/reviewer-brief.md" && echo OK
grep -c -E "交付单|handoff-template" "$R/reviewer-brief.md"   # 期望 0
```
Expected: OK;计数 0。

- [ ] **Step 3: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add references/reviewer-brief.md && git commit -m "$(printf 'v2: reviewer writes -review verdict + digest, working language\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 7: 删除 handoff-template.md + 全局无残留

**Files:**
- Delete: `<ROOT>/references/handoff-template.md`
- Modify: `<ROOT>/PLAN.md`(file structure / Task 7 校验里对 handoff-template 的引用 → 改为 verdict-template,或标注 v2 已并入)

- [ ] **Step 1: 删除文件**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git rm references/handoff-template.md
```

- [ ] **Step 2: 全仓库扫残留引用(SKILL/briefs 已在前序 Task 清掉,这里兜底)**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
grep -rn "handoff-template" "$ROOT"/SKILL.md "$ROOT"/references/*.md 2>/dev/null || echo "[clean] 无残留引用"
```
Expected: `[clean] 无残留引用`(若 PLAN.md 里有历史引用,属旧 v1 计划文档,标注即可,不影响 skill 运行)。

- [ ] **Step 3: 更新 PLAN.md 里对 handoff-template 的引用**

把 `PLAN.md` File Structure 与 Task 7 校验文件清单里的 `handoff-template.md` 改为 `verdict-template.md`,并在 PLAN.md 顶部「执行期迭代」说明追加一句:`v2 起 handoff-template 已并入 verdict-template 并删除`。

- [ ] **Step 4: 提交(main,不推)**

```bash
cd "$HOME/.agents/skills/hypo-driven-ps" && git add -A && git commit -m "$(printf 'v2: retire handoff-template (merged into verdict-template)\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

## Task 8: 全量结构校验(收尾)

**Files:** 只读校验。

- [ ] **Step 1: 文件清单(handoff 应消失,verdict 应在)**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
ls "$ROOT/references" | sort
[ ! -f "$ROOT/references/handoff-template.md" ] && echo "handoff 已删 OK"
[ -f "$ROOT/references/verdict-template.md" ] && echo "verdict-template 在 OK"
```
Expected: handoff 已删 OK;verdict-template 在 OK。

- [ ] **Step 2: SKILL 指向的 references 全部存在(引用不悬空)**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
for f in board-template.md log-template.md verdict-template.md diagnostician-brief.md reviewer-brief.md root-cause-tracing.md condition-based-waiting.md defense-in-depth.md; do
  grep -q "$f" "$ROOT/SKILL.md" && ([ -f "$ROOT/references/$f" ] && echo "OK ref: $f" || echo "DANGLING: $f")
done
```
Expected: 8 行 OK,无 DANGLING。

- [ ] **Step 3: v2 关键点全在**

Run:
```bash
ROOT="$HOME/.agents/skills/hypo-driven-ps"
for p in "worktree 门禁" "using-git-worktrees" "最新轮号" "工作语言" "verdict" "finishing-a-development-branch" "必须在隔离 worktree 中运行"; do
  grep -q "$p" "$ROOT/SKILL.md" && echo "OK: $p" || echo "MISS: $p"
done
```
Expected: 7 个 OK,无 MISS。

- [ ] **Step 4: 人工**:新会话触发 + 真实跑一个最小 case 验证 v2(worktree 建/续跑/verdict 落盘)。非脚本。

---

## Self-Review(规划者自检)

- **Spec 覆盖**:DESIGN v2.1 worktree→Task4 E1/E5;v2.2 verdicts→Task1+Task4 E2/E3+Task3;v2.3 续跑→Task4 E1/E4+Task2;v2.4 语言→Task2+Task4 E2+Task5/6;v2.5 收尾→Task4 E6;v2.6 board 头→Task2;handoff 合并→Task1+Task5/6/7。全覆盖。
- **占位符**:模板内 `<...>` 为使用时填的占位,非计划占位;无 TBD/TODO。
- **一致性**:文件名(`R<NN>-H<NN>.md`)、四类 verdict 节名、状态七态、`工作语言`/`最新轮号` 字段、子轮 y 规则,在 verdict-template / SKILL / briefs / log / board 间一致。
