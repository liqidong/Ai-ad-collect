---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-10
---

# Claude Dynamic Workflows：把多智能体调度交给后台程序

来源日期：2026-10-09
核查日期：2026-10-10（Asia/Shanghai）
状态：公开来源研读草稿；未独立运行或复现实验。

## 核心问题
数百份文档审查、批量迁移和交叉核验，容易把主代理的上下文与调度工作挤满。Anthropic 10月9日新增的 dynamic workflows 让代理生成编排程序，再由服务器后台执行。它是 Managed Agents 的使用机制更新，目前为 beta；不是新模型或准确率提升公告。

## 输入
启用 managed-agents-2026-04-01 beta header 的 Managed Agents 会话。代理配置中使用 multiagent.type=multiagent_20261001，并启用 workflows。 用户任务、可访问的材料，以及关于何时拆分、如何核验和失败时如何处理的指令。可提供 predefined_agents，也可允许工作流定义 inline agents。

## 输出
后台 workflow run、阶段进度与独立线程的执行记录，以及主代理汇总的任务产物。 运行状态与工作质量需要分开验收：API运行结束不等于每份材料都已成功处理。

## 逐步流程
### 1. 1. 配置可用的编排方式
新 multiagent 类型默认同时开启 workflows 与 subagents。若只需动态工作流，应显式关闭 subagents；开发者在系统提示中说明哪些任务值得展开。

### 2. 2. 主代理生成工作流
用户照常发送 user.message。代理决定是否启动，并生成负责调用代理、传递结果和合并输出的程序；客户端无需新增一个启动工作流的API调用。

### 3. 3. 后台分阶段运行
程序可并行分发任务、依据结果选择下一步或重复核验。主线程因此能继续交流，而不必亲自接手每个中间结果。

### 4. 4. 客户端处理线程事件
主事件流提供跨线程活动的摘要与工具请求。用 session_thread_id 区分来源，需要完整细节时读取对应线程；工具结果到达即可回复，无须等待整个会话idle。

### 5. 5. 双重判断是否结束
先确认所有已创建的run都有 workflow_run.status_ended，再等随后自然产生的 session.status_idle/end_turn。之后核对任务覆盖率与产物，避免把预算暂停或局部失败报成完成。

## 关键机制
关键变化是控制流由程序承接：模型生成编排，程序推进各阶段，多个独立上下文负责具体工作。它支持数据驱动的分支与复核循环，适合可拆分任务；持续需要对同一专家追问的任务仍可使用持久subagent线程。

## 来源报告的结果与证据
### 这是10月9日可验证的新能力。
Claude官方平台更新日志当天新增dynamic workflows条目，并给出beta header、配置方式与workflow_run.*事件入口。
来源：https://platform.claude.com/docs/en/release-notes/overview

### 完成标志不能直接充当质量验收。
Workflow runs文档明确说明completed只表示程序结束；即使部分线程失败或未能创建，run也可能completed。
来源：https://platform.claude.com/docs/en/managed-agents/workflow-runs

### 预算是全会话共享的成本控制。
Session budgets按公开标价累计消耗；达到上限后暂停新模型请求，已在途请求仍会完成。不能把它当作绝不超出一分的结算金额承诺。
来源：https://platform.claude.com/docs/en/managed-agents/budgets

## 局限与未知
- 所读官方页面没有给出可对比的成功率或端到端加速基准；并行是否更快、更便宜，需要在自己的任务上验证。
- run默认寿命24小时，暂停时间也计算在内；user.interrupt不保证终止后台run。
- 预算只能在创建session时添加；已移除的预算不能重新加回。amount使用美分整数字符串，例如125代表1.25美元，不是125美元。
- 开启更多代理会增加令牌消耗。上下文隔离也不能自动保证最终报告完整或各项结论正确。

## 工程建议（非实测结论）
- 建议试验：先挑30至50份非敏感、已有人工答案的文档，对比单代理串行、普通subagents和dynamic workflows；固定模型、工具、输入与验收标准。此规模与设计是工程建议，不是官方推荐参数。
- 建议每个子任务返回文档ID、覆盖状态、证据位置、失败原因。汇总器先对账输入清单，再输出结论；指标同时记录覆盖率、错误率、墙钟时间和实际成本。
- 建议加入局部工具失败、预算耗尽和连接重建测试；结果写入采用幂等键，避免重试造成重复副作用。
- 工具审批属于独立链路：主流上的请求由客户端按具体调用处理，不能因为工作流已经获准启动就一律放行。
- 初次接入保留小额session预算并监控session级list_cost；若需提高上限，修改已有预算，勿先移除。

## 实际阅读范围
已打开官方10月9日更新日志，并精读Multiagent orchestration的动态工作流、配置与代理类型章节；Workflow runs的机制、状态、完成判定、中断、预算和限制章节；Session threads的事件与工具权限章节；Session budgets的创建、变更和移除规则。未调用API、未运行实测。机制与限制来自官方文档，实验方案和验收清单为本文工程建议。

## 原始来源
- [Claude Platform release notes：2026-10-09](https://platform.claude.com/docs/en/release-notes/overview)
- [Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)
- [Workflow runs](https://platform.claude.com/docs/en/managed-agents/workflow-runs)
- [Session threads](https://platform.claude.com/docs/en/managed-agents/session-threads)
- [Session budgets](https://platform.claude.com/docs/en/managed-agents/budgets)
