---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-28
---

# Codex 0.165.0 附着线程分叉：给研究主线建立可持久回看的分支，而非另一个自动子代理

核查时间：2026-10-28T00:07:08Z

## 核心问题

算法研究常需要沿同一上下文探索多个假设，既希望保留来源关系，又不希望每个辅助分支占满普通会话列表。已有 thread/fork 可以复制会话历史；本次新增不是“第一次可以分叉”，而是 attachToSource：把持久分支附着到来源线程，并提供专用发现与删除依赖规则。它解决会话组织和生命周期问题，不直接提升模型推理质量，不自动运行实验或合并结论。

## 输入

不涉及训练。输入包括一个已持久化的本地来源线程 ID、thread/fork 参数、启用 experimentalApi 的客户端连接，以及支持本地存储与 SQLite 状态库的 app-server。试用应选择专门建立的无隐私测试会话，不导入真实聊天或私人仓库。后续要让分支生成内容，还需可用模型、账户权限和预算。

## 输出

thread/fork 返回新的 thread.id 及来源信息；分支从普通 thread/list 隐藏，使用 attachedToThreadId 才能专门列出。附着关系可持久化，之后可以 resume 并启动 turn。它输出的是会话状态和后续模型内容，不是实验结果真值、权重、Git 分支或已隔离的工作目录。

## 分步流程

### 1. 固定发布版本并完成协议握手

工程建议：在一次性环境安装/选择 Codex 0.165.0，核验 codex --version，以 codex app-server 启动默认 stdio JSONL 连接；发送 initialize，clientInfo 使用自己的客户端标识，capabilities.experimentalApi=true，成功后发送 initialized。输入为明确版本与客户端能力，输出为可以使用实验字段的连接。不要为测试开放公网 WebSocket。

### 2. 准备没有自动目标的测试来源

工程建议：创建并完成一个只含公开虚构材料的普通本地线程，保存返回的来源 ID；来源不要设置自动延续目标。先通过读取接口确认它确实可恢复，再进行分叉，避免把未持久化或远程存储线程当成有效输入。输入为一段研究背景，输出为可追溯的来源会话。

### 3. 请求附着分支并记录响应

发送 thread/fork，params 为 {"threadId":"实际来源ID","attachToSource":true,"excludeTurns":true}。excludeTurns 只是精简返回、不在响应里装载历史，不能理解为“分支没有继承上下文”。记录响应里的新 ID；不要等待 thread/started 来判断是否创建成功，因为附着分叉特意不发送这条通知。

### 4. 分别验证普通列表和专用列表

先 thread/list 使用普通参数，再使用 {"attachedToThreadId":"实际来源ID","limit":10} 并按 nextCursor 翻页。预期前者不列附着分支、后者能找到它；不要用 parentThreadId 或 ancestorThreadId 代替。输入是来源 ID 与分页参数，输出是显式关联清单，而不是认为“普通列表没有就是创建失败”。

### 5. 恢复、分析与回收分开处理

对新 ID 调用 thread/resume，必要时通过 turn/start 让它分析一个替代假设。工程建议：把结果人工或通过自建审阅逻辑归纳回主线；本机制没有自动合并。重启后的列出/恢复以及删除保护只在可丢弃测试会话验证，先删除附着分支再删除来源；不要删除真实研究历史。

## 关键机制

0.165.0 于 2026-10-27T20:09:38Z 发布（上海 10 月 28 日 04:09:38）；相关 PR 在 10 月 23 日已合并，此次变化是进入可安装发布版，接口本身仍实验性。协议新增 attachToSource 与 attachedToThreadId；前者省略/false 仍是普通分叉，后者把附着关系与 spawned-agent 的父子关系分开。实现先检查本地存储和状态库，拒绝与 ephemeral 或 deferGoalContinuation 同用；列表过滤也拒绝和 parentThreadId/ancestorThreadId 混用。附着元信息写入持久历史与状态库，删除路径会检查依赖关系。这里的“附着”是持久组织关系，不是共享执行状态、自动消息回传或文件系统沙箱。

## 来源事实与效果

- 新接口已进入 0.165.0，但仍须实验能力开关。：官方 release 将 attached thread forks 列入 New Features；tag 内协议对两个新增字段使用 experimental 标注。发布时间取自官方 GitHub release HTML 的 relative-time datetime。无模型跑分、数据集或论文表号。 [来源](https://github.com/openai/codex/releases/tag/rust-v0.165.0)
- 启动与列表行为可在发布版本实现中检查。：thread_processor.rs 的 fork 处理仅在 !attach_to_source 时发 ThreadStarted；列表处理把 AttachedThreadsOf 单独作为关系过滤，并验证互斥参数及 LocalThreadStore。 [来源](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/app-server/src/request_processors/thread_processor.rs#L5294-L5309)
- 作者提交了恢复与空分支的回归测试，而非只给演示。：attached_thread_tests.rs 的参数化测试列出 4 种组合：legacy/restart、paginated/restart、legacy/missing database、paginated/corrupt database。测试建立两个尚未运行 turn 的附着分支，检查普通列表隐藏、limit=1 分页发现、恢复、再运行 turn 和删除顺序。本期只阅读源码，没有执行，因此不声称独立复现通过。 [来源](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/app-server/tests/suite/v2/attached_thread_tests.rs#L35-L41)
- 非法组合和删除依赖具有可检查的拒绝路径。：另一测试验证 ephemeral、deferGoalContinuation 以及两种关系过滤冲突，并检查失败后无可发现残留；本地 delete_thread 在生命周期锁内再次检查附着子项。它是作者测试/实现证据，不是任意故障下绝无丢失的证明。 [来源](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/thread-store/src/local/delete_thread.rs#L33-L59)

## 局限与未知

- 实验性 app-server 字段，不是默认 ChatGPT UI 功能，也不是所有远端 thread store 都支持。
- 当前官方 App Server 总览尚未检索到 attachToSource 字段，具体新增行为以 0.165.0 tag 协议、实现和测试为准；旧客户端需更新协议识别。
- thread/fork 已有能力不能算本次创新；本次新增的是附着持久关系、发现与生命周期规则。
- 普通列表隐藏不是加密、访问控制或隐私保证；有 ID 和访问能力的客户端仍可能读取。
- 附着分支不是 Git worktree。多个分支若写同一工作目录，冲突仍需使用单独工作区、权限和显式整合策略处理。
- 没有作者报告的吞吐/延迟/token节约/研究成功率对照数据；不能宣称分叉必然更省或更准确。
- 未运行官方回归测试，未测模型推理、真实断电、远程后端和多客户端并发；源码测试只说明覆盖意图。
- fork 会复制已有上下文；对内容最小化有要求时，应先在来源侧准备可分享的公开上下文，而不是误用 excludeTurns 当隐私过滤。

## 工程建议，非实测

- 适合试做“同一基线下两个候选方案”的研究组织：主线保存问题与验收标准，附着分支分别分析方案，主线只接收经过审阅的结论与来源。此为工程建议，非官方证明的最优流程。
- 建议最小验收表只核对四件事：分叉响应 ID、两种列表差异、重启后可恢复、清理依赖保护。先证实状态语义，再考虑让分支调用昂贵训练任务。
- 若要做算法实验，另行保存数据版本、代码提交、参数、种子、运行产物与评价脚本；会话分叉不能替代实验追踪系统。
- 客户端要同时处理响应和事件；不能把没有 thread/started 当失败并盲目重发，否则可能新建多个持久分支。网络结果不明时优先用专用列表与已保存 ID 核对。

## 阅读范围

已读取官方 0.165.0 完整发布说明及 PR #56351 描述；阅读全文较短的 attached_thread_tests.rs；研读 tag 内 thread.rs 对应协议、thread_processor.rs 的分叉与列表分支、delete_thread.rs 的删除检查，以及官方 App Server 初始化/传输/实验接口文档。未声称通读整个 Codex 仓库或执行测试。本条为技术实现研读，非论文全文研读。

## 原始来源

- [Codex CLI 0.165.0 官方发布说明](https://github.com/openai/codex/releases/tag/rust-v0.165.0)
- [PR #56351：experimental attached thread forks](https://github.com/openai/codex/pull/56351)
- [0.165.0 线程协议定义](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/app-server-protocol/src/protocol/v2/thread.rs)
- [0.165.0 请求处理实现](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/app-server/src/request_processors/thread_processor.rs)
- [0.165.0 附着线程测试源码](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/app-server/tests/suite/v2/attached_thread_tests.rs)
- [0.165.0 本地线程删除保护](https://github.com/openai/codex/blob/rust-v0.165.0/codex-rs/thread-store/src/local/delete_thread.rs)
- [官方 App Server：握手与实验接口](https://learn.chatgpt.com/docs/app-server)
