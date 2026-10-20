---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-20
---

# Claude Code 2.1.304：把必须确认的工具调用交给人审入口

核查时间：2026-10-20T00:04:17.027036Z（UTC；北京时间加8小时）

## 核心问题

无人值守 agent 中，某些操作必须保留明确的审批入口；仅写一段提示词不足以形成可验证的执行边界。

## 输入与训练前提

Claude Code 2.1.304、已获组织允许的 mod，以及 tool.check 收到的工具参数和现有权限决定。

## 输出

保留原决定、拒绝，或要求由权限对话框／宿主回调审批；没有审批入口时拒绝。

## 关键机制

这是工具权限决策层的控制点，不改变基础模型推理。新增 decider 字段确定谁能批准 ask；mod 中间件决定何时提出 ask。两者需分别测试：规则是否命中，以及宿主是否确实执行人审。

## 逐步流程

1. **先核对运行条件**：用 claude --version 确认版本；正式会话需要受支持的付费／Console账户或第三方API提供方。组织策略可禁用mod，Desktop的WSL会话不支持插件。

2. **在权限判断处收紧**：tool.check 在工具调用与 PreToolUse 等判断之后处理 allow／ask／deny。对选定操作返回 ask 并指定 decider: "person"；不要把所有非只读操作机械视为同等风险。

3. **让审批缺席变成拒绝**：本次新增字段把批准入口限定为权限对话框或宿主回调；无可询问对象时，以mod给出的理由拒绝。宿主回调若自动放行，就不能称为真正的人类审核。

4. **最小试用与验收**：先在隔离样例目录按官方mod模板实现单一测试工具规则，运行 claude plugin test 检查返回对象；再分别验证交互批准、交互拒绝、无回调headless三种路径。只用可丢弃的测试文件，不拿真实删除或生产发布做首轮测试。

## 来源事实与效果证据

- **新增指定审批者与只读标记。**：v2.1.304发布说明列出 decider: "person" 与 e.isReadOnly；这是官方行为说明，没有给出安全评测数字。 [原始证据](https://github.com/anthropics/claude-code/releases/tag/v2.1.304)

- **mod并非天然失败关闭。**：事件文档说明，未捕获的异常／超时可能跳过hook；可加.catch返回拒绝，但错误处理器自身失败仍有跳过风险。 [原始证据](https://code.claude.com/docs/en/plugins/mods/events)

- **可以先做无模型本地测试。**：官方测试工具支持伪造事件与stub，无需会话、登录或联网；它不执行真实模型／工具，因此不能代替完整权限路径验收。 [原始证据](https://code.claude.com/docs/en/plugins/mods/test)

## 限制与未知

- 本轮只阅读官方发布说明与文档，未安装、未运行、未实测。
- 在线reference正文仍标注CLI/Desktop截至2.1.290，未列新decider字段；完整返回类型以已安装2.1.304生成的声明为准，不提供臆造的完整代码。
- 人审入口不等于安全沙箱；mod优先级、异常处理与宿主回调仍可能影响最终行为。
- 文档未给出误放行率、额外延迟或跨宿主一致性测试数据。

## 工程建议（未经本次复现）

- 最小投入：一个无害工具规则、三个审批路径和一个异常路径；记录实际调用是否发生及拒绝原因。
- 保留系统原有deny；测试时验证规则叠加结果，避免用allow覆盖其他权限约束。
- 正式上线仍需服务端权限、最小凭据与资源级访问控制，不能只依赖字符串匹配hook。
- 升级渠道有差异：stable通常滞后一周；先查版本，不因更新命令返回成功就假定已到2.1.304。

## 阅读范围

已打开v2.1.304完整发布页，阅读mod事件、reference、开放范围、测试和安装文档相关章节；核对最新发布列表。未读二进制实现，未复现实验。

## 原始来源

- [Claude Code v2.1.304 发布说明（2026-10-19）](https://github.com/anthropics/claude-code/releases/tag/v2.1.304)
- [Mods：事件处理与失败语义](https://code.claude.com/docs/en/plugins/mods/events)
- [Mods reference：接口与版本边界](https://code.claude.com/docs/en/plugins/mods/reference)
- [Mods 开放范围](https://code.claude.com/docs/en/plugins/mods/overview)
- [Mods 本地测试](https://code.claude.com/docs/en/plugins/mods/test)
- [Claude Code 安装、账户与升级](https://code.claude.com/docs/en/setup)
