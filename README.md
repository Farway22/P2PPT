# p2ppt

一个面向 **“读论文 → 做PPT”** 的 Cursor Skill，目标是：

- 用上游 LLM 做**证据驱动**的论文解析
- 用下游 LLM 做**精简可讲**的幻灯片生成
- 对论文图表采用**人工截图占位**，避免模型乱编图像细节
- 用固定协议与校验规则，降低幻觉和风格跑偏

---

## 为什么做这个 Skill

常见问题是：
1. 做 PPT 的模型很强，但拿不到论文图/表的真实语义；
2. LLM 容易“合理补全”，把猜测写成事实；
3. 不同模型之间接口松散，导致输出不稳定。

`p2ppt` 的思路是把流程拆开：
- **Stage A（Reader）**：只做事实提取 + 证据绑定
- **Stage B（Validator）**：做阻断式校验
- **Stage C（Builder）**：按风格和页数策略生成逐页内容
- **Stage D（Human Patch）**：人工插入截图并终审

---

## 核心能力

- **固定 payload 协议**：统一上游输出结构，字段缺失必须填 `N/A`
- **硬约束防幻觉**：无证据不结论，不确定项统一 `待人工核验`
- **图表占位 DSL**：标准化 `MANUAL_FIGURE_INSERT` 标签，便于人工回填
- **默认自适应页数**：`page_policy=auto`，按内容密度自动映射页数
- **兼容精简模式**：保留 `auto-compact`（5-8 页）
- **风格包注入**：Academic Clean / Minimal Lite / Tech Dark
- **预算控制**：支持 `info_budget`，超预算时自动压缩非核心内容

---

## 目录结构

```text
.
├── .cursor/
│   └── skills/
│       └── p2ppt/
│           └── SKILL.md
└── README.md
```

---

## 快速开始

1. 打开 `./.cursor/skills/p2ppt/SKILL.md`
2. 将论文文本输入上游 Reader，生成 `PAPER_TO_PPT_PAYLOAD`
3. 运行 Validator，确保通过阻断规则
4. 选择：
   - `style_pack`（风格）
   - `page_policy`（默认 `auto`）
   - `content_density`（light/medium/heavy）
5. 调用下游 Builder 生成逐页内容
6. 按 `MANUAL_FIGURE_INSERT` 占位符回填截图，完成终稿

---

## 页数策略（当前默认）

- 默认：`auto`（基于内容密度）
  - light → 6-8 页
  - medium → 8-10 页
  - heavy → 11-14 页
- 兼容：`auto-compact`（5-8 页，旧策略）
- 固定：`fixed:6` / `fixed:8` / `fixed:12` / `fixed:14`

---

## 适用场景

- 组会论文汇报
- 课程论文展示
- 技术调研汇报
- 需要“可追溯、低幻觉”材料的学术场景

---

## 注意事项

- 本 Skill 不做自动截图和自动图表重绘
- 图/表信息以论文原文为准，模型只能输出占位结构
- 若输入质量较差（缺页、解析错误），输出会自动降级为“草稿版”

---

## 版本建议

如果你准备上传 Git：
- 建议将 `./.cursor/skills/p2ppt/SKILL.md` 作为核心规范
- `README.md` 作为项目说明入口
- 后续可加 `examples/` 放 1~2 个真实论文样例（可显著提升可复用性）

---

## License

MIT（可按你的项目需要修改）
