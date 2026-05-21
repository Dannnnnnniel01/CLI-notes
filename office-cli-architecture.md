# Office CLI 三层结构

| 层级 | 定位 | AI Agent 怎么用 |
| --- | --- | --- |
| L1: View | 语义化视图 | view text 提取纯文本；view outline 获取文档大纲；view stats 返回页数/字数 |
| L2: DOM | 语义化操作 | get / set / add / remove 像操作 DOM 一样操作文档元素 |
| L3: Raw XML | 原始 XML | raw / raw-set 直接 XPath 访问 OOXML 底层，兜底极端场景 |
---

AI 负责“理解 + 决策”，其余由 CLI 执行。

---
## 1.第一层：View
```
# L1 — 高级视图
officecli view report.docx annotated
officecli view budget.xlsx text --cols A,B,C --max-lines 50
```

---

## 2.第二层：DOM
```
# L2 — 元素级操作
officecli query report.docx "run:contains(TODO)"
officecli add budget.xlsx / --type sheet --prop name="Q2 Report"
officecli move report.docx /body/p[5] --to /body --index 1
```

---
## 3.第三层： Raw XML
```
# L3 — L2 不够时用原始 XML
officecli raw deck.pptx '/slide[1]'
officecli raw-set report.docx document \
  --xpath "//w:p[1]" --action append \
  --xml '<w:r><w:t>Injected text</w:t></w:r>'
```

---
## office-cli在智能体上的优势
- 确定性 JSON 输出 —— 每条命令都支持 --json，schema 一致。无需正则解析、无需抓 stdout。
- 基于路径的寻址 —— 每个元素都有稳定路径（/slide[1]/shape[2]）。智能体无需理解 XML 命名空间即可导航文（OfficeCLI 自己的语法：1-based 索引、元素本地名——不是 XPath。）
- 渐进式复杂度（L1 → L2 → L3） —— 智能体从只读视图入手，升级到 DOM 操作，仅在必要时降到 raw XML。最大限度节token。
- 自愈式工作流 —— validate、view issues、以及结构化错误码（not_found、invalid_value、unsupported_property）- 会返回 suggestion 和有效范围。智能体无需人工介入即可自纠错。
- 内置 agent 友好渲染引擎 —— view html / view screenshot / watch 原生输出 HTML 和 PNG。无Office。智能体能"看见"自己的产出，并在 CI / Docker / 无头环境里修复排版问题。
- 内置公式与透视引擎 —— 150+ Excel 函数写入即自动求值；从源数据范围一条命令生成原生 OOXML 数据透视表。智能体立刻读到计算值和聚合结果，不需要回到 Office 重算。
- 模板合并 —— 智能体一次性设计版式，下游代码把 {{key}} 占位符填充 N 次。避免每份报告都烧 token 重生成。
- Dump 往返 —— dump 把任意 .docx 转成可重放的 batch JSON。智能体通过读结构化规格学习人类范本，而不是从原始 OOXML XML 反推。
- 内置帮助 —— 属性名或取值格式不确定时，智能体跑 officecli <format> set <element>，不靠猜。
- 自动安装 —— OfficeCLI 自动识别您的 AI 工具（Claude Code、Cursor、VS Code…）并完成配置。无需手动放 skill 文件。