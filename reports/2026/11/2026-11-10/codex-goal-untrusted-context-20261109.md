---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-10
verified_at: 2026-11-10T00:10:33Z
---

# 长期任务不能把保存过的目标当成可信指令：Codex #60342 深读

来源时点：PR #60342于2026-11-09合入main；rust-v0.169.0-alpha.7于2026-11-09 21:55 UTC发布。

开放状态：11月9日源码合入及预发布可取；功能开关相关，未运行

核查时间：2026-11-10T00:10:33Z

## 阅读范围与证据等级

已打开官方PR说明、diff中可见的核心上下文/队列/快照变更、固定commit的创建工具与续跑模板及两份测试、官方发布页及预发布标签steering源码。未本地执行、未检验全部CI结果、未做真实模型或对抗实验。

## 核心问题

旧续跑路径把目标正文嵌入续跑模板。模型写入并保存的目标，可能因此在后续回合混入控制指令。此次把目标数据与续跑指令分离，同时修复自动回合附加上下文与输入不能共同进入首请求的问题。与项目trust隔离的边界不同：后者围绕项目配置的信任范围，本次围绕目标内容再注入、活动回合匹配和队列投递。

## 输入

持久目标正文 objective，以及可选token预算。；创建目标的原始turn ID。；续跑所需预算状态、固定控制模板与已有客户端附加上下文。。

## 输出

包含固定续跑指令与独立goal_objective附加上下文的TurnInputRequest。；目标正文标记为AdditionalContextKind::Untrusted。；开启UntrustedAsToolOutput时，目标以agent命名空间下的function_call_output表示；否则走消息表示。。

## 逐步流程

### 1. 创建并保存目标

创建工具清理并验证正文及预算，写入持久状态；随后按工具调用的原始turn ID注入目标上下文。

### 2. 只向所属活动回合注入

持有活动回合锁，要求回合尚未进入结束状态且ID匹配；不匹配即返回，避免迟到结果污染其他回合。

### 3. 增量合并上下文

使用extend更新目标上下文，而不是把其他客户端上下文替换掉。

### 4. 分离目标与控制指令

continuation_request构造固定控制消息，并以goal_objective键附带Untrusted正文；不再把目标正文插入续跑模板。

### 5. 共同进入首次请求

自动/恢复回合把附加上下文与对应输入放进同一待处理队列，使首轮采样同时看到两者。

### 6. 缺失时读回完整目标

续跑模板指示在目标缺失或截断时调用get_goal；完成审计和预算等既有约束仍保留，不能全部算作本次新增。

## 关键机制

变化不只是加一段提示：目标来源标签、上下文存储合并、回合身份验证和输入队列共同调整。旧流程是目标正文进入续跑控制模板；新流程是控制消息引用独立的不可信目标数据，并按开关选择工具输出或消息表示。它减少持久状态被再次包装成可信指令的风险，但不是新的操作系统权限边界。

## 原始来源报告的结果与证据

### 目标在创建和首次续跑时均保留，不重复进入用户消息。

app-server测试启用Goals和UntrustedAsToolOutput，断言两个请求包含目标上下文，且用户消息中不出现测试目标正文；新增快照检查函数输出形态。

[原始证据](https://github.com/openai/codex/pull/60342/files)

### 关闭计划工具提示不会损坏含模板式文本的目标。

steering测试用包含update_plan及标题文本的目标，检查其原样留在Untrusted条目中，控制提示不包含该正文。

[原始证据](https://raw.githubusercontent.com/openai/codex/8703d32/codex-rs/ext/goal/tests/steering.rs)

### 自动续跑首请求获得目标与续跑指令。

Python模拟服务测试检查两者都到达首请求；三个底层请求聚合为一个上层turn生命周期，完成后路由清空。

[原始证据](https://raw.githubusercontent.com/openai/codex/8703d32/sdk/python/tests/test_app_server_goal_operations.py)

### 不只合入main，已出现在预发布标签。

0.169.0-alpha.7发布页标注11月9日21:55、25da1b0；直接打开该标签steering.rs确认存在Untrusted目标管道。

[原始证据](https://raw.githubusercontent.com/openai/codex/rust-v0.169.0-alpha.7/codex-rs/ext/goal/src/steering.rs)

## 限制与未知

- 仅审阅官方代码、发布页和测试断言，没有本地运行测试。
- 这些是数据表示和投递回归证据，不是实际任务成功率、注入攻击成功率或性能提升测量。
- 该版本是预发布；不能推断所有桌面产品、账户或稳定版均默认开启。
- Goals与UntrustedAsToolOutput相关路径需区分；工具输出表示依赖后者，不能无条件声称所有目标都以工具输出传递。
- 未额外核验feature的默认值或面向终端用户的稳定启用方式。
- 不可信内容标记仍依赖模型处理，不等于消除提示注入，也没有提供exactly-once工具副作用保证。
- Python测试涉及私有_client接口，不应直接推荐作为生产SDK接口。

## 工程建议（未实测）

- 最小验证建议：在隔离环境检出rust-v0.169.0-alpha.7，参照现成app-server测试配置启用Feature::Goals和Feature::UntrustedAsToolOutput，发送含模板式文本的无害目标并捕获模型请求。此建议未执行。
- 检查创建后的请求和自动续跑首请求：目标位于goal_objective、控制消息不含目标正文、无关客户端上下文仍保留。另以错误turn ID验证不跨回合注入。
- 对未启用UntrustedAsToolOutput的分支使用消息表示预期；不要把工具输出快照直接用于该分支。
- 自建长期agent可借鉴：持久目标或记忆恢复时保留来源标签；把调度控制与任务数据分开；首请求完整性与回合归属都应有回归测试。

## 原始来源

- [Codex PR #60342](https://github.com/openai/codex/pull/60342)
- [完整变更与app-server测试](https://github.com/openai/codex/pull/60342/files)
- [0.169.0-alpha.7发布](https://github.com/openai/codex/releases/tag/rust-v0.169.0-alpha.7)
- [预发布标签下目标steering实现](https://raw.githubusercontent.com/openai/codex/rust-v0.169.0-alpha.7/codex-rs/ext/goal/src/steering.rs)
- [创建目标工具实现](https://raw.githubusercontent.com/openai/codex/8703d32/codex-rs/ext/goal/src/tool.rs)
- [续跑模板](https://raw.githubusercontent.com/openai/codex/8703d32/codex-rs/ext/goal/templates/goals/continuation.md)
- [steering回归测试](https://raw.githubusercontent.com/openai/codex/8703d32/codex-rs/ext/goal/tests/steering.rs)
- [Python SDK模拟测试](https://raw.githubusercontent.com/openai/codex/8703d32/sdk/python/tests/test_app_server_goal_operations.py)
