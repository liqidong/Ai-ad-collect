---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-27
---

# RAGFlow Go 画布接通真正的增量流式输出

核查时间：2026-10-27T00:08:00Z

## 核心问题

此前 LLM.Stream 等待完整 Invoke 返回再发单块；直接连接 Message 的 LLM/Generate 也先完整执行，界面无法及时收到模型增量。

## 输入

Go Agent canvas 中的 LLM 或 Generate、模型请求及 Message 模板；底层 invoker 需支持 StreamingInvoker 才有真实增量。

## 输出

content/thinking 增量事件、最终响应及 node_finished 输出或错误。

## 分步流程

### 编译时限定可流式节点

仅单个直接下游为终止 Message、模板引用本节点、且无 JSON/结构化输出时，设置 DeferLLMToMessage。中间节点仍等待完整结果。

### Message 打开延迟调用

LLM 返回 DeferredStream；Message 消费时才调用模型 Stream，sync.Once 避免重复 Open 再调用模型。

### 增量与生命周期一致

转发正文与 thinking 增量，完成后把最终结果写回状态；中途失败也完成节点并记录错误，而非假装成功。

## 关键机制

retryInvoker 透传流式接口；首个非空增量之前可重试，已发增量或消费者回调报错后不再重试，避免重复输出。不支持流式的 invoker 仍退回完整 Invoke，不能因接口名称就假定逐 token 到达。

## 来源事实与效果

- 已合并 main，不是新正式版宣告：#21468 merged_at 为 2026-10-26 08:54:51 UTC；提交时间 08:54:50 UTC。release API 最新非 RC 版本仍 v0.27.3（10-23 06:34:45 UTC）。 [来源](https://github.com/infiniflow/ragflow/pull/21468)
- 新增测试覆盖增量、最终状态和失败边界：提交含 mock 流式 E2E、重复 Open 仅调用一次、JSON/结构化门控、发出增量后不重试、回调失败传播等断言。仅审阅源码，未运行测试或独立验证 PR 作者的 PASS 声明。 [来源](https://github.com/infiniflow/ragflow/commit/1133b9fd39f43465eae1e68b24f9bbfb141154c1)

## 局限与未知

- 无真实模型 TTFT、吞吐、并发或成本基准；不能沿用 PR 示意图的零首 token 延迟说法。
- 多 Message 下游、非 Message 中间节点和结构化输出不进入这条延迟流式路径。
- 本轮未安装、未运行 RAGFlow，未核实发布包或生产部署已包含此提交。

## 工程建议，非实测

- 建议在隔离环境比较首个非空事件时间、完整文本一致性、取消与中途故障行为；这是验证建议，不是实测结论。
- 回归至少包括 LLM→Message、Generate→Message、LLM→处理节点、JSON 输出和不支持 Stream 的驱动。
- 不要仅依据 PR 标题推断能力：同日 #21472 实际只改模型选择前端与标签；放开 provider 筛选不代表任意聊天模型兼容决策协议。

## 阅读范围

已完整阅读 #21468 的全部 12 个文件 patch（生产代码和新增/修改测试）、PR 正文与合并元数据、release API。另完整读 #21472 前端 patch；#21485 已读生产差异及部分测试，未据其宣称完整验证。未读仓库全部代码，未运行测试。

## 原始来源

- [RAGFlow #21468 完整差异](https://github.com/infiniflow/ragflow/commit/1133b9fd39f43465eae1e68b24f9bbfb141154c1)
- [#21468 合并记录](https://github.com/infiniflow/ragflow/pull/21468)
- [RAGFlow releases](https://github.com/infiniflow/ragflow/releases)
