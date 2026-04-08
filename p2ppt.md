---
name: p2ppt
description: Converts academic papers into evidence-grounded, compact PPT plans with manual figure/table placeholders. Use when the user wants to read a paper with LLM and generate slide-ready content while preventing hallucinated claims.
---

# p2ppt

将“读论文”和“做PPT”拆成可控两阶段：
- 上游：事实提取 + 证据绑定
- 下游：精简叙事 + 页面生成

目标：**低幻觉、强可追溯、图表可人工回填、默认精简页数**。

## 1) 何时使用

当用户要：
- 让 LLM 帮忙读论文并输出汇报材料
- 把论文解读结果交给另一个 LLM 生成 PPT
- 要求严格限制脑补、每条结论可追溯
- 需要给图表留“人工截图插入位”

## 2) 工作流

### Stage A — Paper Reader
输入论文文本，输出固定结构 `PAPER_TO_PPT_PAYLOAD`。

### Stage B — Validator
检查结构和证据有效性，阻断高风险输出。

### Stage C — PPT Builder
输入 payload + 风格包 + 页数策略，输出逐页内容。

### Stage D — Human Patch
按图表占位符人工截图回填，最终审阅。

## 3) 全局硬约束（必须遵守）

1. 无证据不得输出结论。
2. 字段缺失必须写 `N/A`，不得留空。
3. 图表只能用人工占位标签，不得伪造图像细节。
4. 不得输出无依据绝对化措辞（如“全面领先”）。
5. 不确定项统一标记 `待人工核验`。
6. 每个 claim 必须是“单一事实”，不得包含 and/or 或多个结论。
7. claim 长度 ≤ 20 English words 或 ≤ 30 中文字。
8. 每个 claim 必须至少绑定一个 evidence。
9. 如果下游 LLM 支持直接生成文档文件（如 .pptx/.md/.docx），应在完成结构化内容后直接产出文档文件。

## 4) 上游输出协议（固定字段）

上游必须使用以下模板，不可改字段名：

```markdown
# PAPER_TO_PPT_PAYLOAD

## meta
- title:
- authors:
- venue:
- year:
- task_type: (method / survey / benchmark / theory / other)
- domain:

## scope
- input_scope: (full / partial / custom)
- missing_sections:
- parse_confidence: (high / medium / low)

## summary
- one_paragraph:

## claims
1. claim_id: C1
   - statement: (单一事实，≤20 English words 或 ≤30 中文字，不得包含 and/or)
   - evidence_refs: [E1, E3]
   - confidence: (high / medium / low)
2. claim_id: C2
   - statement: (单一事实，≤20 English words 或 ≤30 中文字，不得包含 and/or)
   - evidence_refs: [E2]
   - confidence:

## method
- problem_definition:
- key_idea:
- architecture_or_pipeline:
- training_notes:
- inference_notes:

## experiments
- datasets:
- baselines:
- main_results:
- ablation:
- error_or_failure_cases:

## evidence_table
| evidence_id | statement | source_location | source_type | status | supports_claims |
|---|---|---|---|---|---|
| E1 |  | p.5 Fig.2 | figure | supported | [C1] |
| E2 |  | p.7 Table.3 | table | to_verify | [C2] |

## figure_slots
1. slot_id: F_SLOT_01
   - source: Fig.2 (p.5)
   - scope: (chart-only / with-caption / full-block)
   - supports_claim: C1
   - layout_hint: (full / left / right / inset)
   - manual_action: 截图后插入
2. slot_id: F_SLOT_02
   - source:
   - scope:
   - supports_claim:
   - layout_hint:
   - manual_action: 截图后插入

## slide_plan
- slide_id: S1
  claims: [C1, C2]
- slide_id: S2
  claims: [C2]

## limitations
- author_stated:
- uncertain_points:
- potential_misread_risks:

## glossary
- term:
  - definition:
  - first_appearance:

## build_hints
- audience_level: (beginner / mixed / expert)
- preferred_language: (zh / en / bilingual)
- page_policy: (auto / auto-compact / fixed:6 / fixed:8 / fixed:12 / fixed:14)
- content_density: (light / medium / heavy)
- info_budget:
  - total_claims:
  - claims_per_slide:
  - max_slides:

> content_density 上游判定规则：
> - light：claims≤3 且 evidence_table≤5行
> - medium：其他情况
> - heavy：claims≥6 或 evidence_table≥10行 或 figure_slots≥8

## checks
- [ ] claims 全部绑定 evidence_refs
- [ ] evidence_table 非空
- [ ] figure_slots 含 source 与页码
- [ ] 不确定项已标注“待人工核验”
```

## 5) Validator 规则

### 阻断项（必须退回重做）
- claim 未绑定 evidence_refs
- evidence_table 为空
- figure_slots 缺 source 或页码
- parse_confidence=low 且未写 potential_misread_risks

### 警告项（可继续但要提示）
- `to_verify` 占比 > 30%
- 缺 ablation 或 failure cases
- 术语较多但 glossary 为空
- 若 page_policy = auto-compact 且 content_density = heavy，则警告“论文内容丰富，建议使用 auto 或 fixed:12 以获得更完整呈现”

## 6) 下游 Builder Prompt 模板

```markdown
你是“证据驱动且精简优先”的 PPT 生成助手。

[INPUT]
- payload: PAPER_TO_PPT_PAYLOAD
- style_pack
- page_policy
- content_density
- info_budget
- audience_level

[TASK]
在不引入 payload 外事实的前提下，生成可汇报的逐页内容。

[STRICT RULES]
1) 每页必须有 evidence_footer（claim_id + evidence_id + source）。
2) 图表仅允许输出人工占位，不得伪造视觉信息。
3) 信息不足写“待人工核验”。
4) 优先少页高密度，避免冗余背景。
5) 若 page_policy = auto，则根据 content_density 决定页数（light:6-8, medium:8-10, heavy:11-14），每页承载 2-4 个 bullet，每个 claim 至少占 0.5 页。
6) 禁止改写 claim 语义或合并多个 claim。
7) 每个 bullet 必须对应一个 claim_id。
8) 若超出 info_budget.max_slides 或 claims_per_slide 预算，必须优先合并页面与压缩非核心内容。
9) 只要某页存在 manual_figure_slots，就必须输出对应数量的 `[MANUAL_FIGURE_INSERT]` 占位块，禁止仅写 slot_id 列表。
10) visual_layout 必须明确预留图片区域（位置/宽度），保证人工可直接替换截图。

[OUTPUT FORMAT: PER SLIDE]
- slide_no:
- slide_type: (title/problem/method/experiment/ablation/limitation/conclusion/qa)
- slide_title:
- slide_goal:
- bullets: (2-4 条，短句，每条必须绑定 claim_id)
- visual_layout: (必须说明文本区与图片区的比例/位置)
- manual_figure_slots: [F_SLOT_XX]
- manual_figure_insert_blocks: (逐个输出 `[MANUAL_FIGURE_INSERT]` 详细块)
- evidence_footer: (必须包含 claim_id, evidence_id, source, confidence)
- speaker_notes: (40-90 字)

[SELF-CHECK]
- 是否有无证据结论？有则删除。
- 是否存在冗余页？能合并则合并。
- 是否每页都能一句话讲清主旨？
- 若某页声明了 manual_figure_slots，是否输出了完整 `[MANUAL_FIGURE_INSERT]` 块并在 visual_layout 预留图片区？
```

## 7) 页数策略（新版）

默认策略改为 `auto`，基于内容密度自适应页数。

### 默认：`auto`
- 目标范围：**8-14 页**
- 密度映射：
  - light → **6-8 页**
  - medium → **8-10 页**
  - heavy → **11-14 页**

### 用户覆盖策略（fixed 系列）
- `fixed:6` 速讲
- `fixed:8` 标准精讲
- `fixed:12` 详细版
- `fixed:14` 深入版

### 旧版兼容：`auto-compact`
- 保持原行为不变：**5-8 页**
- 适合时间受限、强调主结论速讲场景

### heavy 密度分配模板示例（11-14 页参考）
1. 标题（1页）
2. 背景 + 问题（1页）
3. 威胁模型（1页）
4. 方法（2页）
5. 实验（2页）
6. 消融 + 效率（1页）
7. 局限 + 结论（1页）
8. Q&A（1页）

## 8) 图表占位 DSL（下游必须使用）

```markdown
[MANUAL_FIGURE_INSERT]
slot_id: F_SLOT_01
source: Fig.2 (p.5)
scope: chart-only
layout:
  type: two_column
  ratio: 6:6
  position: right
supports_claim: C1
caption_hint: Main comparison on CIFAR-10
```

最小必填：`slot_id/source/scope/layout.position/supports_claim`。

## 9) 风格包（参数化）

### Academic Clean
- tone: objective
- bullets_per_slide: 2-4
- density: medium-high
- forbidden: hype words, decorative clutter

### Minimal Lite
- tone: concise
- bullets_per_slide: 2-3
- density: medium
- forbidden: long paragraphs, crowded blocks

### Tech Dark
- tone: technical
- bullets_per_slide: 3-4
- density: medium-high
- forbidden: flashy gradients, unsupported claims

风格注入示例：

```markdown
[STYLE_PACK]
name: Minimal Lite
tone: concise
bullets_per_slide: 2-3
density: medium
forbidden: crowded layout, unsupported claims
```

## 10) 新增能力（执行时可开启）

1. 听众适配：`beginner/mixed/expert`
2. 双语输出：`preferred_language=bilingual`
3. Q&A 备答：输出 3 个高概率问题 + 每题 2 条要点
4. 一键压缩：超预算时自动合并页面
5. 证据地图：输出 `slide -> claim -> evidence` 映射
6. 风险评分：输出 hallucination_risk / missing_data_risk

## 11) 失败降级策略

当输入质量差或证据缺失：
1. 保留结构，降低结论强度。
2. 统一增加 `待人工核验`。
3. 输出“草稿版”而非强判断版。

## 12) 标准执行顺序（给代理）

1. 先产出 `PAPER_TO_PPT_PAYLOAD`
2. 跑 Validator 阻断规则
3. 读取用户风格与页数策略（默认 auto，若用户未指定）
4. 生成逐页内容（每页带 evidence_footer）
5. 输出图表占位符与证据地图
6. 最后做自检并给出可汇报版本
