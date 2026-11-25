---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-25
verified_at: 2026-11-25T00:04:55Z
---

# RAGFlow 原生导入 Docling JSON：保留阅读顺序、标题与页坐标

## 问题
此前上传 Docling JSON 时，普通对象解析可能把序列化内容当成一项文本；保留列表、页号和坐标的外部解析价值没进入下游。与上期 Docling 的解析取消不同，此处新增两套工具之间的入库桥接。

## 输入与输出
- 输入：Docling 导出的普通 .json，顶层 schema_name 为 DoclingDocument 且包含 body 对象；面向 General 解析/切块。
- 输出：按文档树顺序输出正文与标题，保留段落换行、标题等级和有效页码/坐标；无有效框时可保留正文与页码。

## 流程
- **判定输入结构**：在现有 JSON parser 中识别特定 Docling 文档；一般 JSON 和逐行 JSON 维持旧逻辑。
- **跟随 body 引用**：以 body.children 为根，按子节点顺序遍历受支持的本地引用。按引用去重，不以文本内容去重，因此不同页的相同段落仍保留；家具节点不在正文树的不会遍历。
- **映射为检索片段**：仅将 texts 节点变为输出项；标题类型映射等级，正文末尾补段落换行，有效位置信息经标准化后交给 General chunker。

## 机制
把文本和位置放在同一项上，而不是随后用字符串重新寻找来源。读取的是导出的树序，不重新推断人类阅读顺序。

## 结果与证据
- **作者用原有导出文件重放验证** 作者报告37页 Docling 1.5 JSON 获得336个带位置文本项（内含12个标题）和9个 chunks。是一个示例的输出计数，不是召回或准确率结论。 [原始证据](https://github.com/infiniflow/ragflow/pull/22728)
- **有实际解析器到 General chunker 的回归** 合成回归断言两页的标题正文合并后换行不丢、等级为2并保留两条坐标；另有阅读顺序、不同页重复文字、坏坐标与普通JSON/JSONL覆盖。作者报告关键测试旧基线失败、补丁后通过，未独立运行。 [原始证据](https://github.com/infiniflow/ragflow/blob/190e62319f238dc09c4df9fe624efd7fa5c373e9/internal/ingestion/component/chunker/general_test.go)

## 局限与未知
- 初版支持正文和标题，不等同结构化表格、图片描述、公式被完整消费，也不承诺原始 PDF 能被界面点击预览。
- 11月24日主分支合并，本轮所核最新正式 v1.0.0 在11月6日，不将主分支写成稳定版已包含。
- 缺失或不合法的坐标不能恢复；未测吞吐、召回质量或不同 Docling schema 版本的普遍兼容性。

## 工程建议
- 以有跨页重复段落的公开 PDF 导出 Docling JSON，再用包含该提交的 RAGFlow 做一次 General 导入；核对正文、页码、框和段落边界。
- 保留源 PDF 与出口版本；对需要表格、图片与方程的文档，先用单独样本验证结构不会在入库丢失。

## 研读范围
已读 PR 原始说明及五文件完整 diff，核到发布记录；未下载示例附件或运行测试。

## 原始来源
- [PR与作者验证](https://github.com/infiniflow/ragflow/pull/22728)
- [固定版本导入实现](https://github.com/infiniflow/ragflow/blob/190e62319f238dc09c4df9fe624efd7fa5c373e9/internal/parser/parser/json_parser_docling.go)
- [固定版本逐项测试](https://github.com/infiniflow/ragflow/blob/190e62319f238dc09c4df9fe624efd7fa5c373e9/internal/parser/parser/json_parser_docling_test.go)
- [General端到端桩测试](https://github.com/infiniflow/ragflow/blob/190e62319f238dc09c4df9fe624efd7fa5c373e9/internal/ingestion/component/chunker/general_test.go)
- [导入配置指南](https://github.com/infiniflow/ragflow/blob/190e62319f238dc09c4df9fe624efd7fa5c373e9/docs/guides/dataset/configuration.md)
- [RAGFlow v1.0.0发行](https://github.com/infiniflow/ragflow/releases/tag/v1.0.0)
