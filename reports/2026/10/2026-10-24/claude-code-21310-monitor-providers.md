---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-24
---

# Claude Code v2.1.310：把事件监视扩到第三方云，先验收停止与权限边界

核查时间：2026-10-24T00:08:34Z。公开研究草稿；不是独立复现实验或采用结论。

## 核心问题与现有瓶颈

等待 CI 或日志时，让主代理不断发起模型回合查询，容易把无变化状态反复放进上下文，也会阻塞当前交互。Monitor 原本就能把后台事件送回会话；此次增量是 v2.1.310 放开此前受 API provider 和遥测开关限制的场景。与 10/23 已收录的 v2.1.309 沙箱启动文件修复不是同一变化。

## 输入

运行 v2.1.310 的 Claude Code 会话、已经合法配置且有权限使用的模型提供商、一个已审查的只读事件源，以及明确的关注事件和停止条件。临时 Monitor 可观察命令输出；插件方案需要监视命令、名称、说明和启动条件。最小验证只用人工生成的本地日志，不需要生产凭据或私有数据。

## 输出

被选择的日志行或状态变化作为通知进入当前会话，Claude 据此汇总；监视超时或取消属于生命周期结果，不等于业务成功。本文没有运行模型，也没有产出性能、费用或可靠性实测数字。

## 分步流程

### 1. 先验证版本与实际工具集

官方 Release API 标记 v2.1.310 为非预发布，发布于 2026-10-23T23:39:00Z。已有受支持账号在测试环境确认版本并询问当前会话是否加载 Monitor；不得仅凭网页有介绍就假设本机已经更新。保持原有 telemetry/nonessential traffic 设置，不为拿到工具而放宽隐私策略。

### 2. 把等待改成有界事件观察

先用临时 Monitor 监视本地合成日志，只让筛选后的关键行进入会话，主会话继续处理其他工作。官方说明临时监视有期限：默认 5 分钟、上限 30 分钟，单次 -p 提示的非交互运行上限 10 分钟。此次小测主动要求 2 分钟，避免把超时重启变成无期限承诺。

### 3. 需要复用时才改为插件配置

插件默认在 monitors/monitors.json 放数组，也可用 experimental.monitors 指向文件或内联数组。name、command、description 必填；when 默认 always，也可写 on-skill-invoke:<skill> 延迟到指定技能首次运行。先用 claude plugin validate 检查结构和路径，校验通过不等于脚本安全。

### 4. 区别临时工具与插件的权限模型

Monitor 工具调用的命令沿用 Bash 权限规则；插件自己启动的监视命令则在沙箱之外以用户权限运行。后者不能因主代理启用了 sandbox 就当作已隔离。只审查并运行自有、无敏感输出的脚本，把外部事件当作数据，不当作授予执行权限的新指令。

### 5. 以停止和丢失情况验收，而非只看一次通知

工程建议：发出带序号的普通行、目标事件、重复事件，再验证无匹配时安静、匹配时通知、主动取消后无新事件。插件方案还要测会话结束后的进程清理；官方文档说明中途禁用插件不会停止已启动的 monitor。记录实际现象，任何不符先保持测试状态，不直接接入生产告警。

## 关键机制

核心是把“持续观察外界”和“语言模型解释事件”拆开：宿主运行观察命令并把输出送入会话，而不是要求主代理每次等待都生成一个新查询回合。脚本可以采用 tail、文件事件或轮询，所以事件通知不等于取消所有轮询。v2.1.310 改的是此机制在第三方云及关闭遥测场景的准入范围，并同步禁止这些场景的长前台 sleep；发布说明未给 sleep 的具体阈值，本文不自行补出数值。插件自动启动与临时工具调用共享事件通知用途，但生命周期和权限不能混为一谈。

## 来源事实与报告结果

- 新增范围有可追溯的正式版证据，而不是把旧 Monitor 机制当作今天首发。
  - 证据：v2.1.310 release 明确列出 Bedrock、Vertex、Foundry 和关闭 telemetry/nonessential traffic 的情形；Release API 的 prerelease=false、published_at=2026-10-23T23:39:00Z。
  - [原始来源](https://github.com/anthropics/claude-code/releases/tag/v2.1.310)
- 监视输出可以进入仍在进行的会话，但有期限和明确取消行为。
  - 证据：Tools reference 的 Monitor 章节说明后台命令输出逐行送达、期限届满会结束监视，取消子代理会连带停止它启动的 monitors。这里只确认文档机制，不推断通知延迟或无丢失保证。
  - [原始来源](https://code.claude.com/docs/en/tools-reference#monitor-tool)
- 插件自动监视不是普通沙箱内工具调用。
  - 证据：Plugin security 明确区分模型工具调用与插件自行运行的代码；后者包括 monitors，运行在沙箱之外。Plugin components 还说明交互会话限制与禁用后已有进程的保留行为。
  - [原始来源](https://code.claude.com/docs/en/plugins/security)
- 当前证据不支持量化降本或可靠性提升。
  - 证据：已读完整 release 条目与相关官方功能章节；公开版本提交只更新 CHANGELOG.md 和 feed.xml，未提供此次 Monitor 平台门控实现或对应性能测试。
  - [原始来源](https://github.com/anthropics/claude-code/commit/b70dc4113ad72e0ee44f7b1b95afaf510b53ee98)

## 限制与未知

- 文档同步差异：核查时 Tools reference、Plugin manifest 仍写 Bedrock/Google Cloud/Foundry 不可用，且 Tools reference 仍写 telemetry 开关会禁用。新开放范围只按 v2.1.310 release 陈述；上线前须在目标账号、版本和 provider 实测。
- 不等于开放新地区或免除账号前提：仍需所在服务支持的地域、模型访问资格和组织许可。本次未登录任何服务或验证真实账号权限。
- 插件 monitors 仍放在 experimental 下，配置形态可能变化；按现行文档仅交互会话自动启动，不能外推为 -p / Agent SDK 中自动运行插件监视器。
- 插件命令可能读取文件或访问网络，输出可能被送入模型上下文。关闭遥测不等于日志不会进入模型请求；不要用含秘密或个人敏感信息的原始日志做试验。
- 没有公开运行时实现差分或独立复现证据；不得把长前台 sleep 的限制扩写成所有 sleep 禁用，也不得声称此次修复自动解决重复事件、断线重连或进程泄漏。

## 工程建议（未复现实验）

- 最小试用建议（未执行）：测试目录新建空日志，由另一终端依次写入带 ID 的 INFO 与 ERROR 行；要求 Claude 只观察 ERROR、持续 2 分钟、只报告不修改文件。先验收 1 条目标事件，再测试重复行、静默期、超时和取消。
- 建议验收指标：预期事件与收到事件的对应关系、重复通知数、从写入到通知的时差、取消后的残留进程、额外模型回合与实际 token 账单。没有对照数据前不写“节省多少成本”。
- 需要长时观察时先写业务停止条件，例如指定 CI run 达到终态；不要无限重启超时 monitor，也不要因为监视器结束就向用户报告构建成功。
- 插件投产前审查 command 与被调用脚本、固定来源版本并验证退出路径；保留最小文件与网络权限的外部隔离，不能把 Claude Code 的工具沙箱误认为插件进程隔离。
- 这条只建议已受影响部署做小范围兼容性回归。已有直连部署且工作流稳定者，无需因新开放范围全面重构等待逻辑。

## 实际阅读范围

已对照 2026-10-09 至 2026-10-23 日报条目与 10/23 coverage 去重。完整读取 v2.1.310 release notes，读取 GitHub Release API 的版本/时间/正式版字段及公开提交 b70dc4113ad72e0ee44f7b1b95afaf510b53ee98 的文件差分；细读 Tools reference 的 Monitor 与工具可用性章节、Plugin components 的 Monitors 章节、Plugin manifest 的 monitors/路径规则、Plugin security 的权限与信任说明。未阅读或获取未公开的 Claude Code 运行时实现，未安装软件、未调用收费模型、未实测任何 provider。

## 原始来源

- [Claude Code v2.1.310：正式版发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.310)
- [GitHub Release API：精确发布时间与非预发布标记](https://api.github.com/repos/anthropics/claude-code/releases/tags/v2.1.310)
- [Claude Code Tools reference：Monitor 工具、期限与权限](https://code.claude.com/docs/en/tools-reference#monitor-tool)
- [Claude Code Plugin components：Monitors 运行生命周期](https://code.claude.com/docs/en/plugins/components#monitors)
- [Claude Code Plugin manifest：monitors 字段与触发方式](https://code.claude.com/docs/en/plugins-reference#monitors)
- [Claude Code Plugin security：插件自身代码不受工具沙箱保护](https://code.claude.com/docs/en/plugins/security)
- [v2.1.310 公开仓库提交：更新 CHANGELOG.md 与 feed.xml](https://github.com/anthropics/claude-code/commit/b70dc4113ad72e0ee44f7b1b95afaf510b53ee98)
