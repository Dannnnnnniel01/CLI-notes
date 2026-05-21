# Office CLI 三层结构

| 层级 | 定位 | AI Agent 怎么用 |
| --- | --- | --- |
| L1: View | 语义化视图 | view text 提取纯文本；view outline 获取文档大纲；view stats 返回页数/字数 |
| L2: DOM | 语义化操作 | get / set / add / remove 像操作 DOM 一样操作文档元素 |
| L3: Raw XML | 原始 XML | raw / raw-set 直接 XPath 访问 OOXML 底层，兜底极端场景 |
---

AI 负责“理解 + 决策”，其余由 CLI 执行。