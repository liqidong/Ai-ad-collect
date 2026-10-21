---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-21
---

# OpenAI 12 小时提示缓存：用更贵的首写，换取跨小时复用

核查时间：2026-10-21T00:01:59Z。状态：草稿；仅公开来源研究，未经本地复现实验。

## 核心问题与本次增量

日报增量是 2026-10-20 新增的 12 小时缓存档位，不是提示缓存首次发布。长规则、工具与资料只在上午、下午各用几次时，短缓存可能反复重写；新档位让工程师把保留时长纳入成本设计，而不是只优化命中率。

## 输入与前提

Responses API 请求，支持 prompt_cache_options 的 GPT-5.6 及后续模型；同一稳定前缀至少 1,024 个可见输入 token。请求级 prompt_cache_options.ttl 可设为 12h；显式断点可以单独设 ttl。

## 输出

正常模型回复不变；额外需要读取 usage.input_tokens_details 的 cached_tokens、cache_write_tokens，必要时读取 prompt_cache_diagnostics。这里的产出是可观测的缓存计费行为，不是答案质量提升。

## 关键机制

缓存复用的是稳定前缀的 KV 状态。新增 12h 是最低计费覆盖时长，不是 12 小时后必删，也不保证每次物理 KV 命中或延迟下降。从已有 30m 覆盖升级到 12h 会重新按长写入收费，不抵扣旧写入。因此应在首次写入前根据复用间隔选档，而非看到未命中再盲目升级。

## 逐步流程

### 1. 先拆稳定资料和即时问题

把全天重复的说明和参考材料置前，变化的问题置后。建议显式模式只缓存前缀，避免把一次性问题也写入长缓存。

### 2. 设置长短两级断点

请求默认仍可用 30m，而稳定资料的 prompt_cache_breakpoint.ttl 设 12h。后面的短期会话内容使用默认期限。断点覆盖其前全部内容，所以越靠后的长断点会扩大长写入范围。

### 3. 写入、复用并核算

首次命中不到已有前缀时按所选期限写入，后续兼容请求复用。30m 写入为普通输入的 1.25 倍，12h 为 2.5 倍，均是总费率而非额外加收；读价不随期限变化。

### 4. 用诊断闭环而非凭感觉

保留首个 response.id，下一次通过 comparison_response_id 对照；若未复用，检查模型、工具、输入及服务档位变化。诊断不会自动加载旧会话，也不会强制缓存命中。

## 来源事实与报告效果

- 2026-10-20 正式列入 API changelog。
  - 证据：官方列在 v1/responses Feature；不是仅靠搜索摘要推断日期。
  - 原始来源：https://developers.openai.com/api/docs/changelog
- GPT-6.1 Sol 标准短上下文价格提供可核算案例。
  - 证据：每百万 token：普通输入 $2、缓存读 $0.10、30m 写 $2.50、12h 写 $5。
  - 原始来源：https://developers.openai.com/api/docs/pricing
- 示例推算：20,000-token 稳定前缀，在 12 小时内使用三次且短缓存每次均已过期，长档前缀成本约 $0.104，短档反复写入为 $0.150，约少 30.7%。
  - 证据：本日报按官方单价计算，不是OpenAI实验结果或本地实测：0.02×5 + 2×0.02×0.10；对照 3×0.02×2.50。忽略相同的新输入及输出费用，并假设后两次完整复用。
  - 原始来源：https://developers.openai.com/api/docs/pricing

## 局限与未知

- 不是一律更便宜：在同样假设下仅使用两次，12h 为 $0.102，30m 两次写为 $0.100；频繁的半小时内请求通常没有必要买长档。
- 12h 不是数据删除 SLA。旧模型的 prompt_cache_retention 与本次 ttl 不是同一参数。
- 官方没有给本次档位发布独立吞吐或延迟基准；没有开展真实 API 测试。诊断 cache_hit 也不等于整条输入都免费或全命中。

## 工程建议（非实验结论）

- 最小试用建议：用无敏感数据的固定长文档，在支持模型上对比 30m/12h 两组；首写后间隔超过 30 分钟但少于 12 小时重复两次，固定模型、工具及服务档位，记录 token 分类、总成本和首 token 时间。本次仅设计试验，未调用付费 API。
- 另做一次工具重命名的负例，验证诊断是否暴露 tools_changed；不要用单个成功请求代表生产命中率。
- 上线门槛建议：根据真实复用次数和间隔回放成本，而不是直接把所有会话改为 12h。

## 阅读范围

已打开官方 changelog、Prompt caching 正文的机制/断点/12h 生命周期/模型差异/优化章节、Diagnostics 核心正文和 Pricing 标准价格表；不是摘要阅读。未阅读全部站点文档，未运行 API 或复现实验。

## 原始来源

- [OpenAI API changelog，2026-10-20](https://developers.openai.com/api/docs/changelog)
- [Prompt caching：期限、断点与计费机制](https://developers.openai.com/api/docs/guides/prompt-caching)
- [Prompt cache diagnostics](https://developers.openai.com/api/docs/guides/prompt-caching/diagnostics)
- [OpenAI API pricing](https://developers.openai.com/api/docs/pricing)
