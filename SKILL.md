---
name: literature-benchmark-runner
description: Use when a literature-reading evaluation workorder must be scored item by item with deterministic rules and failure analysis.
---

# 文献阅读评测执行与失败分析

## 用途与适用条件

读取评测工作单、参考答案、候选输出与运行条件，按确定性规则逐题打分，汇总总分、分类与智能体维度，输出失败原因和版本快照；每个分数都能顺着 `item_id` 追到候选输出、判定理由、评分规则与案例哈希。

适用：`/input/case.json`（或 `--case` 指定），可带 `/input/task.md` 与同目录工作单（`evaluation-workorder.json`、`workorder.json`、`benchmark-items.json`）。输入含 `items`/`tasks`/`benchmark_items`/`evaluation_items`，每项有题目、参考答案与一个或多个候选输出。

不适用：不调用外部模型重新生成答案；不修改评分标准；没有候选项时不给分。

## 输入

- 主输入 `/input/case.json`：只读取其中真实的题目、参考答案与候选输出，不根据文件名、标题或领域标签补造内容。
- 任务说明 `/input/task.md`：存在时作为任务语义依据；缺失时仅使用 `case.json` 的显式字段。
- 配套材料从 `/input` 与案例文件同目录读取，成功读取的文件名记入 `result.version.workorder_refs`，解析失败记入 `errors`。
- 路径解析顺序：`/input` → 案例目录 → 当前目录 → `/app`；找不到的路径写入 `errors`/`uncertainty`，不静默替换。
- 支持 TXT/MD/CSV/TSV/JSON/BibTeX/RIS/XML/HTML、DOCX/PPTX/XLSX（解包 XML）与 PDF（最小流解析）；无文本层文件只登记状态。
- 仅用 CPython 3.9+ 标准库；不联网、不读取环境变量中的令牌、不修改 `/input`，只写 `--out` 与 `--work`。

## 处理流程

1. **读取工作单**：解析题目、类别、参考答案、必含词、满分与候选输出；记录案例哈希。
2. **输出规范化**：统一大小写、空白与标点；保留原始输出用于复核。
3. **规则评分**：空输出或拒答记 0；规范化完全相等记满分；否则按必含词比例与词元 F1（阈值 0.80 / 0.45）给 0/1/2。
4. **逐项取证**：每个分数附参考摘要、候选摘要与判定理由；参考答案缺失时标 `not_run`，不给分。
5. **汇总统计**：总分、均分、归一化分数，按类别与智能体分组。
6. **失败分析**：未满分项给出原因分类与缺失词。
7. **契约落盘**：保留 `case_id` 与全部契约字段，输出确定性版本快照。

## 输出契约

写入 `/app/output/result.json`（UTF-8 JSON，确定性排序，无随机、无时间戳）。顶层必须保留：

- `case_id`：与输入完全一致（缺失时为 `null`）；
- 输入 `output_contract` 声明的每一个字段名，原样保留；无法确定性映射的写 `null`，并在 `contract_notes` 说明原因；
- `status`：`success` / `partial` / `insufficient_input` / `failed`；
- `result`、`evidence`、`uncertainty`、`errors`、`stats`、`contract_notes`、`skill`、`challenge_id`。

主要交付段与必填子字段：

| 段 | 必填内容 |
|---|---|
| `result.items[]` | `item_id`、`agent`、`score`、`max_score`、`reason`、`output_excerpt`、`reference_excerpt`、`task`；调用 Agent 追加 `locator`（来源文件 + 段落/行/表行列） |
| `result.scores` | `item_id`、`agent`、`score`、`max_score` |
| `result.summary` | `items`、`scored_outputs`、`not_run`、`score_sum`、`score_max`、`mean_score`、`normalised`、`by_agent`、`by_category` |
| `result.failures` | 未满分项的 `item_id`、`agent`、`score`、`max_score`、`reason` |
| `result.version` | `workorder_refs`、`case_sha256`、`scoring_policy`、`deterministic` |
| `evidence[]` | 每条含 `item_id`、`locator`、`quote`、`role`；缺定位依据的标 `anchor_status: unverified`，不得写成已核验 |
| `needs_human_confirmation[]` | `item`、`locator`、`reason`：缺参考答案、候选为空、规则无法判定时逐条登记 |

字段名以实际 `output_contract` 为准；不删除、不改名、不放宽契约字段。

## 交付成品要求

1. **交付成品而不是流程说明**：脚本只产出评分骨架，调用 Agent 必须在同一 `result.json` 中补齐全部条目的 `locator`、失败归因与 `needs_human_confirmation`；禁止 `pending`、`TODO` 或空交付段。
2. **覆盖全部输入条目**：每个题目、候选项与智能体都要有对应行，不得只处理首条；确实无法评分的写明原因，不静默跳过。
3. **自检**：`case_id` 与输入一致；`output_contract` 每字段存在或 `null`+`contract_notes`；`status` 取上述枚举；总分、均分与逐项分数自洽；重复运行结果完全相同。
4. **参数等价**：`--out` 与 `--output` 等价；默认 `/input/case.json` → `/app/output/result.json`。
5. **由输入驱动**：不针对单一示例硬编码；字段命名差异走别名匹配，并在 `uncertainty` 记录假设。

## 分数口径

输出的是本 Skill 的规则自评分，不是平台、DSH 或官方案例得分；技术性合成检查日志也不是独立语义验证。不得写入编造的分数、名次或评审结论。

## 异常与降级

| 情况 | 处理 | 输出表现 |
|---|---|---|
| `case.json` 缺失或非法 JSON | 不改写输入，写出结构化失败结果 | `status: insufficient_input`（解析错误为 `failed`），`errors` 记原因 |
| 工作单/字段表缺失 | 继续处理可用材料，不猜测清单 | `errors` 记录缺失文件，相关段为空并说明 |
| 条目无参考答案 | 该条记 `not_run`，不给分 | `score: null`，`needs_human_confirmation` 登记 |
| 条目无候选输出 | 不判分 | `status: partial`，`uncertainty` 说明 |
| 文件损坏、无文本层或无权限 | 登记状态并跳过，继续其余材料 | `read_status`/`uncertainty` 标注 |
| 运行异常 | 捕获后输出结构化失败 | `status: failed`，`errors` 含异常类型 |

进程正常结束（含失败分支）退出码恒为 `0`，失败类别只看 `status`，不要用退出码判断成败。

## 限制

规则评分偏重词面重合，不能判断深层语义正确性；无参考答案或无候选输出时不产生分数。

## 最小调用示例（示例，非官方案例结果）

```bash
python scripts/run_skill.py --case /input/case.json --out /app/output/result.json --work /app/work
```

示意输入（合成，仅演示接口）：

```json
{"case_id":"synthetic-smoke","output_contract":{"scores":"array"},"items":[{"item_id":"I1","reference":"80 C","candidate":"80 C","max_score":2}]}
```

示意输出（合成，仅演示字段形状，不代表任何真实案例或评分）：

```json
{"case_id":"synthetic-smoke","scores":[{"item_id":"I1","score":2,"max_score":2}],"summary":{"items":1,"scored_outputs":1},"status":"success","result":{},"evidence":[],"uncertainty":[],"errors":[],"stats":{},"contract_notes":{}}
```

## 自查清单

- [ ] `case_id` 与输入一致；`output_contract` 每个字段都存在，或记为 `null` 并写入 `contract_notes`。
- [ ] 每个分数都能由 `item_id` 追到候选输出摘要、`reason`、`case_sha256` 与 `scoring_policy`。
- [ ] 缺参考答案、空输出、损坏文件、冲突数值与未报告项均已登记，未写成“全部完成”。
- [ ] 重复运行结果相同；未联网、未修改 `/input`；未声称平台评分。
