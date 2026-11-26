---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-26
verified_at: 2026-11-26T00:03:11Z
---

# Magpie 0.1.4777：核对 Claude Explore/Plan 实际路由再做模型评测

## 问题
做多模型代理评测时，设置页写着某辅助模型不等于每次子代理都请求它。Claude Code 的内置 Explore/Plan 在部分版本有 inherit 标记；如果不看网关记录，就可能把模型路由变化误认为模型能力变化。

## 输入和输出
- 输入：使用 Magpie 代理的 Claude Code 请求、已配置的子代理目标模型和可选 reasoning effort、网关能识别的客户端分类、子代理请求头和已知系统提示标记；小实验另需固定的公开源码和标准答案。
- 输出：符合条件时对 Explore/Plan 工具轮重选请求模型，并按实际路径记录路由；不符合则保留原始请求。路由记录是本地网关观测，不是上游确用某权重的证据。

## 流程
- **区分主线程和子代理请求**：读已识别的客户端分类、子代理标识及已知提示标记。只对 agent=claude 的可识别 Explore/Plan 工具轮处理，不把单次调用的行为扩展到整个会话。
- **确认可选模型与限制**：取该代理的配置目标，比较它和已请求或已解析目标；未设置、同一目标、不可服务、被禁用或网关权限不允许时不强行改写。可从已配置的目标选项解析 effort。
- **先看路由，再评结果**：先用只读已知答案的公开源码题，确有辅助代理请求才计分；矩阵区分原请求、选中的尝试模型/effort 与最终提供方/模型和 fallback，再比较精确路径、行号或可重复测试。

## 机制
网关侧的窄条件改写，不是改 Claude Code 本身的默认继承关系，也不是给代理加一个通用评分器。模拟捕获能证明特定客户端发了什么请求及代理如何解释，不能验证外部中转隐藏的模型。

## 结果与证据
- **维护者在 Claude Code 2.1.327 使用模拟上游捕获 4 类请求。** Explore/Plan 请求主模型；general-purpose 请求配置的子模型；带自己 model 的自定义 agent 请求自定义值。是模拟上游收包，不是行为质量对照。 [原始证据](https://github.com/yetone/magpie/commit/99027d2d560caeb5635f211b884729fc7c402fca)
- **标签代码与测试限定改写边界。** 标记不匹配、代理类型不对、模型不可用/禁用或授权不允许均不应改写；没有提供生产任务准确率、时延或成本 benchmark。 [原始证据](https://github.com/yetone/magpie/blob/v0.1.4777/internal/gateway/claude_subagent_test.go)

## 局限与未知
- 不是检测模型是否降级或核验供应商隐藏权重的能力。只读网关记录能先排除接错模型、effort 或 fallback 等混杂。
- 不要对被分类为 claude-cli 的请求或其他所有代理类型做同样保证；以后客户端标记变化也需要回归。
- 0.1.4777 是被核验的固定版本，不断言最新版或当前部署已升级；无独立运行。

## 工程建议
- 固定公开仓库 commit、提示、工具权限和路由设置；选一个答案已知的代码定位题，记录过关率、命中路径/行号及成本，不用单次主观手感定性。
- 明确主线程行与辅助线程行；没有产生目标辅助请求的试次记为未触发，不混入质量分母。部署权限不因诊断放宽。

## 研读范围
读官方版本（UTC 11 月 24 日 22:32、上海 11 月 25 日）、变更说明、该标签的匹配与选路规则和回归测试、许可。模拟抓取为作者材料，未独立接入或安装。

## 原始来源
- [官方 0.1.4777 发行](https://github.com/yetone/magpie/releases/tag/v0.1.4777)
- [原始变更与请求捕获](https://github.com/yetone/magpie/commit/99027d2d560caeb5635f211b884729fc7c402fca)
- [版本固定的路由实现](https://github.com/yetone/magpie/blob/v0.1.4777/internal/gateway/claude_subagent.go)
- [版本固定的回归测试](https://github.com/yetone/magpie/blob/v0.1.4777/internal/gateway/claude_subagent_test.go)
- [版本固定的 MIT 许可](https://github.com/yetone/magpie/blob/v0.1.4777/LICENSE)
