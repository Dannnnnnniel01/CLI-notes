#Office CLI 三层结构
|层级|定位|AI Agent怎么用|
|L1:View|语义化视图|view text 提取纯文本,view outline 获取文档大纲,view stats 返回页数/字数/表格数|
|L2:DOM|语义化操作|get/set/add/remove像操作网也DOM一样操作文档元素|
|L3: Raw XML|原始XML|raw/raw-set直接XPath访问OOXML底层,兜底一切极端场景|
--
AI负责“理解+决策”，其余由CLI执行
--