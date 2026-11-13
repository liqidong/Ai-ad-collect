---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-13
---

# Qwen Code持久自定节奏唤醒：把/loop从计时器变为可恢复的会话输入

核查时间：2026-11-13T00:13:34Z

来源时点：设计文件名为2026-11-10（不是发布日期）；PR于2026-11-12 18:08:54 UTC合并；nightly于2026-11-12 22:27:16 UTC发布；分别为北京时间11月13日02:08:54、06:27:16。

开放范围：11月12日UTC合并并发布nightly；北京时间11月13日；默认关闭的预发行实现

## 阅读范围与证据边界

核查时间：2026-11-13T00:12:15.769204+00:00。已读官方PR合并信息与对应nightly发布页；完整阅读设计文档、HostedLoopSession实现和authority.loop测试；研读authority、wake pump、权限分类器、工具profile/dispatch相关变更以及loop与调度器关键测试。读取package.json、pom与LICENSE相应字段。没有完整通读整个仓库或所有65个变更文件，没有安装/运行测试、连接模型或做生产故障演练；本文是原始代码/文档研读，不是论文实验复现。公开历史10月9日至11月12日报已查未发现同项。 本期最终核查时间：2026-11-13T00:13:34Z。

## 核心问题与已有瓶颈

普通进程内循环或固定间隔计时器把“何时继续工作”绑定在正在运行的程序上：程序退出后计划可能丢失，重新安排与取消可能竞态，轮询会消耗资源，也难以回答重启后某次唤醒是否仍应执行。Qwen这次为Managed Sessions补齐一个一次性的、由模型主动续约的持久唤醒输入。它不是长期Agent全部架构：任务规划、模型可靠性、远程执行、外部副作用和常驻服务运维仍要另外解决。相对旧hosted-workspace-shell/1，新增shell/3及loop_wakeup/cron_list/cron_delete工具；shell/2已退役，不能混用。相对11月12日报的QuickJS文件后端，本项解决的是时序与恢复，不是文件执行接口。

## 输入

不涉及训练或新权重。运行输入为当前Managed Session、持久journal与资源存储、正在执行turn的权限分类器，以及loop_wakeup的delaySeconds（数字）、prompt（非空且最多10000字符）、可选reason。delaySeconds钳制为60至3600秒，非有限数内部回退1200秒。部署必须显式打开QWEN_MANAGED_AGENT_LOOP_ENABLED=true，QWEN_HOSTED_LIFECYCLE_V2_PROFILES须含hosted-workspace-shell/3，Session实际使用该profile；还需要可用的模型配置和认证。动态sentinel模式可读取用户级loop.md等约定指引，但此Hosted实现把项目文件信任锁为false，不读取仓库内.qwen/loop.md。

## 输出

安排成功返回唯一loop:<UUID>标识、fireAt、钳制后的延迟、是否钳制与被替换的旧ID；持久化形成input.accepted和wake.requested事件，替换时还包含旧turn.settled。到期后输出一条新的模型输入并进入已有turn执行管线，而不是直接执行Shell命令。cron_list返回未启动的待唤醒条目；cron_delete返回是否取消。模型只有再次调用loop_wakeup才续期，不调用即停止。工具名cron_list不意味着此次已提供完整固定周期cron实现。

## 逐步流程

### 1. 设置能力与输入检查

输入为已有Managed Session和部署配置。shell/3声明loop工具；安排入口再检查默认关闭的环境开关，清理prompt、检查长度、钳制延迟。输出为合规的候选唤醒请求。关闭开关只阻止新安排，不会自动取消已经提交的唤醒，已有条目仍能列出/取消。

### 2. 将未来提示作为受审查动作

输入为delaySeconds/prompt/reason和当前turn的上下文。复用本turn权限分类器，审查未来模型输入，分类器拒绝或不可用时不提交新计划；本次拒绝不会破坏原有待唤醒项。分类器在turn结束清除，避免后续操作借用已结束turn的权限闭包。输出为允许的请求及审查结果，不是免审批任意自主执行。

### 3. 计算链预算并事务化替换

从journal取本Session第一次loop输入的occurredAt作为24小时预算起点，计算fireAt。越界则拒绝新安排，并结算原有未触发项为loop_chain_expired；取消与重启均不重置起点。允许时先写提示资源，再把旧项loop_replaced结算、新input.accepted、新wake.requested放在一个串行journal提交中，notBefore=fireAt且deadline为空。输出是一份可恢复的时间约束，不是当前进程独有的计时器。已经有执行记录的turn不属于可替换未触发项。

### 4. 从持久状态挑选到期输入

调度器输入为journal待处理项与当前时间。尚未到notBefore的loop跳过；没有可运行输入时，为最早未来时间设置单个deferral timer，不按500毫秒反复调用next。新的kick会重算并清除过期计时器，已到期的其他输入不会被未来loop挡住；关闭会清理计时器。重启后重建待处理项及到期判断。此优化局限于该wake pump，不代表控制面全部停止轮询。

### 5. 到期解析动态指引并重新审查

普通prompt原样交付，不重复触发时审查。动态sentinel在到期时解析为完整指引，首次或内容变化时调用Session自身模型配置重新分类，避免安排时只审查标记却执行后来改变的文本。明确拒绝则结算loop_admission_denied、不进入模型执行；缺少必要分类器也闭锁。分类器临时不可用或文件临时故障时不交付未审查正文，而给模型空操作重排提示。只有成功交付后才确认缓存；不变内容后续可缩短为提醒，压缩/重启后需完整重读和审查。

### 6. 执行、续约和停止

输入为已接纳的模型turn。模型完成一次工作后自主决定是否及何时再次安排，工具返回结果也进入会话记录；后台任务已会发通知时，工具建议用1200至1800秒作为兜底而不是高频轮询。cron_delete只能结算尚未启动的唤醒；已开始的turn要走另一个运行中取消通道。若当前tick崩溃且未成功续约，不能假定系统自动补下一次。

## 关键机制与真实增量

关键不是循环提示词，而是把“未来再次输入”并入已有Session事件协议。notBefore表示最早可执行时间，不能当超时deadline；accepted/settled与wake事件共同决定恢复后是否还有未处理工作。替换的三个事件在同一提交中保持一致，避免旧计划取消成功而新计划未写入的夹缝；固定turn事件ID约束重复结算。调用方负责排除已开始的项，不能把这解读为所有任意调用天然具有exactly-once保证。权限边界分为安排时审核和动态内容触发时审核，后者使用临时安全配置、禁用hooks/MCP发现等，并始终不信任项目loop文件。未启动pending最多一个，但已执行中的旧turn与下一次待唤醒项可以同时存在。

## 来源报告的效果与证据

### 新增代码确实合并且存在对应预发行下载，不只是设计提案。

PR #16835于11月12日18:08:54 UTC合并为70a487be84b2a14daae21b9c2369b0ad243646dd；同前缀nightly于22:27:16 UTC发布，标为prerelease，包含cli.js及各平台包。不能把同日更早的v1.0.0-preview.1当作包含此变更，也不能因GitHub latest返回SDK标签就宣称CLI正式版支持。

[原始证据](https://github.com/QwenLM/qwen-code/releases/tag/v0.27.2-nightly.20261112.70a487be84)

### 原子替换与恢复有可检查的程序测试证据，但没有真实长程任务成功率。

authority.loop测试文件含6项断言，覆盖notBefore、旧输入兼容、三事件原子序列、旧项结算与新pending、重复替换拒绝和拒绝后journal不变。使用临时目录、真实LocalJsonl journal/资源存储及SessionWriterLease；这是本地组件回归，不是真实模型或生产分布式故障演练。本轮未执行，不能写6/6已复现通过。

[原始证据](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/core/src/managed-runtime/managed-session-authority.loop.test.ts)

### 自定节奏的边界可在代码中逐项审计。

HostedLoopSession测试用注入时间与allow/deny分类器覆盖59秒变60、3601变3600、NaN变1200，24小时边界允许而越界拒绝，取消不重置锚点、journal重开保留锚点、并发安排串行化、已启动项不可取消，以及动态正文重新审查/缓存失效/临时故障。它们证明测试设计覆盖边界，不能替代真实API可用性或模型服从续约指令的测量。

[原始证据](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-loop-session.test.ts)

### 避免未来唤醒项引起500毫秒重试轮询是局部调度改造，不是编码效率实验。

新增5项fake timer测试：最早未来时间单计时器、已有到期输入优先、kick重算、关闭清理、运行中关闭不遗留计时器。其中等待3600秒案例先推进5000毫秒断言next仍只调用1次，再推进到期。没有真实吞吐、成本或延迟分位数数据，没有数据集/benchmark表格。

[原始证据](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-monitor-wake.test.ts)

## 限制与未知

- 状态为默认关闭的公开源码+nightly预发行能力。完整Managed Session部署、模型认证、持久存储与宿主服务缺一不可；不能仅安装普通CLI就推断托管调度已开启。
- 24小时是Session首个loop事件起算的不可自动重置锚点，而不只是本次取消前的连续运行时长；长寿命Session后续再次开启可能仍被拒绝。需产品上明确新Session/预算策略，不要偷偷绕开限制。
- 取消只处理尚未开始项；不能靠cron_delete停止当前工具或正在运行turn。关闭新安排开关也不会清空已提交工作。
- 单个journal事务不等于外部副作用exactly-once。网络结果不明、已执行工具崩溃、模型漏续约、控制面多副本恢复、资源冲突仍须额外设计与测试。
- 权限分类器依赖模型调用及认证，会增加耗时/费用；不可用时给出的只是no-op重排指导，是否真正续约仍依赖模型执行。未见真实任务成功率、误主动率、token成本和长时间稳定性指标。
- 设计文档仍保留proposal/not implemented字样，PR中英文对触发时审查的进度有不一致；本解读以固定合并源码和测试为准。Java侧fixture replay parity仍为待完成项，不能把TypeScript局部测试推广为完整跨语言重放一致性。
- 源码Apache-2.0；模型API、接入服务及第三方依赖遵守各自条款。package.json要求Node>=22，Managed Agent Server pom要求Java21且使用Spring Boot3.5.16，完整部署成本高于直接复用几个TS类。
- nightly发布页和下载资产存在，不代表已解包核验每个二进制内的Managed Session入口，也不证明部署路由自动开放；本轮未下载运行发行物。PR作者还记录基线typecheck已有zod相关错误，不能把变更文件检查或测试计划写成全仓绿灯。

## 工程判断、最小试用与建议

- 最小试用计划（未执行）：固定上述SHA或nightly，在不含私人凭据/数据的隔离环境先运行官方authority.loop、hosted-loop-session和hosted-monitor-wake测试。以临时journal复核原子替换、拒绝不落盘、跨重开24小时锚点，再进入实际服务配置。
- 交互试验（未执行）：用具备shell/3的Managed Session及正确部署开关，请模型调用loop_wakeup({delaySeconds:60,prompt:"/loop 只读取指定公开测试文件是否存在；报告后不再续约",reason:"验证一次持久唤醒"})。用cron_list记录ID和fireAt；到期前正常重启Harness并保持journal/Session不变，确认只发生预期检查。再安排第二次覆盖第一次、取消并读回，确认被取消项不执行。示例是模型工具调用形状，不是可直接在系统Shell执行的命令。
- 应独立做故障矩阵：安排提交后响应丢失、到期前后宿主退出、分类器超时、文件不可读、压缩后重评、取消与启动竞态；记录重复/漏执行、恢复正确性、取消延迟、token与费用。未通过前仅用于只读低风险工作，不授权付款、公开发布或修改共享数据。
- 适合复用journal事件+notBefore+原子替换思路，为长程实验观察器构建可恢复的一次性调度；不是自动获得Super-agent能力。若已有持久工作流引擎，优先映射其timer/cancel/idempotency语义，不建议为了loop整体迁移栈。

## 原始来源

- [官方PR #16835：合并时间、实现说明及未完成项](https://github.com/QwenLM/qwen-code/pull/16835)
- [对应nightly预发行及下载资产](https://github.com/QwenLM/qwen-code/releases/tag/v0.27.2-nightly.20261112.70a487be84)
- [设计（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/docs/design/2026-11-10-managed-loop-wakeup.md)
- [唤醒实现（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-loop-session.ts)
- [唤醒测试（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-loop-session.test.ts)
- [原子提交（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/core/src/managed-runtime/managed-session-authority.ts)
- [提交测试（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/core/src/managed-runtime/managed-session-authority.loop.test.ts)
- [调度器（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-monitor-wake.ts)
- [调度测试（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-monitor-wake.test.ts)
- [权限审查（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-harness-model.ts)
- [工具接入（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/cli/src/serve/hosted-workspace-tool-turn.ts)
- [运行依赖（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/package.json)
- [服务端依赖（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/packages/managed-agent-server/pom.xml)
- [许可证（固定合并版本）](https://github.com/QwenLM/qwen-code/blob/70a487be84b2a14daae21b9c2369b0ad243646dd/LICENSE)
