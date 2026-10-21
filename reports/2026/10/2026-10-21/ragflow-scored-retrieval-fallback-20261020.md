---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-21
---

# RAGFlow 删除无相关性分数的检索回退：宁可零召回，也不把文档扫描伪装成证据

核查时间：2026-10-21T00:01:59Z。状态：草稿；仅公开来源研究，未经本地复现实验。

## 核心问题与本次增量

文档限域检索先执行 hybrid 查询；旧逻辑一旦零命中且存在 doc_id 过滤，就移除文本/向量 MATCH，改为仅按范围扫描文档。Infinity 对这种没有 MATCH/FUSION 的查询不能投影 score，服务层再把缺失分数填为 0，并复制到 similarity、term_similarity、vector_similarity 三个字段。由此会出现“分块存在、total 正确、三项分数全部 0”的假象，还可能把未经相关性匹配的整页文档片段交给 LLM。问题不等于 Infinity 的融合计算归零，也不证明底层索引健康。

## 输入与前提

用户问题 Question；TenantIDs/KbIDs/DocIDs 与其他范围过滤；可选 EmbeddingModel；VectorOnly、AllowDenseFallback；向量权重、相似度门槛、候选数量与分页。此次变更针对有问题文本的相关性检索失败回退，不移除显式的空问题列举路径。

## 输出

带匹配表达式产生的候选分块、相关性分数及原文档范围；如果允许的带分数检索分支都没有结果，返回空结果。取整篇文档/全部分块属于 chunk-list API，而不是用 retrieval 假装完成相关性检索。

## 关键机制

本次不是替换 embedding、reranker 或 fusion 算法，而是修正检索失败时的语义契约：范围匹配只回答“属于哪篇文档”，相关性匹配才回答“是否支持当前问题”。旧状态机把前者冒充后者；新状态机把“无相关结果”保留为空，并将普通检索的容错召回与 Agentic 证据检索的保守策略分开。维护者通过真实 Infinity 复现实验指出，缺失 score 被填 0 才是三项相似度完全一致为零的原因；报告者随后撤回 filter_fulltext 导致 fusion 归零的原诊断。

## 逐步流程

### 1. 1. 建立范围并执行原检索

保留租户、知识库、doc_id 与其他 filters。常规查询构造文本 MATCH、向量 MATCH 和 fusion；显式 VectorOnly 请求仍直接走向量检索，未被本修复禁用。

### 2. 2. 文档限域零命中也走带分数重试

删除 hasDocIDFilter → MatchExprs=[] 的分支。常规 hybrid 首次为零时，重建较宽的文本匹配条件，通常 min_match 从 0.3 改为 0.1，并将克隆后的向量表达式 similarity 设置为 0.17；范围过滤不改变。若向量权重至少 0.8，minMatch 辅助函数仍返回 0。

### 3. 3. 按使用场景决定是否再做向量救援

普通人机检索若未显式传 AllowDenseFallback，默认 true；宽松 hybrid 仍为零时，可再执行只包含一个向量 MATCH 的救援。Agentic retrieval tool、dataset-navigation 的内容召回层，以及 search_chunks/search_semantic_chunks 共用的 locate 路径显式传 false，不再自动增加这条救援。这里限制的是“回退”，不是禁止模型主动调用语义搜索。

### 4. 4. 不再把无分数扫描结果当成证据

有问题文本的这些重试分支至少保留一个 MATCH；全部失败就返回零结果。需要原文证据的文档读取使用 list_chunks/ChunkService.List。另修补删除失效分块后重建结果时漏传 IndexNames 的问题，保留后续按索引计算所需信息。

### 5. 5. 用代码断言和集成 CI 验证行为

Go 回归测试固定文档范围、检索表达式数量、默认 dense fallback、Agentic opt-out、无相关内容必须返回空，以及 IndexNames 保留；Python 0.27.x 同步删除 filter-only 分支并添加/更新对应测试。主提交的 Elasticsearch、Infinity 与 Go integration 工作流状态为 success。本轮未在本地运行仓库。

## 来源事实与报告效果

- 真实增量已合并：Go main #21240，Python 0.27.x #21242；不是把旧 rc2 重新写成新版本。
  - 证据：GitHub 元数据显示 #21240 于 2026-10-20 11:31:30 UTC 合并，8 文件 +343/-40，merge SHA b99300a；#21242 于 11:31:20 UTC 合并，3 文件 +47/-17，merge SHA cbb2011。nightly 正文显示 2026-10-20 13:18:51 UTC 从 30a6808 构建，比较结果证明其包含 b99300a。
  - 原始来源：https://github.com/infiniflow/ragflow/pull/21240
- 文档范围不再触发无 MATCH 扫描，测试明确覆盖“数据存在但不相关”的零结果。
  - 证据：TestSearchDocumentScopeNeverResortsToFilterOnly 使用模拟返回总数 [0,0,0,3]，要求仅消费前三次请求、MATCH 数为 [3,3,1]、最终 total=0；第 4 个可返回三行的 filter-only 分支不得执行。另一个端到端服务单测要求 reachedFilterOnly=0 且 Chunks/Total 均为 0。这是回归断言，不是数据集上的召回率实测。
  - 原始来源：https://github.com/infiniflow/ragflow/blob/b99300a41d5e0b4302b960521d7e774f28ab5f83/internal/service/nlp/retrieval_test.go
- 不能继续把全零分数解释为 Infinity fusion 坏了。
  - 证据：维护者在 #3523 的复现记录中，无过滤 hybrid 与加 filter_fulltext 范围的 hybrid 均得到 0.705597；filter-only 扫描则返回 3 行且 _score/SCORE/SIMILARITY 缺失，服务层最终填为 0。报告者确认旧问题描述的 filter_fulltext 路径归因错误，并说明其探针用了 similarity_threshold=0.0 才让这些行进入结果。
  - 原始来源：https://github.com/infiniflow/infinity/issues/3523#issuecomment-6219266946
- 普通检索保持额外向量救援默认开启，Agentic 入口显式关闭它。
  - 证据：TestSearchDefaultsDenseFallbackOn 固定 nil→true，模拟 [0,0,3] 后返回三行；retrieval、dataset navigation、locate 三组新测试捕获到 AllowDenseFallback=false。显式 VectorOnly 仍在首轮构建单向量 MATCH。
  - 原始来源：https://github.com/infiniflow/ragflow/blob/b99300a41d5e0b4302b960521d7e774f28ab5f83/internal/service/nlp/retrieval.go
- 提交级集成检查成功，但没有公开的准确率/幻觉率/延迟提升结果。
  - 证据：b99300a 的 check-runs 显示 ragflow_tests_elasticsearch、ragflow_tests_infinity、ragflow_go_integration 均 completed/success；ragflow_native_backend 为 skipped。检查状态只证明对应工作流完成，不能推导新增测试逐项运行或业务 benchmark 提升。
  - 原始来源：https://github.com/infiniflow/ragflow/actions/runs/39415569105

## 局限与未知

- 没有提供标准 QA 数据集上的正确率、召回率、幻觉率或延迟 benchmark，不能宣称这些指标提升。
- 取消无相关性回退会让一些以前“总有内容”的查询变成空结果，这是预期行为变更；上层 UI 与 Agent 应能区分没有证据、服务错误、显式文档读取。
- 报告者怀疑其文档限域首轮零命中还涉及自身索引不一致，但未在该线程给出闭环证明。此修复消除伪分数与错误回退，不保证修好底层索引或所有零召回原因。
- Agentic 的关闭范围是额外 dense-only fallback，不等于关闭所有向量检索或要求每条证据都命中词面；显式 semantic/VectorOnly 查询仍保留。
- 0.17 是重试使用的固定相似度参数，不应笼统称为比初始阈值更低：首轮默认代码值可为 0.1，实际约束还取决于后端及请求参数。
- 该新修复已进入 main/nightly 与 0.27.x 分支；未见带有本修复的新稳定 release。latest RC 仍为 1.0.0-rc2，不能等同于已经自动覆盖所有安装。
- 本轮读完相关 diff 和核心调用上下文，并核对 CI 状态，未在本地部署或复现查询。

## 工程建议（非实验结论）

- 把范围过滤、相关性检索与显式文档读取分成不同 API/评估契约；不要用未经打分的范围扫描填补 RAG 检索失败。
- 迁移测试应加入：指定文档内存在块但无关、首轮零命中而宽松 hybrid 有结果、普通检索 dense rescue、Agentic 禁止额外 rescue、显式 semantic 正常工作、空问题列举不回归。
- 排查全零相似度时先查 Search window 的 matchExprs 与原始命中是否存在 _score/SCORE，而不是立即换 embedding 模型或重建整个向量库。
- 保留文档/租户范围至所有重试，不能为救召回删除权限或数据边界。
- 将 PR #21240 / #21242 的具体 SHA 固定到预发布验证环境；不要仅跟随可变 nightly 标签。上线前以真实中文文档和既有 metadata filters 回放失败样例。
- 若需要全部文档块，调用 chunk-list API；若需要支持回答的证据，接受零召回并让上层选择改写查询、补充读取或明确不足。

## 阅读范围

2026-10-21 Asia/Shanghai 日报核验。完整阅读 #21240 的 8 个变更文件 diff、Go retrieval.go 第 500–970 行核心上下文、全部新增回归测试；完整阅读 #21242 的 3 文件 diff；完整阅读 Infinity #3523 正文与全部 3 条评论；核对两个 PR 合并元数据、nightly 31-commit 比较及 b99300a 的全部 10 个 check-run 状态。没有本地运行/部署，没有把测试源码断言当成实测业务指标。

## 原始来源

- [Go 修复 PR #21240](https://github.com/infiniflow/ragflow/pull/21240)
- [Go 完整差异](https://github.com/infiniflow/ragflow/pull/21240/files)
- [固定 SHA 的检索实现](https://github.com/infiniflow/ragflow/blob/b99300a41d5e0b4302b960521d7e774f28ab5f83/internal/service/nlp/retrieval.go)
- [固定 SHA 的检索回归测试](https://github.com/infiniflow/ragflow/blob/b99300a41d5e0b4302b960521d7e774f28ab5f83/internal/service/nlp/retrieval_test.go)
- [Python 0.27.x 回移 PR #21242](https://github.com/infiniflow/ragflow/pull/21242)
- [Infinity #3523 根因分析与复现](https://github.com/infiniflow/infinity/issues/3523#issuecomment-6219266946)
- [报告者纠正初始归因](https://github.com/infiniflow/infinity/issues/3523#issuecomment-6219932635)
- [维护者确认两条版本线修复](https://github.com/infiniflow/infinity/issues/3523#issuecomment-6288431838)
- [提交集成测试工作流](https://github.com/infiniflow/ragflow/actions/runs/39415569105)
- [nightly 构建来源](https://github.com/infiniflow/ragflow/releases/tag/nightly)
- [前后 nightly 比较](https://github.com/infiniflow/ragflow/compare/7eb673b...30a6808)
