---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-27
verified_at: 2026-11-27T00:06:39Z
---

# Claude Code：MCP 可以发布技能，加载时的内联命令可单独拦截

## 问题
团队希望集中分发同一套研究步骤，不想在每个项目复制技能目录；但能动态更新的技能不应未经检查就运行加载阶段的本地命令。发布和执行是两个边界。

## 输入和输出
- 输入：合适版本和账户的本地交互 Claude Code；受信任且实现 Skills 扩展的 MCP 服务；可选的本地/托管 hook。服务交付技能，应用本地权限和加载规则。
- 输出：在具备资格的会话中出现带服务命名空间的技能；客户端可按规则提示、阻止或加载。对适用的内联命令，hook 可拦截命令或阻断加载流程。不是 Codex 的执行接口，也不自动变为远程调度。

## 流程
- **把共享技能归到服务器命名空间**：服务器实现列表和交付技能的协议扩展；受支持客户端可在斜杠菜单看到 server:skill。普通同名本地技能保留自己的命名。
- **先核查客户端和模型适用范围**：官方文档把 MCP 技能最低版本写成 2.1.331，2.1.332 的发布说明明确列出此 beta 和新 hook。某些订阅或组织策略才启用；还要求符合条件的模型与直连 Anthropic；自定义网关、claude -p、Agent SDK 与云会话不在文档支持范围。
- **分离加载与执行期控制**：限制哪些服务器/技能可被加载；内联命令通过现有权限判断后，适用时再经过 PreSkillShellCommand。若加载时未执行而之后变成常规 tool call，需用 PreToolUse。两层 hook 不能互相替代。
- **用小样岛验证触发点**：用无凭据的临时项目和可信本地测试服务器，先发布纯文档技能，确认发现与显式拒绝；再用本地技能的无副作用打印命令测试新事件与普通工具调用各自的否决路径。不要从远端下载陌生技能来测。

## 机制
MCP 技能是集中分发，客户端不执行它的内联 shell，也忽略其中的 allowed-tools、hooks 等若干前置元数据。新 hook 则管支持在展开时执行内联 shell 的技能或命令；不把它误当成给远程 MCP 脚本增加第二重审批。MCP 内容仍可能引导后续工具调用，需独立权限与 PreToolUse。

## 结果与证据
- **v2.1.332 发布说明记录两项新增。** 作者发行页记录按服务器命名空间显示的 beta、可拒绝加载时内联命令的事件；没有任务成功率、性能或安全基准。 [原始证据](https://github.com/anthropics/claude-code/releases/tag/v2.1.332)
- **官方 hook 给出本地可检查的输出。** 文档使用无副作用输出命令演示在 shell 前被拦下。普通错误或超时不一定阻断；官方建议对要求必须阻止的场景核对 matcher、类型、位置与失败处置。 [原始证据](https://code.claude.com/docs/en/hooks#preskillshellcommand)

## 局限与未知
- 老版本可能忽略新 hook 并弹出告警，不应下发配置前先假定已受保护。 凡要求失败即阻断的 hook，另核对 onFailure:block。
- 服务器可以更新技能，第三方服务器不会普遍经过官方扫描；基本 MCP 资源读取可能仍另受独立规则允许。
- 没有实际安装或接入测试；账号、组织或支持方式决定了 beta 可用性。

## 工程建议
- 如果现有执行链必须走自定义网关或 headless CLI，暂不把该 beta 设计为关键依赖。可用固定源码版本/local skill 作为可审阅对照。
- 验收分开记录：技能是否被发现、加载是否需提示、普通工具调用是否符合规则、缺失/超时的 hook 是否确实按预期失败。保持人审和沙箱约束。

## 研读范围
核对官方 v2.1.332 发行、MCP 支持范围/版本/审批政策及 hook 事件、输入与决策；未实际接入，不把公开 beta 表述为全平台可用。

## 原始来源
- [Claude Code v2.1.332 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.332)
- [Skills over MCP 官方支持与审批](https://code.claude.com/docs/en/mcp#use-skills-from-mcp-servers)
- [PreSkillShellCommand 官方事件参考](https://code.claude.com/docs/en/hooks#preskillshellcommand)
- [技能来源和 MCP 限制](https://code.claude.com/docs/en/skills#skills-from-mcp-servers)
