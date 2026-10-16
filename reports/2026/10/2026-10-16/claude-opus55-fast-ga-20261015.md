---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-16
---

# Opus 5.5快速模式取消申请门槛：重点是部署方式变化

核查时间：2026-10-16T00:03:07.676638Z（UTC；北京时间加8小时）

## 核心问题

交互式长回答的输出等待较长；此前快速模式还需申请。今日是开放门槛变化，不是首次发布快速推理。

## 输入与训练前提

Messages API：model=claude-opus-5-5、messages、max_tokens，另设speed=fast。

## 输出

常规回答；usage.speed用于核验实际速度档位。

## 关键机制

同一模型权重使用更快推理配置；主要提高持续输出速率，不能据此推算首token或整项任务同幅提速。

## 逐步流程

1. **确认可用范围**：10月15日起，Opus 5.5每个usage tier都有默认fast限额，无需申请或beta header；保留旧header仍兼容。

2. **最小试用**：用已有API账户提交一条非敏感提示，固定输出上限并开启fast；本文未实际调用。

3. **验收**：检查usage.speed，再用同提示和上限比较标准模式。建议分别记录首token等待、输出耗时、总费用。

4. **处理容量不足**：429后可等待，或由客户端去掉fast参数重试；回退不是自动保证。

## 来源事实与效果证据

- **输出token速率最高2.5倍**：官方文档上限表述；未独立实测，不是保证值。 [原始证据](https://platform.claude.com/docs/en/build-with-claude/fast-mode)

- **10月15日Opus 5.5退出research preview**：更新日志明确移除申请和beta header要求；Opus 5与4.8仍保留预览要求。 [原始证据](https://platform.claude.com/docs/en/release-notes/overview)

## 限制与未知

- fast另有配额；标准与fast不共享提示缓存。
- Opus 5.5每百万token输入8美元、输出40美元；其他计费修正另计。
- 不支持Batch和Priority Tier。
- 可用Claude API；Foundry限Azure托管，Anthropic托管不适用。Bedrock、Claude Platform on AWS及Google Cloud不支持。

## 工程建议（未经本次复现）

- 建议先测长输出场景；若耗时来自工具或网络，收益可能较小。
- 建议统一日志记录速度档位和缓存命中，避免把切换成本误判为模型价格变化。

## 阅读范围

直接打开官方10月15日更新记录及Fast mode全文；另通过HTTPS读取原始HTML核对被网页提取遗漏的云平台范围。仅文档核验，无API实测。

## 原始来源

- [Claude Platform release notes：2026-10-15](https://platform.claude.com/docs/en/release-notes/overview)
- [Fast mode：机制、接入和限制](https://platform.claude.com/docs/en/build-with-claude/fast-mode)
