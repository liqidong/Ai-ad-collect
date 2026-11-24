---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-24
verified_at: "2026-11-24T00:04:59.620Z"
---

# Claude Ship：把 PR 审查、修复和批准分成三步

## 问题

代码审查经常在提出意见后把人叫回旧任务。新的托管工具尝试把修复和审批判断接起来，但修复权限与合并权限不应由同一环节默默获得。

## 输入

GitHub PR、连接的仓库、组织计划及仓库权限；启用审批时读默认分支的 `.claude/approvals.yml`。不是把本地任意代理加入调度的接口。

## 输出

分级审查意见、修复提交及回滚说明、审批检查状态或依规则的批准。最终是否合并仍受 GitHub 分支规则、审批和合并队列约束。

## 流程

- **审查与提证据**：已正式可用的 Code Review 给可定位的差异意见。新增的是与 Steward、Approvals 组合，而不是重发布代码审查本身。
- **单独处理修复**：新增 beta 的 PR Steward 可处理 CI、反馈或部分冲突并推提交；它不负责批准或合并。手动模式可作为小规模试验起点。
- **把批准与合并分开**：Approvals beta 根据仓库策略报缺少哪些批准或发批准。观察模式通常不阻塞且不发批准；要阻塞需成为 GitHub 必需检查。若开自动合并且策略没有人审要求，可能无人参与即合并。
- **不要把 /ship 当预览按钮**：个人入口可以推送分支并创建PR，落到什么状态取决于权限与分支规则；仓库级常驻配置则由组织管理。

## 关键机制

把代码发现、修复写权限、审批判断和GitHub合并门分层。配置新审批文件甚至在观察模式也会改变哪些PR自动审查；若旧规则未匹配 `claude/code_review`，可能停掉原来自动审查。

## 作者结果与可核对事实

- **新两角色11月23日开放 beta**：Anthropic 面向 GitHub 的组织版 Team/Enterprise 上线 Steward 和 Approvals，Code Review 已GA。/ship 开始从客户端逐步推出，未在账户上试用。 [来源](https://claude.com/resources/articles/introducing-claude-ship)
- **发布案例不是独立效果证据**：厂商分享内部产出和较低回滚比例，但其代码审查只是整套流程的一层；不能归因于新 beta、也不是单仓复现实验。 [来源](https://claude.com/resources/articles/introducing-claude-ship)
- **观察仍可能计费**：组织条线按 token；Approvals 无基本费，但分类器及策略触发的审查可收费。ZDR、HIPAA配置、客户管理密钥组织不支持这组托管代理。 [来源](https://code.claude.com/docs/en/claude-ship)

## 边界与未知

- 代码访问、发布分支、GitHub App权限都需要团队先审，不把可用性描述为默认已接通。
- 不保证发现一切缺陷；不能由 CI 绿色推断安全合并，不适合作为人工审查替代。
- 不同团队配置、触发逻辑与客户端滚动进度需要自己验证。本文未连任何用户仓库。

## 工程建议（未实测）

- 建议仅在无密钥、无生产部署权限的测试仓，从文档或测试PR开始：保留人工合并门、关闭自动合并、观察审批，按需要才人工召唤Steward。先测试误报、遗漏、重复修复、停止及回滚，再扩范围。
- 不要为了接入先把仓权限整组上交；评估报告同时记录每PR费用、人工等待和错误修复次数。

## 阅读范围

已核厂商发布全文、产品页、主管理文档与 GitHub连接权限说明；未登录、安装、授权或试用，也不把厂商内部反馈算独立对照。

## 原始来源

- [11月23日正式发布说明](https://claude.com/resources/articles/introducing-claude-ship)
- [Claude Ship设置与边界](https://code.claude.com/docs/en/claude-ship)
- [主管理流程与模式](https://claude.com/product/claude-ship)
- [Code Review连接权限](https://code.claude.com/docs/en/code-review-setup)
