# 迭代日志:<topic>  (append-only,只追加不改写)

<!--
  规则:本文件只能在末尾追加新条目,禁止修改/删除既有条目。
  每轮产生两类条目:
    A. 轮次记录(由 **Reviewer** 确认每个节点后追加:节点1 一片段、节点2 一片段;只写 结论/learning/提交/verdict 指针)
    B. 主 agent 板更新记录(由主 agent 在回灌重排后写:跨轮 insight / 状态变化 / 重排 / 处置)
-->

---
## 第 N 轮 — A. 轮次记录(Reviewer 按节点追加;细节在 verdicts）

- **轮次/假设**:R<NN> · H<NN> — <描述>
- **节点1 结论**:判据成立(真)/ 证伪(假)/ 无法判断 — <一句 learning>
- **节点2 结论(若有修复)**:修复解决 / 失败(原因) — <一句 learning>
- **提交(有 git)**:diag <hash/N/A> · fix <hash/N/A> · 若回退 fix-failed <hash> + patch 路径
- **→ verdict 文件**:`docs/hypo-driven-ps/<yyyy-mm-dd>-<topic>/<topic>-verdicts/R<NN>-H<NN>.md`(完整证据/测试清单/各子轮 D/R verdict 在此)

## 第 N 轮 — B. 主 agent 板更新记录

- **通读全量日志得到的跨轮 insight**:<把本轮与历史轮连起来看到的更深结论>
- **对整盘假设的改动**:新增 H? / 排除 H? / 改判据 / 改可能性·验证难度 / 改状态 / 重排
- **下一轮计划**:验 H?,理由 <…>
---
