---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-30
---

# Claude 统一用量成本 Analytics：完整归因之前，先守住分页、数据水位与金额单位

来源日期：2026-10-29（官方发布说明仅提供日期）

核查时间：2026-10-30T00:07:26Z

状态：2026-10-29官方beta开放公告；不是SDK当日新增

## 核心问题

代理评测常把调用日志里的token估算、组织成本报表和最终账单混在一起。若迁移时仍按旧桶结构读取，或把limit当总行数，低频身份与长尾模型就会消失；把美分字符串当美元还会放大100倍。此次新增价值是统一组织级查询与更细归因契约，不是新模型推理能力。

## 训练与推理输入

获授权的组织分析凭证、明确UTC起止区间、组织适用的筛选维度、group_by和时间桶；建议先选择model与actor等少量维度。Console侧工作区、服务档位等筛选不能直接套给Enterprise。本文只研究公开文档与源码，未读取任何组织数据。

## 输出与资产形式

带起止时间、group维度对象及usage或cost指标的行；外层还有data_refreshed_at、has_more、next_page。用量包含未缓存输入、缓存创建/读取、输出token及requests；费用保留amount、list_amount、currency，不先转换成浮点数。

## 分步流程

### 1. 先固定范围与粒度，不让默认值改变报表语义

工程建议：先对一个已结束的UTC日做验证，例如2026-10-28T00:00:00Z至2026-10-29T00:00:00Z，设time_bucket_width='1d'、order_by='time'、order='asc'。源码契约要求带时间桶时显式ending_at；一般跨度最多31天，1m桶最多24小时。无时间桶时为整段聚合且边界按小时向下取整。

### 2. 分开查询用量与费用，保留一致查询条件

最小接口为client.beta.organization.analytics.usage.list(...)及cost.list(...)；共同参数包括starting_at、ending_at、group_by=['model','actor']和limit=1000。v2.3.0的两种实现都注入analytics-usage-cost-2026-08-24头并调用各自GET端点。此版本已含接口，不必把10月29日公告误写为新SDK版本。

### 3. 逐页保存后再提交完整结果

SyncReportPageCursor和异步对应类明确携带data_refreshed_at，并按next_page构造page参数；has_more=false才表明无下一页。工程建议：每轮采集暂存全部页，核对水位一致再整体替换分区，勿边读边累加到正式总账；410表示游标失效，应丢弃本轮暂存并从第一页重取，不能拼接两次快照。

### 4. 金额与请求数必须按字段含义处理

费用模型规定amount为折后、抵扣前的美分字符串，list_amount为折前美分。建议Decimal(amount)/Decimal('100')仅在展示美元时转换。先不拆cost_type/token_type保留requests；若需要token、web_search、code_execution、session_usage构成，再发细分查询。两种细分会令requests为空，不能按组件累加请求数。

### 5. 把归因结果接回评测，而不是自动下结论

actor_ids回答费用归属谁，api_key_ids回答由哪个key认证，二者不是别名。Managed Agents还可在适用组织中按agent_id、session_id或deployment_id归因。工程建议：与自有脱敏任务ID映射及成功率一起比较，每次变更只改变一个模型或提示因素；这些报表本身不证明因果、质量提升或真实节省。

## 关键机制

这是一组服务端聚合查询，而非逐请求trace。group_by最多五个常规维度；费用端额外cost_type/token_type不计此限，也不计页面limit，故一页实际行数可超limit。每个有数据的维度组合会跨页返回，分页参数不等于top-N筛选。SDK仅封装查询、类型与游标；后端导出、身份归因、刷新一致性仍取决于服务契约，源码不可证明真实账户数据完整。

## 来源事实与报告效果

- 10月29日新增的是统一API的beta公告，旧报表进入legacy但继续支持。：官方日期条目给出usage/cost两端点与beta头；Python最新稳定v2.3.0为10月23日，已经具有对应实现。头部的2026-08-24不是本次发布时间。 [原始证据](https://platform.claude.com/docs/en/release-notes/overview)
- 完整归因需要消费分页，而非仅取第一页。：SDK端点说明全部组合跨页返回，pagination.py显式实现next_page与has_more；返回水位可由采集器保存。该结论来自静态阅读，未以真实API回包验证。 [原始证据](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/pagination.py)
- 费用与requests存在容易误算的口径。：类型定义标明fractional cents及折前/折后字段；组件细分后requests为空。sandbox/code-execution的requests代表执行跨度而非HTTP请求，不能混用做平均每请求成本。 [原始证据](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/types/beta/organization/analytics/beta_analytics_cost_metrics.py)

## 限制与未知

- 仅完成公开材料与实现阅读，没有部署、安装SDK、API调用、基准测试或真实账单对账；不宣称任何降本比例。
- 该API不是实时预算闸门：通常4小时刷新，最长可24小时；截至刷新水位后的数据不完整，月末后约7天仍可能调整。正式结算不能只看当天快照。
- 组织类型决定可用维度；不适用的维度/筛选可能为空或匹配不到。工程上应先用无筛选总量建立对照，不能把空响应自动解释为零成本。
- 权限敏感：分析采集应与模型推理凭证分离，且不要将管理员密钥交给执行模型或写入报告；actor可能含姓名与邮箱，公开输出仅保留必要聚合。
- 成本不等于完整账单：Priority承诺费不在此明细中，amount也不是扣除credits后的最终应付额。

## 工程建议，非复现实验

- 最小验证不需要联网：自造两页、空页仍有next_page、空actor、组件requests=null及游标410的fixtures，验证完整性、类型和重启策略；本文只提出测试方案，没有冒称执行。
- 迁移采用双读对照：在相同UTC范围与相同口径下比较总token、费用和身份覆盖率；不一致先查日期取整、遗漏页、维度适用性，再查数据修订。
- 不要对公开API参考中的通用示例JSON做统计推断；字段契约比占位样例更重要。金额换算用十进制定点，保留原始美分值供复核。
- Python资源源码在v2.3.0已可核验；应用仍应固定依赖并先检查本地实际版本。未把旧版接口存在时间等同于本次beta开放时间。

## 阅读范围

完整阅读Usage and Cost Analytics指南正文（含两类迁移、actor、刷新与FAQ）；阅读usage/cost API相关参数、响应和游标契约；完整阅读v2.3.0 usage.py与cost.py同步/异步资源、pagination.py、cost_item及cost/usage_metrics类型，并核验Apache-2.0许可证与SDK发布日期。其余候选仅发布说明/PR级筛查，未声称完整研读服务端实现。

## 原始来源

- [Claude 平台10月29日发布说明](https://platform.claude.com/docs/en/release-notes/overview)
- [Usage and Cost Analytics：接入、迁移与刷新契约](https://platform.claude.com/docs/en/manage-claude/usage-cost-analytics-api)
- [usage API：分页、数据水位与游标过期](https://platform.claude.com/docs/en/api/beta/organization/analytics/usage/list)
- [cost API：费用聚合契约](https://platform.claude.com/docs/en/api/beta/organization/analytics/cost/list)
- [Python v2.3.0：usage同步/异步实现](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/resources/beta/organization/analytics/usage.py)
- [Python v2.3.0：cost同步/异步实现](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/resources/beta/organization/analytics/cost.py)
- [Python v2.3.0：分页游标](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/pagination.py)
- [Python v2.3.0：费用字段单位](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/types/beta/organization/analytics/beta_analytics_cost_metrics.py)
- [Python v2.3.0：usage指标与执行跨度](https://github.com/anthropics/anthropic-sdk-python/blob/v2.3.0/src/anthropic/types/beta/organization/analytics/beta_analytics_usage_metrics.py)
- [Python v2.3.0正式版日期](https://github.com/anthropics/anthropic-sdk-python/releases/tag/v2.3.0)
