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