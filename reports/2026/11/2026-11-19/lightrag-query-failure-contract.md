---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-19
verified_at: "2026-11-19T00:01:52.312Z"
---

# LightRAG：回答模型故障不再伪装成知识库没找到

## 问题
回答模型超时或上游异常会被 aquery_llm 捕获，变成内容为空的失败字典；库的简化调用和 HTTP 路由随后按空内容处理。于是已经检索到证据的故障请求，仍可能显示知识库无结果。此项不是 11 月 11 日所述最终证据为空门槛，也不改变 11 月 16 日所述重排失败后的候选降级策略。

## 输入
用户问题、查询模式、是否流式以及是否启用进度输出。RAG 模式检索证据并请求回答模型；bypass 直接调用回答模型。

## 输出
正常结果仍为回答或流；真正无结果仍由结构化库接口表示为 no_results，HTTP 保持 200。回答调用失败时，库传播异常；所核 REST 路由在响应尚未开始时返回脱敏 500；已开启的进度流以 NDJSON error 行表达失败。

## 流程
### 构造回答输入
保留 kg_query、naive_query 和 bypass 的原有分支，正常命中构造上下文，真正无结果仍走既有 no_results 分支。

### 让执行失败向调用方传播
移除 aquery_llm 中捕获所有异常后生成空内容字典的分支；aquery 与同步包装不再把这类已抛出的故障变成空答案。

### 根据传输状态反馈故障
无进度的 /query/stream 在建立响应前等待查询；有进度的路径先打开响应，由后台查询任务把错误送回 NDJSON。普通 /query 和 Ollama 兼容 /api/chat 的 RAG 分支也使用已有的安全错误转换。

### 区分用户界面和诊断
客户端正文只返回通用错误和关联 ID，服务端日志用于找对应异常。集成方应把无证据、执行失败、流中断分别记录，不能只根据 HTTP 200 判断成功。

## 机制
本次不发明新的 HTTP 错误助手，而是移除吞错点，让真实异常到达既有边界。对于真正为空的查询，no_results 是结构化结果，不是异常；对于已经开始消费的回答流，迭代器也可能在之后抛错，这是独立的流中失败边界。

## 来源事实与报告效果
### 离线回归直接核对跨层行为
库测试 12 项：naive、mix、bypass × stream 两档 × aquery_llm 与 aquery；路由测试 10 项：普通查询 3 项、无进度流 4 项、有进度流 1 项、Ollama RAG 2 项。用 stub 传输、合成证据和模拟超时，不是真实 API 稳定性实验。

[原始证据](https://github.com/HKUDS/LightRAG/pull/4478)

### 进度流保持 HTTP 200，但机器可读错误
真实 FastAPI 路由测试断言最后一行仅有 error，无 response 行；另检查模拟上游地址、部署名与错误原文未出现在客户端正文。

[原始证据](https://github.com/HKUDS/LightRAG/blob/9f58c8d7ba7c6221c7312e5dd1a42430d4aa686f/tests/api/routes/test_query_failure_status.py)

### 作者报告旧行为失败、新行为通过
PR 作者报告 22 项新增用例在基线全失败、补丁分支全通过；扩展本地子集 2180 通过、14 跳过。未独立运行或将这些数值视为完整 CI 或线上结果。

[原始证据](https://github.com/HKUDS/LightRAG/pull/4478)

## 局限与未知
- 2026-11-18 12:55:01 UTC 合并 main；所核最新 GitHub 公开标签 v1.5.8rc1 发表于 11 月 15 日，不能把本提交写成已随稳定包交付。
- 超时主要被转为通用 500，不保证区分 504；include_progress=true 时不要只查状态码。
- 不改良召回率，也不消除所有内部已经降级的错误；若重排内部正常回落，外层未必有异常可传播。新增测试重点是回答调用失败，不代表各种底层提供方或流迭代故障均已验证。

## 工程建议（未实测）
- 迁移时为 aquery、aquery_llm 及同步接口加显式异常处理；不要把任何空字符串或 status=failure 都视为同一种失效。区分 no_results、回答为空和执行异常。
- 用合成证据做三组契约对照：真正零证据；检索命中后回答超时；先发进度再失败。记录 HTTP 状态、流中 error、库异常和无结果指标。再加一组流已输出部分答案后中断，验证 UI 不误标完整。

## 阅读范围
已核对 GitHub PR 实时合并状态、5 文件完整 diff、固定提交的查询路由处理段和两期历史归档。代码与作者离线回归记录审阅；未运行测试、线上服务、模型或生产迁移。

## 原始来源
- [PR #4478：合并时间、行为对照和作者回归](https://github.com/HKUDS/LightRAG/pull/4478)
- [固定提交：aquery_llm 实现与调用契约](https://github.com/HKUDS/LightRAG/blob/9f58c8d7ba7c6221c7312e5dd1a42430d4aa686f/lightrag/lightrag.py#L5986-L6131)
- [固定提交：HTTP 与流式包装](https://github.com/HKUDS/LightRAG/blob/9f58c8d7ba7c6221c7312e5dd1a42430d4aa686f/lightrag/api/routers/query_routes.py#L610-L983)
- [固定提交：API 离线回归](https://github.com/HKUDS/LightRAG/blob/9f58c8d7ba7c6221c7312e5dd1a42430d4aa686f/tests/api/routes/test_query_failure_status.py)
- [固定提交：库离线回归](https://github.com/HKUDS/LightRAG/blob/9f58c8d7ba7c6221c7312e5dd1a42430d4aa686f/tests/llm/test_query_llm_failure_raises.py)
- [公开发布标签](https://github.com/HKUDS/LightRAG/releases)
