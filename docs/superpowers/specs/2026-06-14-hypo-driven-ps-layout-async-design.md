# hypo-driven-ps：课题文件夹收拢 + proofs 证据夹 + 子 agent 异步交互

> 设计日期：2026-06-14
> 目标 skill：`~/.claude/skills/hypo-driven-ps`（独立 git repo）

## 背景与问题

当前 hypo-driven-ps 的所有产物以 `<topic>-` 前缀**直接散落**在 `docs/hypo-driven-ps/` 下（`<topic>-board.md`、`<topic>-log.md`、`<topic>-verdicts/`、`<topic>-probes/`）。该目录会同时承载多个课题，文件互相混杂。本设计解决三个问题：

1. **课题收拢**：每个课题的全部产物收进单一文件夹 `docs/hypo-driven-ps/yyyy-mm-dd-课题名/`。
2. **新增证据夹**：截图/图片等非测试的视觉证据需要一个落点，且必须按 `R<NN>-H<NN>` 归档。
3. **子 agent 异步交互**：当前 Diagnostician/Reviewer 串行阻塞 spawn，主 agent 卡在工具调用里，子 agent 工作时用户无法与主 agent 交互。

---

## 变更 1：课题文件夹收拢

每个课题的所有产物收进单一文件夹，**日期为课题创建日**：

```
docs/hypo-driven-ps/
└── 2026-06-14-课题名/
    ├── 课题名-board.md
    ├── 课题名-log.md
    ├── 课题名-verdicts/          R<NN>-H<NN>.md（文字结论，markdown）
    ├── 课题名-probes/            诊断探针测试代码 + failed-fixes/*.patch
    └── 课题名-proofs/            新增（见变更 2）
```

**统一路径前缀**：凡 skill 内出现 `docs/hypo-driven-ps/<topic>-xxx` 之处，一律改为 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-xxx`。

### 续跑与 worktree 探测

- **续跑匹配键 = 课题名，与日期无关**。续跑探测从「找 `docs/hypo-driven-ps/<topic>-board.md`」改为 glob `docs/hypo-driven-ps/*-<topic>/<topic>-board.md`：
  - 唯一命中 → 用该文件夹续跑（日期前缀沿用，**不变**）。
  - 多命中（同名课题多次创建）→ 报警，列出候选让用户选，不擅自挑。
  - 无命中 → 全新课题，用**当天日期**建 `yyyy-mm-dd-课题名/`。
- **日期前缀创建时定死，续跑永不改写。**
- **worktree 名保持 `hypo-driven-ps-<topic>`（不含日期）**，续跑靠它定位（原生 `EnterWorktree`，落项目内 `.claude/worktrees/`）。
  - 已知边角：同名不同日期的两个课题会争用同一 worktree 名。发生时报警提示用户，**不**把日期塞进 worktree 名（那会破坏「按课题名续跑」的定位）。

---

## 变更 2：proofs 证据夹

新增 `课题名-proofs/`，存放截图、图片、抓取产物等**非测试的视觉/二进制证据**。

### 三类产物分工（消歧）

| 目录 | 内容 | 写入者 |
|---|---|---|
| `<topic>-verdicts/` | 文字结论（诊断/复核 verdict，markdown） | Diagnostician 写主体节、Reviewer 写 review 节 |
| `<topic>-probes/` | 可复用的诊断探针**测试代码** + `failed-fixes/*.patch` | Diagnostician |
| `<topic>-proofs/` | **截图/图片/抓取等视觉/二进制证据** | Diagnostician / Reviewer 取证时 |

### proofs 归档铁律

- `<topic>-proofs/` 下**不得直接放文件**，所有证据必须落 `R<NN>-H<NN>/` 子文件夹（**连字符**，与 verdicts 命名一致）。
- 该 `R<NN>-H<NN>/` 下**建议**再按用途建子文件夹区分（如 `登录失败截图/`、`接口响应/`）。
- 示例：`docs/hypo-driven-ps/2026-06-14-课题名/课题名-proofs/R02-H03/登录失败截图/step1.png`
- verdict 引用证据时写其 proofs 相对路径（与现有「probes 路径写进 verdict」同理）。verdict-template 的「证据」字段补充可引用 proofs 路径。
- proofs 跟随 board/log/verdicts/probes **一起提交进 git**（同属可追溯诊断档案；覆盖全局「`_tmp_` 不入库」规范的同款豁免）。

---

## 变更 3：子 agent 异步交互（background + buffer-and-relay）

将 Diagnostician/Reviewer 的 spawn 从「串行阻塞」改为 **`run_in_background: true`**。这只**解阻塞主 agent 的回合**，**不并发多假设**——任一时刻仍恒有一个 subagent。

### 机制

1. **pre-spawn 用户检查点不变**：spawn 前简报 + 等 go-ahead（自主模式只汇报不等）这条门禁**照旧**——它发生在 spawn *之前*，不受 background 影响。
2. **运行期主 agent 保持可对话**：subagent 在跑时用户随时可找主 agent。主 agent **仍恪守不下场**（不读源码 / 不跑测试 / 不亲自取证）——它只回答用户关于假设板的问题、接收用户指令。
3. **buffer-and-relay 送达**（用户已选定，排除中途 `TaskStop`/实时注入）：
   - 纯状态提问（"现在第几轮""H3 验得怎样了"）→ 主 agent **直接回答**，不入队。
   - 与当前 subagent 任务相关的可执行指令 → 主 agent **即时记下（buffer）**，待当前 subagent **返回 digest 时，先用 `SendMessage` 续同一实例转达，再走下一协议步**。
   - 改排序 / 增删假设 / 改判据类指令 → 自然并入下一轮「回灌重排（第 6 步）」。
   - **无中途打断**：本轮原子完成。已知代价：若用户给的是「方向错了/该换题」类指令，当前轮仍会跑到完成，redirect 在其返回后才生效。
4. **完成通知**：background subagent 完成即重新激活主 agent，主 agent 据 digest 推进（spawn Reviewer / 转修复 / 写板）。主 agent 需能区分本次激活是「subagent 完成」还是「用户插话」，分别走「推进协议」或「答复/buffer」。

### 对现有约定的措辞调整

- 现 SKILL「**串行（不 run_in_background）**」的表述，改为「**单 subagent 串行推进，但用 `run_in_background: true` 非阻塞 spawn**」——强调「串行」指的是「不并发多假设」，而非「阻塞主 agent」。
- 轮内整改/修复重试仍用 `SendMessage` 续同一实例（现有机制不变，恰是 buffer-and-relay 转达的同一通道）。

---

## 落地改动清单（文件级）

1. **`SKILL.md`**
   - 全部产物路径：第 41–46、59、67、72、79、142–144、152–161 行等，统一改为 `docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-xxx`。
   - 产物清单（第 41–46 行）新增 `<topic>-proofs/` 及其归档铁律。
   - 第 0 步续跑探测逻辑：改为 glob `*-<topic>/` 匹配 + 多命中报警（第 62、65 行附近）。
   - 第 2 步 spawn（第 72 行等）：串行阻塞 → `run_in_background: true`；新增「异步交互（运行期可对话 + buffer-and-relay）」小节。
   - Iron Laws / Red Flags：同步「串行不 background」措辞为「单 subagent + 非阻塞」。
   - 用户检查点章节：补一句「background 后运行期主 agent 可对话，但 pre-spawn 门禁不变」。
2. **`references/diagnostician-brief.md`**
   - 路径（第 8、16、21、37 行等）统一为新布局。
   - 新增 proofs 写入规则（取证时落 `<topic>-proofs/R<NN>-H<NN>/<用途>/`）。
   - 补「会被主 agent 续聊转达用户运行期指令」的措辞。
3. **`references/reviewer-brief.md`**
   - 路径（第 8 行等）统一；proofs 证据引用说明。
4. **`references/verdict-template.md`**
   - 路径示例（第 3、22、45 行等）统一为新布局。
   - 「证据」字段补充：可引用 `<topic>-proofs/R<NN>-H<NN>/…` 路径。
5. **`references/board-template.md`**
   - 配套文件路径示例（第 4 行）统一为新布局。
6. **`references/log-template.md`**
   - verdicts 指针路径（第 18 行）统一为新布局。

---

## 不在范围内（YAGNI）

- **并发多假设**：仍恒一个 subagent，本设计不引入并行诊断。
- **中途 `TaskStop` / 实时注入**：用户已明确选 buffer-and-relay，不做中途打断。
- **worktree 名含日期**：会破坏按课题名续跑，不做。
- 其它未触及 skill 行为（优先级规则、修复成本闸门、升级协议等）保持原样。

---

## 验收

- 新建课题：产物全部落在单一 `yyyy-mm-dd-课题名/` 文件夹下，`hypo-driven-ps/` 根目录不再有散落的 `<topic>-` 文件。
- proofs 证据只出现在 `R<NN>-H<NN>/` 子层，根 proofs 目录下无裸文件。
- 续跑：按课题名找回唯一文件夹，轮号续号、日期不变。
- 子 agent 运行期：用户可与主 agent 对话并得到回应；可执行指令在 subagent 返回后经 SendMessage 转达生效；主 agent 全程不下场读码/跑测试。
