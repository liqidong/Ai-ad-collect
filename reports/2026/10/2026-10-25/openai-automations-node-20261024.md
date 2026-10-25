---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-25
---

# OpenAI 调度接口进入 Node 主分支：投递完成不等于任务完成

核查时间：2026-10-25T00:05:47Z。公开研究草稿；不是独立复现实验或采用结论。

## 核心问题与现有瓶颈

周期研究、日报和利用历史上下文的复查，需要可靠地保存日程、选择目标会话，并区分“已到点投递”与“实际完成工作”。本次增量不是模型质量提升，而是将这些调度对象和投递记录暴露为托管 API 与 agent 工具。

本期收录的是 **Node 客户端主分支新增接口**，不是“10月25日发布新模型”或“正式 SDK 已可升级”。[官方 Node 提交 d8931e7](https://github.com/openai/openai-node/commit/d8931e7facfd0b453cc1683c31c60be8cc1ae37e)时间为 2026-10-24 02:39:23 UTC，晚于上期核查截止。它新增调度资源、invocation 资源及 `automations` 工具类型。对应 [Python 提交 43045fc](https://github.com/openai/openai-python/commit/43045fcaf91aec08571e54c57706597cefd32d45)为10月23日21:23:35 UTC。

[社区公告](https://community.openai.com/t/introducing-scheduled-automations-to-the-agents-api/1409223)在10月23日21:24 UTC发布，作者为社区版主；只将其当日期旁证，不把版主身份当产品开放权限证明。能力与字段依据官方文档和固定提交。本轮核到正式发布仍是 [Node v7.43.0](https://github.com/openai/openai-node/releases/tag/v7.43.0)及 [Python v3.37.0](https://github.com/openai/openai-python/releases/tag/v3.37.0)，均为10月22日。因此 API 能力属于上期遗漏补收，今天的可证明增量是 Node 主分支接口落地。

## 输入

应用使用 OpenAI Platform 项目的 API key，并提供 OpenAI-Beta: agents=v1。周期日程输入 type=periodic、标准五字段 cron expression 和 IANA timezone；一次性日程输入 type=once 与未来 Unix 秒级 run_at，后者不配 expression/timezone。目标 new_session 需在 agent 与 agent_id 中二选一，并给出 environment 与 target.input；输入至少含一条非空文本或图像消息。目标 existing_session 提供 session_id 与 instructions。created_by 由服务设置，不由调用方填写。

用自己预先建立的无外部工具测试 session，把 SESSION_ID 换成真实测试 ID，把 run_at 改成实际试用时五分钟后的整数时间戳。下面为结构示例，不是已执行结果：

POST https://api.openai.com/v1/agents/automations

请求头：Authorization: Bearer YOUR_PROJECT_API_KEY；Content-Type: application/json；OpenAI-Beta: agents=v1

```json
{
  "type": "once",
  "name": "synthetic-schedule-check",
  "run_at": 1798794000,
  "target": {
    "type": "existing_session",
    "session_id": "SESSION_ID",
    "instructions": "仅输出合成测试标记 schedule-check-001，不调用工具，不发送外部消息。"
  }
}
```

上例 run_at 为2027年1月1日09:00 UTC，试用前改成近期未来时间。请求字段依据 [Create automation](https://developers.openai.com/api/reference/resources/beta/subresources/agents/subresources/automations/methods/create)。保存返回的 automation ID，读取其 invocations；获得 session_id 后检查对应回合和结果。不要为了试用先安装主分支包，更不能把文档中的 beta.agents.automations 示例当作当前正式包已具备。

建议分成五个独立用例，不接生产资源：
- 正常到点：核对计划时间、实际投递时间、主回合结束时间和文本标记。
- 忙会话：观察等待及最终处理，不将“没有立刻输出”直接认定丢任务。
- 手动与定时并存：手动 invocation 不消耗 once 的到点运行，防止误跑两次。
- 调度暂停：确认未来 scheduled delivery 与 manual invocation 的差异。
- 正在执行时停止：分别停止日程与取消主回合，检查是否仍有在途投递。

## 输出

创建返回 automation 资源及 ID，只保存生效日程，不立即运行。随后可查询 invocation 历史，其中 queued / completed / failed / skipped 是投递状态，并带目标 session 关联信息；历史不是所有尚未发生的未来日程列表。投递成功后仍须沿 session_id 检查主回合终态、保存的 items 和具体结果。对 existing_session，instructions 以通知而不是新用户消息方式进入当前会话，使用当前配置和已保存状态；忙会话须等待可投递时机。

1. 保存日程：创建成功不立即执行。
2. 投递记录：invocation 的 queued / completed / failed / skipped 描述投递；源码还有 `skip_reason: pending_delivery`，但这个枚举本身不能证明完整重叠调度策略。[invocations 固定源码](https://github.com/openai/openai-node/blob/d8931e7facfd0b453cc1683c31c60be8cc1ae37e/src/resources/beta/agents/automations/invocations.ts)
3. 执行结果：沿 session_id 看主回合 completed / failed / cancelled，再检查输出与保存的 items。idle、断流或子 agent 完成都不是整个任务成功；主回合完成也不保证所有工具都成功。[Events and items](https://developers.openai.com/api/docs/guides/agents-api/sessions/events)

应用仍负责向用户展示结果及发送通知。对忙会话，通知等待可投递，而不是打断正在运行的回合。[调度指南](https://developers.openai.com/api/docs/guides/agents-api/sessions/automations)

## 分步流程

### 1. 1. 确认版本与使用入口

核对 Node 固定提交 d8931e7 的 2026-10-24 02:39:23 UTC 日期，并确认本轮最新正式 Node 7.43.0 / Python 3.37.0 都仍为 10 月 22 日。因此本期记为 Node 主分支接口新增，API 能力为 10 月 23 日补收，不能让用户仅升级正式包便假定获得该接口。最小验证可按 HTTP beta 请求结构进行，本轮不调用。

### 2. 2. 先配置任务授权，再保存日程

先用无外部工具的独立测试 session，确定任务只输出合成标记。应用 API 可管理项目日程；若启用 agent automations 工具，agent 仅可管理当前会话中由 agent 自建的日程，不能管理其他会话或用该工具新建会话。调度许可不等于外部通信、支付或数据修改许可。创建 schedule 成功仅说明日程已保存。

### 3. 3. 到点投递或明确手动触发

periodic 等到下个匹配时点，once 等待未来 run_at。手动触发使用 POST /v1/agents/automations/{id}/invocations，无请求体，不改变日程、不消耗 once 的定时机会，暂停时也允许手动触发。测试手动后仍可能在原计划时间再次执行，应独立观察两种来源。忙会话不被打断，调度也不能补齐未完成的工具返回或审批。

### 4. 4. 分别验收投递、主回合和业务结果

查询 invocation 判断输入是否已被 session 接受，再观察 subagent_id 为 null 的主回合 completed / failed / cancelled 与保存结果。invocation completed 不是任务完成；idle、流中断或子 agent 结束都不足以证明成功。主回合 completed 也不保证每个工具调用成功。应用仍负责验证文本标记、展示结果以及按授权发送通知。

### 5. 5. 处理更新、暂停与停止

更新仅改变未来 invocation，已排队 invocation 保留原快照。未提供字段保持不变；target 更新为同类型目标的整体替换，agent 创建的日程仍属于创建会话。暂停拦住尚未投递的 scheduled delivery，但不影响 manual invocation；删除阻止未来手动与定时触发。暂停或删除不取消已接受的工作，且在途投递可能到达；停止活动回合另发 agent.session.input.cancel 并核实终态。

### 6. 6. 以隔离测试补齐服务语义未知

将正常到点、忙会话、手动与定时并存、暂停、执行中停止拆为五个用例，记录计划时刻、实际投递时刻、主回合结束与输出标记。调度 create / invocation 的 exactly-once、重试、时效、DST 与故障恢复不能从 SDK 类型或 mock 测试推导，应在采用前另行验收。

## 关键机制

以前自行使用 cron 调 session，需要自己保存日程、目标和投递记录；现在这些有了托管 API 资源。它不是10月9日 Claude 动态 workflow 的“分阶段运行许多 agent”，也不是 Decisions 的快速类型化判定，更不是本地 Monitor 的事件监听。它解决“何时把任务交给哪个会话”。

[固定客户端契约](https://github.com/openai/openai-node/blob/d8931e7facfd0b453cc1683c31c60be8cc1ae37e/src/resources/beta/agents/automations/automations.ts)区分：
- `periodic`：五字段 cron 加 IANA 时区。
- `once`：未来的 Unix 秒级 `run_at`。
- `new_session`：每次独立会话，提供 agent 或 agent_id、environment、target.input。
- `existing_session`：目标 session_id 加 instructions，适合利用先前调查结果。

这是客户端与服务契约的变化，不是已经测得的速度、成本或成功率提升。

仍走 `OpenAI-Beta: agents=v1`。需要自己的 OpenAI Platform 项目、可用模型及 API 权限；[quickstart](https://developers.openai.com/api/docs/guides/agents-api/quickstart)列出 session 所需 api.agents.read / write 与推理所需 api.responses.write。此处未实际验证账号资格、地区或网关支持，不推断跨云同样可用。密钥留在应用端，不放进 agent 沙箱。模型、工具和托管容器依照[既有计费](https://developers.openai.com/api/docs/guides/agents-api/overview#pricing)，不称调度后的工作免费。

[指南](https://developers.openai.com/api/docs/guides/agents-api/sessions/automations#let-the-agent-schedule-follow-ups)规定，agent 内置 automations 工具默认不存在；启用后只管理当前会话由 agent 自建的日程，不能替别的会话管理，也不能借此新建会话。应用侧 API 权限更广。移除工具不会清除已有日程。调度不是用户审批，也不能补齐缺失的工具返回或批准。

工程建议：日程配置与任务授权分别记录；先只读后考虑写工具，不因用户同意“每天检查”就允许每天发信、付款或修改数据。

日程、投递记录与会话执行是三个分离对象。once 日程自身的 completed 也不能解释为工作成功；invocation 的 completed 表示 session 已接受输入、通知或开始工作的请求，实际执行及业务验收继续由 session 和应用负责。

## 来源事实与报告结果

- 上期核查截止后有可证实的 Node 主分支实质接口新增，但不是正式 SDK 发版。
  - 证据：固定提交 d8931e7facfd0b453cc1683c31c60be8cc1ae37e 的时间为 2026-10-24T02:39:23Z，新增 automations 与 invocations 资源、agent automations 工具类型及生成测试；早于本轮的正式 Node v7.43.0 仍发布于 10 月 22 日。
  - [原始来源](https://github.com/openai/openai-node/commit/d8931e7facfd0b453cc1683c31c60be8cc1ae37e)
- API 能力按 10 月 23 日遗漏补收，而非 10 月 25 日新发布。
  - 证据：Python 对应官方提交为 2026-10-23T21:23:35Z；社区版主公告为 21:24:34Z，仅作时间旁证。能力和字段以官方源码、文档交叉核对，版主身份不被当成账户开放权限证明。
  - [原始来源](https://github.com/openai/openai-python/commit/43045fcaf91aec08571e54c57706597cefd32d45)
- 托管接口新增“日程 → 投递 → 会话”链条，而非替代任务执行验收。
  - 证据：指南明确创建日程不立即执行；manual invocation 不改变 schedule；invocation completed 仅说明成功向 session 交付，应用仍负责观察工作和呈现结果。
  - [原始来源](https://developers.openai.com/api/docs/guides/agents-api/sessions/automations)
- agent 工具具有会话和创建来源限制，但移除工具不会自动删除既有日程。
  - 证据：官方指南限制 agent 只能管理当前 session 中由 agent 创建的 automations，不能新建 session 或管理其他 session；工具默认不启用，移除后已有日程仍需由应用另行处置。
  - [原始来源](https://developers.openai.com/api/docs/guides/agents-api/sessions/automations#let-the-agent-schedule-follow-ups)
- 完成投递、完成主回合和业务成功必须分别记录。
  - 证据：invocation 使用 queued / completed / failed / skipped；事件文档另列主回合 completed / failed / cancelled，且主回合完成不等于每个工具均成功。断流或 idle 不能替代终态与输出检查。
  - [原始来源](https://developers.openai.com/api/docs/guides/agents-api/sessions/events)
- 暂停或删除调度不能代替取消已运行的任务。
  - 证据：指南说明已被 session 接受的工作不会因 pause/delete 被取消，在途投递可能继续；Sessions 文档提供独立 agent.session.input.cancel 事件，取消后仍应验证终态。
  - [原始来源](https://developers.openai.com/api/docs/guides/agents-api/sessions#cancel-an-active-turn)

## 限制与未知

- 本轮只确认官方文档与固定主分支提交，不把主分支 API 当作已发布的 npm / PyPI 包能力，也没有安装主分支包。
- 未调用实际服务，未实测账号资格、地区或网关支持，不能推断跨云可用性、开放范围或真实运行时延。
- 新增客户端测试主要是 mock API 的请求/响应形状检查，不等于验证托管服务的端到端调度。
- 没有找到调度 create / invocation 的 exactly-once 或完整重试保证；SDK 通用 idempotencyKey 选项不能证明服务端去重语义。
- skip_reason: pending_delivery 是源码枚举，不能单凭该字段推断完整重叠调度、跳过或重试策略。
- 服务时效、夏令时边界、限流、故障恢复和业务幂等均未验证，需作为采用前的明确验收项。
- 创建、投递或主回合 completed 都不是独立的业务正确性证明；仍须检查目标输出与工具结果。
- 调度不是新的用户批准，无法补齐挂起的工具结果或授权；忙会话等待可投递时机，不会被定时通知直接打断。
- 模型、工具与托管容器依既有计费，不将定时后发生的工作称为免费。

## 工程建议（未复现实验）

- 最小试用的完整 HTTP 结构保留在 inputs。仅用无外部工具的测试 session，合成标记 schedule-check-001，不发外部消息。示例 run_at 为 2027-01-01 09:00 UTC，实际试用前替换为约五分钟后的未来整数时间戳。本轮未执行该请求。
- 密钥留在应用端而不是 agent 沙箱。session 权限参考 api.agents.read / write，推理参考 api.responses.write；实际项目权限与可用性需独立核实。
- 业务层保存 automation ID、invocation ID、session ID、主回合 ID 与业务任务键，分别统计投递延迟、执行耗时与最终验收；这些是工程建议，不是平台保证。
- 手动测试不消耗 once 的 scheduled occurrence，暂停也允许 manual invocation。不要把手动成功后保留的定时机会误当成不会再次执行。
- 为外部副作用建立业务任务键与去重记录，超时先查历史，不盲目重复 POST。不要将通用 SDK 参数当成调度 exactly-once 保证。
- 更新未给出的字段保持不变，target 要作同类型的整体替换；排队 invocation 使用旧快照。配置修改后应分清旧队列和新计划。
- once 在暂停期间过期，恢复前需设置未来 run_at；已 completed 的 once 不能 pause/resume，但仍可手动触发或另建一次性日程。
- 停止任务时分两步：处理未来 schedule，再取消活动 session turn 并验证终态。移除 automations 工具不清除已存在的 schedules。
- 取消不是回滚，已发生的外部副作用需独立处理；已有用户授权仍须按操作、目的和受众边界执行。

## 实际阅读范围

完整核读相关官方 Scheduled automations 指南的调度、目标、工具权限、投递、更新和停止语义；Create automation 与 List invocations 契约；Events and items 的主回合终态与断流恢复；Sessions 的取消说明；Quickstart 的权限及密钥要求、Overview 计费。检查 Node 固定提交 d8931e7 的资源类型、主要请求/响应实现及新增测试，并用 Python 固定提交 43045fc 与官方 release 列表核对日期和发布状态。社区版主公告只用作日期旁证。未执行多语言示例、未运行 SDK 测试、未调用 API、未安装 SDK。

## 原始来源

- [OpenAI Node 官方提交：新增调度 API 与 agent 工具](https://github.com/openai/openai-node/commit/d8931e7facfd0b453cc1683c31c60be8cc1ae37e)
- [固定源码：Automations 请求与生命周期](https://github.com/openai/openai-node/blob/d8931e7facfd0b453cc1683c31c60be8cc1ae37e/src/resources/beta/agents/automations/automations.ts)
- [固定源码：invocation 投递状态与来源](https://github.com/openai/openai-node/blob/d8931e7facfd0b453cc1683c31c60be8cc1ae37e/src/resources/beta/agents/automations/invocations.ts)
- [官方 Scheduled automations 指南](https://developers.openai.com/api/docs/guides/agents-api/sessions/automations)
- [官方 Create automation API 契约](https://developers.openai.com/api/reference/resources/beta/subresources/agents/subresources/automations/methods/create)
- [官方 Events and items：完成与断流恢复](https://developers.openai.com/api/docs/guides/agents-api/sessions/events)
- [OpenAI Node 最新正式发布列表](https://github.com/openai/openai-node/releases)
- [OpenAI Python 官方提交：2026-10-23 调度接口](https://github.com/openai/openai-python/commit/43045fcaf91aec08571e54c57706597cefd32d45)
- [OpenAI 社区版主公告：仅作发布日期旁证](https://community.openai.com/t/introducing-scheduled-automations-to-the-agents-api/1409223)
- [Node v7.43.0 正式 release](https://github.com/openai/openai-node/releases/tag/v7.43.0)
- [Python v3.37.0 正式 release](https://github.com/openai/openai-python/releases/tag/v3.37.0)
- [Agents API quickstart：应用权限与密钥边界](https://developers.openai.com/api/docs/guides/agents-api/quickstart)
- [Agents API overview：计费](https://developers.openai.com/api/docs/guides/agents-api/overview#pricing)
- [Cancel an active turn：单独取消当前回合](https://developers.openai.com/api/docs/guides/agents-api/sessions#cancel-an-active-turn)
- [List automation invocations：投递记录字段](https://developers.openai.com/api/reference/resources/beta/subresources/agents/subresources/automations/subresources/invocations/methods/list)
