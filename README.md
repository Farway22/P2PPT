# 📚 p2ppt

> **Evidence-driven Paper → PPT Skill for Cursor**
>
> 把“读论文”和“做 PPT”拆成可控流程：
> 上游负责事实与证据，下游负责精简叙事，人工负责图表截图回填。

---

## ✨ Highlights

- 🧾 **证据优先**：无证据不结论，降低幻觉
- 🧠 **结构化输出**：统一 `PAPER_TO_PPT_PAYLOAD` 协议
- 🖼️ **人工图表占位**：强制 `MANUAL_FIGURE_INSERT`，避免编造图像细节
- 📉 **精简页数策略**：默认 `auto`，按内容密度自适应
- 🎨 **风格可选**：Academic Clean / Minimal Lite / Tech Dark
- 🧮 **预算约束**：支持 `info_budget`，超预算自动压缩

---

## 🤔 为什么需要它

传统论文转 PPT 常见问题：

1. 模型看不懂或拿不到论文图/表关键细节；
2. 模型容易“合理化脑补”，把推测写成事实；
3. 不同模型串联时接口不稳定，输出风格跑偏。

`p2ppt` 通过 **固定协议 + 校验器 + 占位 DSL**，把流程从“能生成”变成“可审阅、可追溯、可接管”。

---

## 🧩 Workflow

```text
Stage A: Paper Reader   -> 生成 PAPER_TO_PPT_PAYLOAD
Stage B: Validator      -> 校验证据/字段/图位
Stage C: PPT Builder    -> 生成逐页内容（受风格与预算约束）
Stage D: Human Patch    -> 人工插入图表截图 + 终审
```

---

## 🛠️ Core Capabilities

| 能力 | 说明 |
|---|---|
| Fixed Payload Contract | 统一字段，缺失必须 `N/A` |
| Anti-Hallucination Rules | claim 必须绑定 evidence |
| Figure Placeholder DSL | 统一图位占位格式，可直接回填 |
| Adaptive Page Policy | `auto` 按密度映射 6-14 页 |
| Compact Compatibility | 保留 `auto-compact`（5-8 页） |
| Budget-aware Compression | 超过 `info_budget` 自动压缩 |

---

## 📂 Project Structure

```text
.
├── .cursor/
│   └── skills/
│       └── p2ppt/
│           └── SKILL.md
└── README.md
```

---

## 🚀 Quick Start

1. 打开 `./.cursor/skills/p2ppt/SKILL.md`
2. 将论文文本输入上游 Reader，生成 `PAPER_TO_PPT_PAYLOAD`
3. 运行 Validator，确保阻断项全部通过
4. 选择参数：
   - `style_pack`
   - `page_policy`（默认 `auto`）
   - `content_density`（light / medium / heavy）
5. 调用 Builder 生成逐页内容
6. 按 `[MANUAL_FIGURE_INSERT]` 占位符回填截图

---

## 📄 Page Policy

- **默认**：`auto`
  - light → 6–8 页
  - medium → 8–10 页
  - heavy → 11–14 页
- **兼容**：`auto-compact`（5–8 页）
- **固定**：`fixed:6` / `fixed:8` / `fixed:12` / `fixed:14`

---

## 🎯 Use Cases

- 组会论文汇报
- 课程论文展示
- 技术调研分享
- 需要“可追溯、低幻觉”输出的学术场景

---

## ⚠️ Notes

- 本 Skill 不做自动截图与自动重绘图表
- 图/表信息以论文原文为准，模型仅输出占位结构
- 输入质量较差时，结果会自动降级为“草稿版”

---

## 📜 License

MIT

