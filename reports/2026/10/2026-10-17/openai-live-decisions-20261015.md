---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-17
---

# 语音尚未说完就能做类型化判断：GPT-Live 原生接入 Decisions

核查时间：2026-10-17T00:04:04.444739Z（UTC；北京时间加8小时）

## 核心问题

语音代理需要快速驱动界面动作，同时把开放任务交给更完整的推理流程。

## 输入与训练前提

实时输入或输出转写、预定义的离散选项，以及应用回读确认的当前状态；需已建立的 GPT-Live 会话及 api.decisions.write 权限。

## 输出

带问题名的类型化答案；触发器结果还携带来源事件标识。应用据此执行动作，最终把真实结果返回 Live 供口头汇报。

## 关键机制

改动发生在语音前端与快速判断后端之间。Live 负责听说，Decisions 负责有限选项判断，应用负责权限与执行。复杂请求仍可交给 Responses；该分工减少一次转发，但不自动解决业务动作的正确性和安全性。

## 逐步流程

1. **1. 先把应用动作做成有限选择**：输入是明确的可执行动作集合。建议先用屏幕色块试验，只有 red、blue、none 三种结果；none 不改变界面。输出是问题与选项，而不是让模型直接写任意执行代码。此场景为建议试验。

2. **2. 选择原生委派或转写触发**：在 session.delegation 配置 decisions，Live 会直接调用 Decisions；或在 session.triggers 监听输入／输出转写增量。前者处理委派，后者可随话语更新界面。原生路径省去 Live 先通知应用、再由应用调用 Decisions 的转发往返。

3. **3. 把结果归到正确的请求**：委派保存 delegation.id，并匹配 decision.event 中的 delegation_id；读取 decision.completed 的答案。转写触发器则按 source_event_id 找回对应增量。结果可能合并或乱序，输入与输出方向分别只采用最新相关结果。

4. **4. 执行后读取真实状态**：应用执行选择，再读取实际状态。使用 decision.item.create 追加确认过的完整状态快照；等待 decision.item.created 后，以原 delegation_id 调用 session.commentary.append 把结果返回 Live。模型选择本身不是操作成功的证据。

5. **5. 用失败与改口检验整条链**：建议连续说出改口指令，模拟慢请求晚到、重复结果及 UI 更新失败。分别记录源事件、模型选择、实际状态与汇报；验收目标是旧结果不会覆盖新状态，失败不会被说成成功。尚未运行该测试。

## 来源事实与效果证据

- **支持边听边判断，或随 Live 发言更新界面。**：官方更新日志列为原生集成；指南展示控制与虚拟角色示例，没有发布独立端到端基准。 [原始证据](https://developers.openai.com/api/docs/changelog)

## 限制与未知

- 需可用的 GPT-Live 连接和 api.decisions.write；不需 Decisions 专属 alpha header。指南示例使用 gpt-live-1 与 gpt-6-luna。
- delegation 类型会话内固定；更新问题后须等待 session.updated。使用事件过滤时，必须放行 decision.event 和对应源转写事件，否则收不到触发结果。
- 官方材料说明结构性少一次往返，没有给这条完整工作流发布独立端到端速度、正确率或费用基准。不能把 Decisions 的其他速度宣传直接套到应用延迟。
- GPT-Live-1.1 应用端更新不代表其 API 已开放；本文复现入口以已核读指南模型为准。

## 工程建议（未经本次复现）

- 先跑无副作用界面，再接真实业务。建议为每项动作加权限检查、重复执行保护和错误回传；不要在用户尚未说完时触发不可逆操作。
- 事件 ID 用来关联来源，不应假定字符串字典序就是时间序；建议应用记录实际源事件顺序，为每个方向维护最后已应用的位置。此为工程建议。
- 不要在每个转写碎片后覆盖整个历史。状态变化后追加最新完整快照，既能避免只记局部更新造成歧义，也遵循指南保留缓存前缀的做法。
- 浏览器试验需要 HTTPS 或 localhost、麦克风与可信服务端。密钥保留在服务端；结束后关闭会话、核对语音时长和后端用量。

## 阅读范围

已读 Connect voice to Decisions 的完整主体（原生 delegation、执行回报、转写 triggers、乱序处理）及代码示例；未运行示例、未测延迟。 另读 Getting started with GPT-Live 主体全文。

## 原始来源

- [OpenAI API 更新日志：2026-10-15](https://developers.openai.com/api/docs/changelog)
- [Connect voice to Decisions](https://developers.openai.com/api/docs/guides/decisions-voice)
- [官方论坛 10 月 16 日补充](https://community.openai.com/t/gpt-live-1-1-is-rolling-out-with-new-voices-and-better-accents/1406704)
- [Getting started with GPT-Live：前端与后端责任](https://developers.openai.com/api/docs/guides/live)
