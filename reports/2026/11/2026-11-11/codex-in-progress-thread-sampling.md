---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-11
---

# 运行中采样的边界：观察副本，不接管源任务

核查时间：2026-11-11T00:11:03Z

来源时点：PR #60620于2026-11-10合入main，合并提交c37fd5665f55fa2b5accdf1d229855416ce696c1；rust-v0.169.0-alpha.9于2026-11-10 22:09 UTC发布。已逐文件核验该标签包含变更；同日稳定版rust-v0.167.1仍为旧行为。

开放范围：11月10日合入及预发布可取；实验接口；源码与模拟测试审阅，未运行

## 阅读范围与证据边界

已读取日报2026-11-01至2026-11-10的条目标题/发现和相关覆盖说明，排除长期目标注入、持久恢复、压缩计数及旧Message Threads等重复主题。
已读取PR #60620完整变更、固定提交元数据、alpha.9发行记录、alpha.9协议与完整thread_sample实现；对照稳定版0.167.1协议和实现。
已读取alpha.9 thread_sample测试文件，重点核对新增运行中/压缩用例及既有工具拦截、Stop hooks、无效输入和清理测试；核对common.rs实验接口注册。
已定向检索OpenAI Codex、Agents Python、Anthropic Agent SDK及Deep Agents于11月10日合并PR；查Claude Code发布页、Anthropic工程索引及Agents Python发布页。未穷尽全部仓库或所有未发行改动。
未安装或执行任何被研究项目，未运行模型或测试，未使用私有数据。
补核固定alpha.9的LICENSE、NOTICE、README、安装构建文档、Rust工具链、app-server Cargo清单及CLI入口，区分预编译运行与源码构建/测试依赖。 本期最终核查时间：2026-11-11T00:11:03Z。

## 核心问题与已有瓶颈

自建Agent常需要回答“现在进展如何、下一步可能做什么”，却不希望把这种旁路请求排进正在执行的主回合。此前thread/sample必须等待源线程空闲，并要求指定最新已完成、失败或中断的回合；采样期间源历史变化也会使结果被拒绝。本次只放开显式请求的运行中快照，不改变工具执行权限或主任务调度。

## 输入

已加载、持久化的根线程threadId，以及调用时预期的最新expectedTurnId。临时线程、非根Agent仍不满足源线程要求。

prompt为非空字符串，最多4,000字节。新增allowInProgressTurn默认false；true才允许运行中源回合，并取消结束时的源历史版本一致性检查。

可选effort只在模型支持推理强度更新且使用OpenAI provider时生效。disableMcp不是无条件禁用所有MCP的开关，它只针对Responses Lite和已有增量工具目录的特定条件。

## 输出

一次模型响应的sourceTurnId、responseId、text及可选的inputTokens、cachedInputTokens和outputTokens。

如果响应尝试工具调用、包含非文本输出或没有有效文本，text可为null；不能把null解释为主任务失败。

采样响应不会自动写回主对话或更新主任务。源线程可在副本响应返回前继续推进，因此结果代表捕获的历史，而非返回瞬间的最新状态。

## 逐步流程

### 1. 校验入口与源身份

thread/sample在协议中仍标记为experimental。实现检查提示长度、持久根线程以及expectedTurnId是否为最新回合。未显式启用allowInProgressTurn时仍拒绝Running源线程。

### 2. 捕获上下文而非暂停源线程

实现刷新源rollout并读取最新模型上下文。Legacy模式从历史回合判断状态，Paginated模式另查最新回合状态；这使已发生回合内压缩、上下文未包含turn start的情况也能判断。

### 3. 建立临时副本并标注不完整工具状态

沿用原线程配置和设置创建ephemeral副本，禁止副本自动压缩。运行中捕获的待完成工具调用在复制历史里可被补为aborted；新增thread_snapshot上下文明确这是副本占位、源任务并未停止，并抑制副本默认中断提示。

### 4. 拦截一次响应，再清理副本

已有ModelResponseInterceptor在Core执行模型输出前截获响应。工具调用会令text为空，而不是启动一条新的工具执行链。接着关闭并移除临时线程；超时、取消和客户端交互需求走错误路径。

### 5. 接受有明确时点局限的结果

当allowInProgressTurn为true时，源历史在采样期间变化不再让结果失效。新测试故意先让主工具返回、主回合完成，再释放副本响应，检查两条路径独立。

## 关键机制与真实增量

实质差异是准入条件和快照语义，不是新建一个可持续执行的并行Agent。0.167.1稳定源码没有allow_in_progress_turn，要求空闲且结束时检查源history_version；alpha.9在显式开启后允许运行中的最新回合并跳过结束时版本检查。副本中未完成工具的aborted与源线程真实工具结果是两种记录，新增上下文注释负责避免把复制处理误读为用户中断。一次采样响应、工具输出拦截、临时线程清理等机制在旧实现里已有，本次沿用。

## 来源报告的效果与证据

### 新能力已进入可取的预发布源码，但未进入同日稳定版。

alpha.9协议和实现均包含allow_in_progress_turn；对照rust-v0.167.1的相同实现文件，仍直接拒绝Running源线程且没有该字段。发行页标记alpha.9为Pre-release。

[原始证据](https://github.com/openai/codex/releases/tag/rust-v0.169.0-alpha.9)

### 新增测试覆盖主任务继续执行和副本只见捕获状态。

sample_in_progress_turn_uses_captured_history覆盖Legacy、Paginated及Paginated回合内压缩三种情形：先挂起副本响应，再向源工具回传真实结果并断言源回合Completed，最后释放副本。子请求有aborted占位及一个thread_snapshot注释，父请求只有真实工具结果。

[原始证据](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server/tests/suite/v2/thread_sample.rs)

### 另有回合内压缩但无待完成工具的采样用例。

sample_after_mid_turn_compaction_without_a_pending_tool_call比较采样前后父rollout字节不变，检查继承摘要且没有工具调用项，再释放源响应并断言完成。测试文件还把已结束源回合的上下文保持矩阵扩展到选项false和true。

[原始证据](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server/tests/suite/v2/thread_sample.rs)

### 一次响应和不执行采样工具调用属于既有保护，不是本次新功能。

alpha.9及0.167.1实现均有SampleInterceptor；现有sample_blocks_mcp_execution_without_changing_parent_tools测试统计MCP tools/call仅发生源线程的一次，采样返回text:null。现有Stop hook测试检查采样没有再触发该hook，但跳过Windows和远端环境。

[原始证据](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server/tests/suite/v2/thread_sample.rs)

### 本轮没有可报告的实测任务收益或全量测试通过结果。

本轮只读发布、协议、实现、测试和差异。PR页面出现“7 of 8 checks passed”，不能解释为本轮运行通过或全平台验证；提交combined-status接口返回空列表，也不足以确认整套CI。

[原始证据](https://github.com/openai/codex/pull/60620)

## 限制与未知

- 只核验实验性app-server协议。初始化需允许实验API，并且调用要明确allowInProgressTurn:true；不能据此宣称桌面界面、CLI命令或Python Agents SDK已提供用户入口。
- 快照不保证返回时最新，也不保证与外部世界状态一致。不要依据副本aborted自动重试付款、消息或写入等真实工具，更不要据此判断源任务已中断。
- 模型可能尝试工具而得到null文本；它不是持续研究分支、状态回滚、工具重放、审批或执行承诺。
- 沿用源配置会发起额外模型请求；源代码的30秒超时、4,000字节prompt及64KiB文本输出上限是实现约束，不是延迟或成本实测。现有disableMcp有条件，不能把工具调用拦截泛化为绝无任何初始化或网络活动。
- 测试主要采用模拟Responses/SSE及假MCP；没有提供真实观察准确率、主任务速度影响、缓存收益、竞态穷举或跨平台完整结果。本轮未构建、运行测试或连接真实模型。

## 工程判断、最小试用与建议

- 最小试验：在隔离目录固定rust-v0.169.0-alpha.9，使用官方app-server测试夹具和假动态工具，不接真实外部动作。完成initialize/initialized握手，initialize参数中设capabilities.experimentalApi:true；随后用thread/start建立持久根线程，再以turn/start启动任务，记录返回的turn id并让假parent_tool保持未答复。稳定版0.167.1没有该选项，不应直接套用此请求。
- 发起thread/sample请求：{ "threadId": "源线程ID", "expectedTurnId": "最新回合ID", "allowInProgressTurn": true, "prompt": "只总结已完成事实，并明确列出还不知道的工具结果。" }。应在模型请求记录里看到副本thread_snapshot说明及工具占位，而源回合仍在运行。
- 随后给源parent_tool返回固定测试文本，在副本响应仍挂起时检查源回合可完成；最后释放副本，核对sourceTurnId。不要期待副本看见捕获之后的工具结果。
- 对照组：省略allowInProgressTurn应拒绝运行中源；传旧turn id应拒绝；模拟采样模型发出MCP或apply_patch调用时应得到null文本，确认没有额外tools/call或目标文件写入。另查取消、超时后的临时线程清理。
- 若用于进度面板，为每条采样显示采集时点与对应turn id，区分已观察事实、待完成工具及模型推测。是否据此改变主任务必须另设控制接口和授权边界，不把采样结果直接当执行指令。
- 只有完成本地回归，再用少量非敏感真实任务比较观察质量、额外tokens与主任务延迟。本轮建议不等于已验证收益或上线建议。
- 许可证与部署前提：固定rust-v0.169.0-alpha.9源码采用Apache-2.0；重分发应保留LICENSE及适用NOTICE、标注修改，NOTICE还列出Ratatui的MIT归属。最小部署可用该预发布的平台二进制运行codex app-server，CLI源码确认默认stdio传输；另需能够处理初始化、请求和通知的客户端，以及可写的配置/历史目录。真实模型采样还需已授权provider的网络与凭据，并产生额外推理用量，开源许可不含推理额度。若自行编译，该tag固定Rust 1.95.0，走Cargo工作区；文档的just、DotSlash、cargo-nextest为开发/测试辅助工具，不应混作预编译二进制的运行时依赖。先做本地stdio及mock回归，不以本条作为公网部署安全说明。

## 原始来源

- [PR #60620：允许对运行中回合采样及合并日期](https://github.com/openai/codex/pull/60620)
- [0.169.0-alpha.9预发布记录](https://github.com/openai/codex/releases/tag/rust-v0.169.0-alpha.9)
- [alpha.9：thread/sample实现与输出拦截](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server/src/thread_sample.rs)
- [alpha.9：请求和响应协议](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server-protocol/src/protocol/v2/thread_sample.rs)
- [alpha.9：运行中快照、工具拦截和清理测试](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server/tests/suite/v2/thread_sample.rs)
- [alpha.9：实验接口注册](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/app-server-protocol/src/protocol/common.rs#L593-L604)
- [0.167.1稳定版：仍拒绝运行中源线程](https://github.com/openai/codex/blob/rust-v0.167.1/codex-rs/app-server/src/thread_sample.rs)
- [固定合并提交](https://github.com/openai/codex/commit/c37fd5665f55fa2b5accdf1d229855416ce696c1)
- [alpha.9源码许可证](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/LICENSE)
- [alpha.9第三方归属声明](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/NOTICE)
- [alpha.9 CLI app-server入口及默认stdio](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/cli/src/main.rs#L579-L608)
- [alpha.9安装与模型认证说明](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/README.md)
- [alpha.9构建及开发工具说明](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/docs/install.md)
- [alpha.9固定Rust工具链](https://github.com/openai/codex/blob/rust-v0.169.0-alpha.9/codex-rs/rust-toolchain.toml)
