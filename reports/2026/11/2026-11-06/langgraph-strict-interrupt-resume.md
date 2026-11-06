---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-06
---

# LangGraph 1.2.20：过期审批显式失败，连续中断使用不同身份

核查时间：2026-11-06T00:04:50Z

## 问题
等待人工审批的持久图可能收到过期、重复或歧义答案。旧答案被静默忽略或被连续下一次中断误用，会让应用误以为审批已正确恢复。

## 输入与输出
输入：带 checkpoint 的 LangGraph、当前待处理 interrupt ID、恢复值和严格校验配置；远程调用还依赖服务端能力声明。

输出：有效答案交给对应中断；无效恢复在写入前抛 UnknownInterruptError，携带原因、请求 ID 和待处理 ID。

## 流程

### 记录中断
检查点保存图状态，客户端从当前状态或流中取待审批 interrupt.id，并让审批动作指向该身份。

### 恢复前校验
根图在生成 command 写入前检查 pending IDs；严格模式下拒绝不存在或已消费 ID、空映射，以及多个中断时没有 ID 的歧义单值。

### 区分连续问题
以命名空间与已获得答案数量生成中断身份，让同一节点连续问题不同；旧 checkpoint 的首个 ID 提供兼容回退。

### 重试与恢复
节点重试从头执行时重置计数器并复用已记录答案；应用捕获错误后刷新当前审批，而不是把旧答案自动改挂到新问题。

## 机制
在 checkpoint 管理的 pending interrupt 集合上进行严格匹配，加上按答案槽构造的新 ID，防止重复审批静默进入后续问题；不是对外部业务操作的 exactly-once 保证。

## 作者结果与证据

- 严格模式可区分 not_pending、empty_map 与 ambiguous。 固定版本实现及测试覆盖无效 ID、空映射和并行多中断单值；失败后断言 checkpoint 数量和状态不变。 [原始来源](https://github.com/langchain-ai/langgraph/blob/1.2.20/libs/langgraph/tests/test_strict_resume.py)

- 作者报告核心回归测试通过，但不是独立生产验证。 PR #9574 报告 memory/SQLite 和不同 durability 的44场景；#9589报告核心2886及prebuilt 256测试通过，测试环境 NO_DOCKER=true；本次仅读代码与测试。 [原始来源](https://github.com/langchain-ai/langgraph/pull/9589)

- SDK 新参数合并不等于已进入当日发布包。 SDK PR 在 0.4.13 发布后合并；远端 /info 必须声明能力，旧服务端可能忽略字段。 [原始来源](https://github.com/langchain-ai/langgraph/pull/9599)

## 局限和未知
- 严格模式默认关闭；核心1.2.20通过私有 configurable 键启用，不能视为稳定公共 API。
- 显式回放已被后续状态替代的历史 checkpoint 不检查；嵌套图由根图处理，静态中断另有兼容行为。
- 同一 task 的并发中断按实际抛出顺序处理；仍需权限校验、外部副作用幂等和恢复策略。
- Python SDK 0.4.13 发布早于参数合并，不能说升级至此版即可使用；未独立运行。

## 工程建议（未经实测）
- 隔离环境先构造连续两问、重放旧ID、空映射、并行和节点重试；检查拒绝后状态是否保持。
- 对外部写操作仍使用业务幂等键，并记录审批对象和版本；不要自动把旧答案套到新的审批问题。
- 正式远程部署先确认 SDK 版本和服务端 /info 能力；本条不建议依赖私有键作为长期契约。

## 阅读范围
已读核心固定版本实现、测试和三个 PR；作者报告的测试没有独立运行，也未测试远程服务。

## 原始来源
- [核心 1.2.20 发布](https://github.com/langchain-ai/langgraph/releases/tag/1.2.20)
- [严格恢复校验 PR](https://github.com/langchain-ai/langgraph/pull/9574)
- [连续中断 ID 修复 PR](https://github.com/langchain-ai/langgraph/pull/9589)
- [Python SDK 新参数 PR](https://github.com/langchain-ai/langgraph/pull/9599)
- [固定版本测试](https://github.com/langchain-ai/langgraph/blob/1.2.20/libs/langgraph/tests/test_strict_resume.py)
- [检查实现](https://github.com/langchain-ai/langgraph/blob/1.2.20/libs/langgraph/langgraph/pregel/_loop.py#L1026)
