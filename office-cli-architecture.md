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
L1 层的能力边界：
* 读取文本
* 提取结构
* 生成摘要
* 识别标题层级
* 输出统计信息
* 转 JSON / view model
L1作用是：复杂office文件 -> AI可理解结构
### CLI在这一层做了什么
当 AI 或者是用户输入了一条 L1 的命令（如 officecli view outline），OfficeCLI 的 .NET 运行时在后台其实经历了一场流式解析的暴风雨。它主要做了三件事：**解包过滤、语义提取、结构化输出（把文档翻译成“AI 友好的 JSON”）**。
### AI在这一层做了什么
AI 不需要碰具体的XML、DOM， 在拿到 CLI 吐出的 JSON 报告后，主要做三件事：**建立空间感、诊断问题、下达下一阶段指令**。
```
{
  "path": "/body/paragraph[42]",
  "text": "北京飞书科技有限公司",
  "styles": {
    "font": "宋体",
    "size": 12,
    "bold": true
  }
}
```
```
# L1 — 高级视图
officecli view report.docx annotated
officecli view budget.xlsx text --cols A,B,C --max-lines 50
```

---

## 2.第二层：DOM
L2 层是日常读写、修改、增删的核心。它借鉴了网页开发的 DOM（Document Object Model） 思想，把 Office 文档里的段落、表格、单元格、形状，全部映射成了一条条清晰的路径（URI Selector）。

### AI在这一层做了什么
在 L2 层，AI 的大脑已经脱离了宏观的大纲，它的注意力完全聚焦在 L1 规划出来的特定局部节点上。AI 在这一层主要执行符号参数化（Parameterization）和内容生成（Generation）。

**触发条件**：基于第一层（L1）拿到的文档快照情报，以及用户输入的最终业务目标。

**AI 的工作**：LLM 在内部进行参数化映射（Parameterization）。它将“把公司名改成飞书并加粗”的自然语言，编码成符合 OfficeCLI L2 规范的控制参数。

**输出物**：AI 亲手生成了那段包含 path、text、styles 的控制 JSON。

### CLI在这一层做了什么
在 L2 层，CLI 的核心任务是：把 AI 发出的高级、抽象的 DOM 路径命令，翻译成底层极其严苛的 OpenXML 规范修改，并保证文件不损坏。

它对 AI 传过来的 JSON 进行语法和类型强校验（如检查 size 是不是数字，bold 是不是布尔值）。校验通过后，CLI 的抽象层开始将 JSON 对象序列化（Serialization），翻译成微软的 <w:p>、<w:rPr>、<w:b/> 等底层 OpenXML 节点。最终，本地磁盘/内存的文件系统执行覆写操作，重组 ZIP 压缩包


```
# L2 — 元素级操作
officecli query report.docx "run:contains(TODO)"
officecli add budget.xlsx / --type sheet --prop name="Q2 Report"
officecli move report.docx /body/p[5] --to /body --index 1
```

---
## 3.第三层： Raw XML
第三层属于极限兜底机制，专门用来处理 L2 层（标准 DOM 接口）无法解决的、极度复杂的非标准排版或高级样式。在第三层中，控制流的顺序与第二层相同：也是 AI 先工作，然后 CLI 接着工作。

### AI在这一层做什么
AI 在第二层（L2）尝试用标准的 DOM 接口去修改某个元素的复杂格式，但发现 L2 的 API 不支持（或者报错提示该属性无法映射），于是被迫降级启用 L3 。AI直接扮演程序员，手工编写严苛的 OpenXML 标签源码和物理 XPath 定位表达式。

### CLI在这一层做什么
Office-cli不做任何业务校验，直接按 XPath 暴力将 XML 源码强行注入文件流，重组压缩包。
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