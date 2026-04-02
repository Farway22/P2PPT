# p2ppt

`p2ppt` 是一个将“论文阅读”与“PPT生成”解耦的 Skill 规范：
- 上游模型负责**事实提取与证据绑定**。
- 下游模型负责**受控排版与叙事压缩**。
- 人工负责**图表截图回填与最终审阅**。

设计目标：**规范、可追溯、可复用、少废话、低幻觉**。

---

## 1. 规范定位（Spec）

### 1.1 适用场景
- 组会论文汇报
- 文献精读讲解
- 课程作业演示
- 项目立项前技术扫描

### 1.2 非目标
- 自动截图与自动裁剪
- 自动重绘论文图表
- 多论文自动融合综述

### 1.3 核心原则
1. 证据优先：所有结论必须可回溯。
2. 结构固定：上游输出字段不可随意变化。
3. 严禁脑补：信息不足统一标记。
4. 精简优先：默认做“短讲可用稿”，不是“信息堆砌稿”。

---

## 2. 总体架构

### Stage A: Paper Reader
输入论文文本，输出结构化载荷 `PAPER_TO_PPT_PAYLOAD`。

### Stage B: Validator
检查字段完整性、证据覆盖率、图表占位有效性。

### Stage C: PPT Builder
按风格包 + 页数预算，生成逐页内容。

### Stage D: Human Patch
按图表占位插入截图，做最终审阅。

---

## 3. 全局硬约束（MUST）

1. 无证据不得输出结论。
2. 缺失字段必须写 `N/A`，不得空置。
3. 图表页必须使用人工占位标签，不得伪造图像细节。
4. 不得出现无依据绝对化措辞（如“全面领先”“彻底证明”）。
5. 不确定内容必须显式标记 `待人工核验`。

---

## 4. 上游输出协议（Contract v1.1）

上游必须使用以下模板输出，不可改字段名。

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
   - statement:
   - evidence_refs: [E1, E3]
   - confidence: (high / medium / low)
2. claim_id: C2
   - statement:
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
| evidence_id | statement | source_location | source_type | status |
|---|---|---|---|---|
| E1 |  | p.5 Fig.2 | figure | supported |
| E2 |  | p.7 Table.3 | table | to_verify |

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
- page_budget_hint: (auto-compact / fixed)

## checks
- [ ] claims 全部绑定 evidence_refs
- [ ] evidence_table 非空
- [ ] figure_slots 含 source 与页码
- [ ] 不确定项已标记“待人工核验”
```

---

## 5. Validator 规范

### 5.1 阻断项（出现则退回重做）
- 存在 `claim` 未绑定 `evidence_refs`
- `evidence_table` 为空
- `figure_slots` 缺 source 或页码
- `parse_confidence=low` 且未提供 `potential_misread_risks`

### 5.2 警告项（可继续，但需提示）
- `to_verify` 比例 > 30%
- 缺失 ablation 或 failure cases
- 术语表为空且技术术语较多

### 5.3 质量评分（可选）
- evidence_coverage: 0-100
- structure_completeness: 0-100
- hallucination_risk: low/medium/high

---

## 6. 下游 PPT Builder Prompt 模板（标准版）

```markdown
你是“证据驱动且偏精简”的 PPT 生成助手。

[INPUT]
- payload: PAPER_TO_PPT_PAYLOAD
- style_pack: 用户风格配置
- page_policy: auto-compact
- audience_level

[TASK]
在不引入 payload 外事实的前提下，生成可直接汇报的逐页内容。

[STRICT RULES]
1) 每页必须包含 evidence_footer（claim_id + evidence_id + source）。
2) 图表只能输出人工占位标签，不得虚构图像信息。
3) 信息不足时写“待人工核验”，禁止猜测补全。
4) 优先“少页高密度”，避免冗余背景铺垫。

[OUTPUT FORMAT: PER SLIDE]
- slide_no:
- slide_type: (title/problem/method/experiment/ablation/limitation/conclusion/qa)
- slide_title:
- slide_goal:
- bullets: (2-4条，短句)
- visual_layout:
- manual_figure_slots: [F_SLOT_XX]
- evidence_footer:
- speaker_notes: (40-90字)

[SELF-CHECK]
- 是否有无证据结论？有则删除。
- 是否有冗余页？能合并则合并。
- 是否每页都能一句话讲清主旨？
```

---

## 7. 页数策略（改为精简优先）

> 不再默认 10 页，默认采用 `auto-compact`。

### 7.1 auto-compact 规则
- 目标页数范围：**5-8 页**
- 合并原则：
  - 背景 + 问题合并
  - 方法总览 + 核心模块合并
  - 主结果 + 消融（轻量）可合并
- 删除原则：
  - 无证据支撑的“扩展讨论”删除
  - 与主结论弱相关的次要实验删除

### 7.2 固定页数模式（用户指定时）
- `fixed:6`（速讲）
- `fixed:8`（标准精讲）
- `fixed:12`（详细版）

### 7.3 默认建议
- 首次输出：`auto-compact`
- 若听众为专家：优先 6-8 页，减少背景解释
- 若听众为初学者：6-8 页内适当增加术语解释

---

## 8. 页型规范（精简版）

建议按以下最小骨架生成：
1. Title & Thesis（1页）
2. Problem + Gap（1页）
3. Method Core（1页，含图位可选）
4. Main Evidence（1-2页，图表位优先）
5. Limitation + Risk（1页）
6. Conclusion + Q&A（1页）

---

## 9. 图表占位 DSL（严格格式）

下游必须用以下标签输出图位：

```markdown
[MANUAL_FIGURE_INSERT]
slot_id: F_SLOT_01
source: Fig.2 (p.5)
scope: chart-only
position: right-6/12
supports_claim: C1
caption_hint: Main comparison on CIFAR-10
```

最小必填字段：`slot_id/source/scope/position/supports_claim`。

---

## 10. 风格包（参数化）

### A. Academic Clean
- tone: objective
- density: medium-high
- bullets_per_slide: 2-4
- forbidden: hype words, decorative clutter

### B. Minimal Lite
- tone: concise
- density: medium
- bullets_per_slide: 2-3
- forbidden: long paragraphs, crowded blocks

### C. Tech Dark
- tone: technical
- density: medium-high
- bullets_per_slide: 3-4
- forbidden: flashy gradients, unsupported claims

风格注入样例：

```markdown
[STYLE_PACK]
name: Minimal Lite
tone: concise
density: medium
bullets_per_slide: 2-3
forbidden: crowded layout, unsupported claims
```

---

## 11. 新增功能（比基础版多）

### 11.1 听众适配
- `audience_level=beginner`：术语首次解释
- `audience_level=expert`：减少背景，强化实验证据

### 11.2 双语模式
- `preferred_language=bilingual` 时：标题中文、术语保留英文

### 11.3 Q&A 备答包
- 输出末页附 3 个高概率问题 + 每题 2 条答题要点

### 11.4 一键压缩模式
- 当页数超预算时，自动执行“页合并 + 次要信息剔除”

### 11.5 证据地图
- 额外输出 `slide -> claim -> evidence` 映射，便于复核

---

## 12. 失败降级策略

当输入质量低或证据缺失时：
1. 保留结构，减少结论强度。
2. 增加 `待人工核验` 标签。
3. 将结果降级为“草稿版”（不输出强判断）。

---

## 13. 最小使用流程（5步）

1. 把论文文本喂给 Reader。
2. 获取 `PAPER_TO_PPT_PAYLOAD`。
3. 运行 Validator 过滤阻断项。
4. 选择 `style_pack` + `page_policy=auto-compact`。
5. 交给 Builder 生成内容并按图位回填截图。

---

## 14. 默认参数（更新后）

- page_policy: `auto-compact`
- target_pages: `5-8`（不是默认10页）
- bullets_per_slide: `2-4`
- speaker_notes_length: `40-90字`
- uncertain_policy: `待人工核验`

---

## 15. 一句话定义

`p2ppt` 是一个“证据约束 + 结构化接口 + 精简页数策略”的论文转 PPT Skill：
让 LLM 既能高效产出，也能被人类快速审阅和接管。
