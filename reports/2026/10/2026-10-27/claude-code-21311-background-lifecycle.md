---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-27
---

# Claude Code 2.1.311：长任务不能只看对话是否结束

核查时间：2026-10-27T00:08:00Z

## 核心问题

长时工具执行中，模型回合、后台进程、工作目录和会话恢复是不同生命周期。只把对话没有新文字当成任务结束，或把重连当成新任务，容易造成重复执行和状态丢失。本次关注 10 月 26 日正式版的相关修复，不把已有后台 Agent 功能重新包装为新发明。

## 输入

运行时输入是已有任务、会话历史、MCP 工具和宿主机进程状态，不涉及模型训练数据。建议验证使用可丢弃仓库及可控的本地延时任务，避免生产任务或真实支付、发布等副作用。

## 输出

产品输出仍是会话事件、工具结果及文件产物。工程验收应另外记录进程是否退出、产物是否完整、结束回调次数和恢复后是否重做；显示 done 或工具返回文本本身不是业务验收。

## 分步流程

### 1. 先确定产品边界

官方把本机后台会话与企业 self-hosted runner 分开：前者由本机 supervisor 管理，后者承接云会话到组织自有机器。不要把 runner 新参数当成本机所有会话自动获得的能力。

### 2. 读取状态而不是连续催问

已有 CLI 提供 claude agents --json、claude logs <id> 和 claude daemon status。状态列表可区分 working、blocked、done、failed、stopped。使用这些接口获得证据，再决定等待、提供输入或恢复；这些命令不是本次新发布。

### 3. 在测试任务里制造等待

建议让一个无外部副作用的脚本延时写入完成标记，并记录开始与结束时点；另让 MCP 服务延迟建立连接。观察后台等待时是否产生无意义主回合，以及晚连接工具能否被子代理使用。这是建议的回归实验，本轮未执行。

### 4. 核查退出与恢复

在测试副本中记录子任务使用的 worktree 和结束钩子。模拟恢复后检查目录仍存在、已完成工作没有再次执行。不要直接杀生产训练进程来验证；正常停止、进程崩溃和删除会话具有不同后果。

### 5. 分开交付与基础设施保活

新版发布说明新增 runner 的 --hold-idle-file 与 --max-hold-min，可由宿主报告仍运行进程来保留空闲会话。本次检索到的参数参考尚未出现两者，因而不编造文件格式、默认值或保活上限语义；实际部署前以对应版本 help 核验。

## 关键机制

官方发布说明报告修复了后台 auto 会话等待自身 shell 时反复唤起主回合、SubagentStop 重复触发、后台子代理仍工作时 worktree 被移除、晚连接 MCP 工具不可用，以及无视 API 禁止重试信号的问题。其共同工程含义是把事件等待、执行存活、完成通知和重试边界分开；这不是新增模型推理算法。源码内部实现未核验，不能将上述解释当作实现级证明。

## 来源事实与效果

- 已发布的行为修复，不是性能基准：v2.1.311 发布页时间为 2026-10-26 21:51 UTC。官方列出上述后台生命周期及重试修复；未提供任务成功率、节省 token、吞吐提升或独立复现数据。 [来源](https://github.com/anthropics/claude-code/releases/tag/v2.1.311)
- 接口存在不等于本次新增：CLI 与 agent-view 文档提供状态查询和日志命令，用于设计验证流程。文档同时区分会话状态和 supervisor 进程；不据此推断每个后台工具都能跨重启存活。 [来源](https://code.claude.com/docs/en/agent-view)
- 自托管有独立准入条件：官方说明 self-hosted environments 为 Team/Enterprise 公测且默认关闭，模型推理和会话内容仍涉及服务端，不等于全离线部署。 [来源](https://code.claude.com/docs/en/self-hosted-environments)

## 局限与未知

- 本轮读了完整发布说明，以及 CLI、agent-view、self-hosted 概览和参数参考中的相关章节；未安装、调用模型或复现实验。
- 新 runner 两参数只有发布说明证据，已读参数页未同步；配置格式、轮询频率和与其他寿命限制的交互仍待对应版本核验。
- 修复不能证明第三方 MCP、自定义守护进程或所有 API 网关都会遵循同样恢复语义。
- 现有 self-hosted 文档说明会话内容仍发往 Anthropic；自有执行机器不等于数据不出网。

## 工程建议，非实测

- 最小试用：先查版本和只读状态；在隔离测试目录分别验证等待、MCP 晚连接、回调只触发一次、恢复后不重复执行四个场景。先记录基线，再在确认升级后重测。
- 把每次业务执行绑定稳定任务 ID，并记录产物哈希或完成清单。对写数据库、发消息等副作用做应用层幂等，不能把升级客户端当作替代。
- 等待策略应随可观察状态调整：有进度则等待，缺输入则明确阻塞，终态则读取产物。避免“无新文字就重启”或不断空轮询。
- 故障定位优先保存日志和任务状态；终止、删除、修改权限与新建持久凭据都应单独审批。本日报没有执行这些动作。

## 阅读范围

完整阅读 2.1.311 官方 release notes；阅读 CLI reference 的状态与恢复命令、agent-view 的状态表与 supervisor 机制、self-hosted 概览及参数参考相关章节。没有读取闭源运行时实现；没有实机测试。

## 原始来源

- [Claude Code v2.1.311 正式发布](https://github.com/anthropics/claude-code/releases/tag/v2.1.311)
- [CLI reference：状态、日志与恢复入口](https://code.claude.com/docs/en/cli-reference)
- [Agent view：后台会话状态与进程生命周期](https://code.claude.com/docs/en/agent-view)
- [Self-hosted environments：适用范围](https://code.claude.com/docs/en/self-hosted-environments)
- [Self-hosted runner reference：参数与生命周期边界](https://code.claude.com/docs/en/self-hosted-environments-reference)
