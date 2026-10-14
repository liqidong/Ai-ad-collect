---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-14
---

# ScaleDown接入长运行代理：把压缩、摘要和抽取拆成窄任务

来源日期：2026-10-13
核查时点：2026-10-14T00:12:19Z（UTC）
状态：公开来源研读草稿，未独立复现实验。

## 核心问题
长运行代理每轮都把旧对话、读过的文件和工具输出重新放进上下文，成本会重复累积。即使主模型的规划能力足够强，也可能把时间花在摘要、复制订单号等很窄的工作上。10月13日这次新增的实用点是LangChain/Deep Agents中间件和LangSmith Gateway接入，把这些步骤从主模型循环里明确拆开；不应把它理解为主代理模型被替换，或只凭压缩比就认定任务质量提高。

## 输入
输入有三类：需要压缩的旧会话；只读工具返回的长文本和本次具体想找什么；用户原话与待抽取字段的说明。实际使用需要ScaleDown账户/API访问，以及能运行集成的Python环境。作者pyproject当前标0.2.0、MIT、Python>=3.10，依赖LangChain>=1.4.5、langchain-core>=1.4.0、LangGraph>=1.2.0；这些比README的宽泛版本描述更适合做安装检查。

## 输出
主模型收到较短的上下文，近期关键消息仍可保留原样；工具压缩结果注明压缩目标和前后长度，支持再取全文。抽取步骤把字段、置信度和原始实体结果存入scaledown_extraction，供后续工具读取。未找到的字段为空，避免为了凑齐工具参数而自造值；这些仍是待校验的数据，不应仅凭有置信度就允许外部写操作。

## 分步流程

### 1. 先只改一个上下文入口
以读取政策文档的工具为例，先保留原有主模型和工具实现，只给选定工具加ScaledownToolCompressionMiddleware。输入仍是正常工具参数，模型额外填写compression_query说明要找的内容。中间件先取走这个新增参数，再运行原工具，所以原工具无需理解压缩，也不会收到陌生参数。

### 2. 工具结束后再按问题缩短文本
中间件检查结果是否足够长、是否为纯文本、是否成功，然后把全文和compression_query交给压缩服务。输出替换成相关内容，并带压缩说明。错误、图片等非文本结果和较短输出保持原样；模型省略该参数即可请求全文。工程上应把它用于可安全重复的只读工具，不能为了取回文本再次触发付款或写入。

### 3. 历史接近窗口时独立做摘要
历史超过trigger后，摘要器接收较旧的消息，返回任务目标、已做工作、产物与下一步等信息；keep指定的近期内容不变。现行作者README建议Deep Agents保留自己的SummarizationMiddleware，仅注入ScaledownSummarizer，使被移出的完整历史仍能落盘回读。这个细节比把所有旧对话直接删掉更重要。

### 4. 原话抽取和分类走不同路径
每次agent invocation开始时，抽取器读取用户消息和字段schema。订单号、错误文本等字段走Extract，输出原文片段、位置和置信度；带固定labels的字段走Classify的Decisions API，输出允许标签。结果进入agent state。配合checkpointer，后续用户补充信息后会重新抽取；工具直接读字段，减少主模型改写标识符的机会。

### 5. 先看回退和保真，再看省多少
验证应同时覆盖正常压缩、全文回读和服务失败。快捷摘要中间件捕获ScaleDown API错误后保留旧历史；独立ScaledownSummarizer本身却会抛出错误，作者测试也明确验证这一点，必要时另配fallback。因此要按实际接入类设计故障测试，不能把“失败放行”当作所有路径都不会中断的保证。

## 关键机制
这里有两个不同的约束。内容约束是把任务缩小到摘要、压缩、抽取和分类，让不同模型只处理指定工作；接口约束是把抽取字段写入显式状态，把分类结果限制为给定标签。Classify官方文档说明，它针对每个标签的判断标准评分并归一化；标签含义重叠会影响分数，不能把0.8直接等同于经过业务校准的80%正确率。主模型仍负责开放式规划、代码和回答。建议把分类器用作额外信号，并保留确定性的预算、重试次数和权限检查。

## 来源结果与证据

### 可核查的是集成机制与回退实现，而非端到端速度保证。
作者仓库含独立中间件、摘要器和单元测试；本次静态核对了工具schema增强、参数剥离、错误跳过、仅抽取HumanMessage及摘要错误传播逻辑。没有运行线上API或复现质量/成本指标。
来源：https://github.com/scaledown-team/langchain-scaledown/blob/main/libs/scaledown/langchain_scaledown/middleware.py

### 发布文中的工具缩短例子只能当单例。
示例文本长度从18,402变成3,110字符；这不是token账单，也不等于整个任务省同样比例。本次不采用厂商“200x/467x”等头部宣传值作为结论。
来源：https://www.langchain.com/blog/building-long-running-agents-with-scaledown-and-deep-agents

## 局限与未知
- 开放范围不能一概而论：代码公开可读，但summarizer.py及作者README仍把sd_summarize标为private preview；LangChain现行集成文档又写明摘要输出最多200 tokens。使用前要确认账户实际权限和该版本输出限制。
- 压缩可能丢掉反例、限定条件或权限说明。官方集成文档明确不建议把短指令、创意任务或包含harness规则的system prompt直接压缩。
- 服务不可用时保留全文可避免丢数据，但上下文仍可能超限；抽取失败还可能留下上一轮字段。业务工具应检查字段的新鲜度和必填条件。
- 开源的是集成代码。旧会话、工具输出或用户消息会按所选中间件发往ScaleDown；本次未发送用户数据、未创建账户、未调用计费服务。

## 工程建议（非复现结论）
- 最小验证建议：选一批已知答案的长材料，埋入编号、否定词、数字单位、时间条件和彼此冲突的版本。对原文与压缩文分别跑同一任务，检查这些条件是否保留，记录最终答案正确率、全文回读次数、总token和包含压缩请求在内的总延迟。
- 先只启用指定只读工具的压缩，不全局改system prompt；保存原始输出的可回读位置。把压缩服务超时、额度耗尽、格式异常和无compression_query分别做测试，再判断是否放大范围。
- 若要加入漂移熔断，发布文提供了自定义after_model分类思路，但并非预制middleware。先在只记录标签的影子模式观察误报，验证后再让它中断任务；不要让概率分类独占安全边界。

## 实际阅读范围
完整读取2026-10-13发布正文、现行LangChain集成文档、ScaleDown Extract/Classify官方文档与作者README；静态读取middleware.py（blob SHA 523b0194a8d7d78329935cf7009a0196eebb0479）、summarizer.py和pyproject.toml，并查看作者摘要器测试及中间件测试用例。未安装依赖、未执行测试、未调用任何付费API。

## 原始来源
- [LangChain：Building Long-Running Agents with ScaleDown and Deep Agents（2026-10-13）](https://www.langchain.com/blog/building-long-running-agents-with-scaledown-and-deep-agents)
- [LangChain：ScaleDown middleware integration](https://docs.langchain.com/oss/python/integrations/middleware/scaledown)
- [ScaleDown作者实现：middleware.py](https://github.com/scaledown-team/langchain-scaledown/blob/main/libs/scaledown/langchain_scaledown/middleware.py)
- [ScaleDown作者实现：summarizer.py](https://github.com/scaledown-team/langchain-scaledown/blob/main/libs/scaledown/langchain_scaledown/summarizer.py)
- [ScaleDown：Extract overview](https://docs.scaledown.ai/api-reference/extract-overview)
- [ScaleDown：Classify overview](https://docs.scaledown.ai/api-reference/classify-overview)
- [作者包依赖与版本](https://github.com/scaledown-team/langchain-scaledown/blob/main/libs/scaledown/pyproject.toml)
- [作者单元测试：summarizer错误与回退](https://github.com/scaledown-team/langchain-scaledown/blob/main/libs/scaledown/tests/unit_tests/test_summarizer.py)
