---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-22
verified_at: "2026-11-22T00:07:24.832Z"
---

# LightRAG：先校验 Jina 批次行号，再把向量写给对应文本

## 问题

旧绑定按服务响应顺序返回向量。如果供应端给出的行号正确、排列却不同，行数和维度检查即使通过，按位置配对的向量库仍可能把向量写给错误文本。相较此前的引用定位、重排重试和查询错误传播，本次保护的是入库映射。

## 输入

按输入顺序的一批文本、Jina 兼容响应中的编码向量及可选整数 index；示例后端 NanoVectorDB 按行配对。

## 输出

带索引批次按原输入重排后返回；无索引批次按响应顺序返回；不完整或非法索引批次抛出 ValueError。

## 流程

- **检查批次长度**：先验证返回行数与文本数相同；行数相同只说明数量，不说明对应关系。
- **验证并排序**：只要任意行含 index，就要求全部索引是严格整数且完整覆盖该批次的零起始编号。重复、缺失、越界和布尔值均不被当作合法索引。校验通过才排序。
- **解码与写入**：再做 base64 解码。如使用 NanoVectorDB，其待写记录与向量按顺序匹配；输入顺序得到恢复才避免误绑。全无 index 的兼容响应不重排。

## 关键机制

把行数断言升级为可验证的一一配对；这不是重排结果排序，也不修复已经写错的旧索引。

## 作者结果与可核对事实

- **变更已进入候选版**：PR #4532 于2026-11-21 14:29:21 UTC合并（2112670c）；v1.5.8rc2 于18:43:44 UTC发布，已核其标签源码含新检查而 rc1 不含。 [来源](https://github.com/HKUDS/LightRAG/releases/tag/v1.5.8rc2)
- **最终离线源码覆盖六组**：两组正向测试为倒序与全无 index；四组反向测试为重复、越界、布尔和缺失。桩反馈小型二维向量，未联真实提供方。PR描述的九组与多批通过数是较早作者验证，不作为最终六组的运行记录。 [来源](https://github.com/HKUDS/LightRAG/blob/v1.5.8rc2/tests/llm/jina_impl/test_jina_embedding_order.py)

## 边界与未知

- RC 不是正式稳定版；未独立运行测试或重建索引。
- 无行号的自定义接口仍假设响应顺序等于输入顺序；批次行号正确也不证明服务端未把错误向量放进该行。
- 不证明线上供应端此前发生过乱序，也无检索质量或性能提升数字。

## 工程建议（未实测）

- 可用合成向量模拟两个显著不同的文本，先让响应倒序但保留 index，再删除、重复一条 index，分别验收向量配对及失败路径。
- 已有索引若疑似受影响，先抽样对照原始向量与单条重嵌入结果，再决定是否重建；不要用更新候选版代替存量审计。

## 阅读范围

核对实时 PR、两个变更文件完整 diff、两个发行标签适配器和固定版本 NanoVectorDB 写入路径。未运行代码、重建或联系提供方。

## 原始来源

- [合并PR与作者验证](https://github.com/HKUDS/LightRAG/pull/4532)
- [固定RC绑定](https://github.com/HKUDS/LightRAG/blob/v1.5.8rc2/lightrag/llm/jina.py#L160-L195)
- [固定RC离线回归](https://github.com/HKUDS/LightRAG/blob/v1.5.8rc2/tests/llm/jina_impl/test_jina_embedding_order.py)
- [按位写入示例](https://github.com/HKUDS/LightRAG/blob/v1.5.8rc2/lightrag/kg/nano_vector_db_impl.py#L765-L790)
- [RC发行记录](https://github.com/HKUDS/LightRAG/releases/tag/v1.5.8rc2)
