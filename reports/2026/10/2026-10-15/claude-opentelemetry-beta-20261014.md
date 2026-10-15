---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-15
---

# Claude开放双端追踪：把请求、工具与代理回合接回业务trace

来源日期：2026-10-14
核查时点：2026-10-15T00:11:37Z（UTC）
状态：公开来源研读草稿，未独立复现实验。

## 核心问题
只有HTTP总耗时，难以区分一次代理任务是在客户端工具、平台内模型调用还是服务端工具上变慢。10月14日新增的是Claude SDK与Claude API原生OpenTelemetry导出：前者覆盖应用内步骤，后者补充平台内执行；两路可送入同一观测后端。它提供诊断证据，不自动判断任务是否真正完成。

## 输入
需要现有Claude API或Managed Agents工作负载，以及OpenTelemetry tracer provider。SDK追踪覆盖Python、TypeScript、C#、Go、Java、PHP、Ruby；本次核对Python v1.14.0。服务端导出需要管理员配置支持HTTPS 443、OTLP/HTTP protobuf的目的地及workspace范围。SDK元数据试验可先只输出到本地控制台。

## 输出
输出带时间、状态、模型与token属性的span。客户端请求为CLIENT，API请求为SERVER，模型执行和工具等为INTERNAL。会话可能跨多个trace；用anthropic.session.id聚合同一session，用gen_ai.conversation.id定位线程。默认metadata_only；启用content后才包含支持的提示词、响应和工具内容。

## 分步流程

### 1. 固定SDK，先验证一个最小请求
建议在测试环境安装anthropic==1.14.0和opentelemetry-sdk，按官方示例在首次请求前注册TracerProvider、SimpleSpanProcessor和ConsoleSpanExporter。用一条无敏感数据的Messages请求确认出现anthropic.messages.create CLIENT span；此阶段不必配置第三方观测服务。本次未实际调用API。

### 2. 按实际执行位置补齐span
SDK的tool runner负责客户端自定义工具；API负责内部模型及其执行的服务端/MCP工具。自托管Managed Agents sandbox中的工具需在worker进程启用SDK追踪，不能只开Console导出就认为完整。SDK core把一个方法调用及其重试放进同一call span，并用resend_count反映重发。

### 3. 只把测试workspace接到API导出
管理员先选Selected范围、metadata_only，再配置目的地；发送anthropic.test并在后端确认收到，然后运行一个真实测试请求，检查service.name=claude-platform。应用内包一层自己的span，SDK通过W3C traceparent传播关系，两路发往同一后端。AWS路径目前各自起trace，需按会话ID关联。

### 4. 对流式与内容上限做边界验证
若确需记录响应内容，Python应使用messages.stream()并读完或显式关闭流。官方测试明确区分create(stream=True)：能记usage等元数据，但不积累完整输出文本。内容按UTF-8字节计算，默认750,000字节；源码优先移除system及较早输入，工具结果则从尾部丢块，并标记anthropic.content.truncated。

### 5. 选定统计层级，再做去重与对账
Messages请求只选SDK或API一路汇总；已有API时优先API，并在request层与model_request层中择一。compaction另计；Managed Agents需计model_request和outcome_evaluation。用trace ID加span ID去重，缓存读写已包含在gen_ai.usage.input_tokens里，不要再加一次。建议用Usage API做完整用量核对。

## 关键机制
这是标准化埋点与上下文传播，不是新增推理模型。SDK侧围绕调用、流和tool runner记录，服务端把回合下的模型与工具拆开，再以trace上下文连接业务链。Python源码确认配置在client建立时解析；显式open_telemetry对象会整体覆盖对应环境变量，with_options也替换整个对象。只改conversation_id时应从client.open_telemetry复制，避免无意丢掉content mode或自定义provider。重试、取消和流提前关闭的span含义也不同，不能把没有ERROR一概等同于业务成功。

## 来源结果与证据

### 新增能力已有版本化实现，非仅合作公告。
官方release notes列于2026-10-14；GitHub release API给出Python v1.14.0的published_at为2026-10-14T17:13:08Z，发布内容包括API调用、流和工具追踪，以及Admin API遥测目的地。
来源：https://github.com/anthropics/anthropic-sdk-python/releases/tag/v1.14.0

### 配置替换、流式内容和截断行为能从作者测试交叉核验。
本次静态阅读对应测试：with_options替换整个配置；create(stream=True)没有gen_ai.output.messages；超限流式输出可挤掉最早输入并设置truncated。测试文件存在不等于本次测试通过，本次没有安装或运行测试。
来源：https://github.com/anthropics/anthropic-sdk-python/blob/v1.14.0/tests/test_open_telemetry.py

### token属性含义与普通响应字段不同，错误求和会重复统计。
响应实现把普通input_tokens与缓存读写相加后写入gen_ai.usage.input_tokens；compaction从iterations单独累加。作者测试对这些字段分别断言。没有本次实测的降本、提速或准确率结果。
来源：https://github.com/anthropics/anthropic-sdk-python/blob/v1.14.0/src/anthropic/lib/_tracing/_response.py

## 局限与未知
- 开放边界：服务端beta支持Claude API与Claude Platform on AWS；不能外推到所有托管平台。OpenAI兼容的/v1/chat/completions不产生对应Messages API服务端span。
- 可观察性不是完整审计：被拒绝的Managed Agents工具使用、multiagent/advisor工具本身不产生工具span；客户端工具与自托管工具也要另做SDK追踪。评估span的OK描述执行状态，不等于结果一定satisfied。
- 投递是best-effort且可能重复，丢失后不能补发。目的地返回重定向、解析到非公网IP或连续24小时全部失败会被禁用；不能依赖trace抓全量用量或泄漏密钥使用。
- metadata_only仍可能包含agent/tool名称与描述等自定义文字；Full content涉及额外数据副本及后端自身的留存/访问规则。图像、文档只留类型，超限内容也会删减，不能当作无损回放材料。
- SDK选项和span schema在beta期间可能破坏性变化。本次只核公开文档、版本化源码和选定测试，没有实际接入、性能测量或独立复现。

## 工程建议（非复现结论）
- 最小试用按流水线前两步开始：一条普通请求、一条正常流、一次提前关闭、一个成功和一个报错的只读工具。验收span数量、父子关系、状态、usage与内容字段，再接服务端导出。
- 先用脱敏合成数据与单个测试workspace。开启Full content前审查工具描述、conversation_id、传播baggage以及观测端的访问和留存；不要把metadata_only理解为自动匿名化。
- 建立固定样例：让同一请求包含缓存和compaction，验证按指定层级汇总；故意重复导入一个span验证去重，再将结果与权威用量来源对照。把观测缺失率与任务成功率分别记录。
- 建议把truncated、目的地Failing/Disabled与流提前关闭单独标记。抽样做eval时排除或显式标识缺失轨迹，避免将缺少证据误判为代理没有执行。

## 实际阅读范围
完整阅读Claude OpenTelemetry总览、SDK配置、API配置与span reference正文，并核对10月14日release notes；用官方GitHub release API验证v1.14.0精确发布时间。静态阅读该tag的_tracing.py、_content.py、_response.py；针对test_open_telemetry.py选读配置整体替换、关闭追踪、流式输出、字节截断和compaction计数测试，未声称通读整个测试文件。去重检查了公开日报2026-10-09至2026-10-14。未创建目的地、未上传日志、未运行SDK或调用计费API。

## 原始来源
- [Claude Platform release notes：2026-10-14发布OpenTelemetry beta](https://platform.claude.com/docs/en/release-notes/overview)
- [Claude OpenTelemetry总览：覆盖、内容模式与token统计](https://platform.claude.com/docs/en/manage-claude/opentelemetry)
- [Claude SDK追踪：安装、配置与流式读取](https://platform.claude.com/docs/en/manage-claude/opentelemetry/sdk)
- [Claude API追踪：目的地、权限、传播与投递边界](https://platform.claude.com/docs/en/manage-claude/opentelemetry/api)
- [OpenTelemetry span reference：父子关系、usage与工具盲区](https://platform.claude.com/docs/en/api/opentelemetry/spans)
- [官方Python SDK v1.14.0发布（2026-10-14T17:13:08Z）](https://github.com/anthropics/anthropic-sdk-python/releases/tag/v1.14.0)
- [Python v1.14.0追踪核心：调用、重试、配置与关闭流](https://github.com/anthropics/anthropic-sdk-python/blob/v1.14.0/src/anthropic/_tracing.py)
- [Python v1.14.0内容处理：UTF-8限额、截断与二进制排除](https://github.com/anthropics/anthropic-sdk-python/blob/v1.14.0/src/anthropic/lib/_tracing/_content.py)
- [Python v1.14.0响应usage：缓存与compaction计数](https://github.com/anthropics/anthropic-sdk-python/blob/v1.14.0/src/anthropic/lib/_tracing/_response.py)
- [Python v1.14.0官方回归测试：流、配置、截断及usage](https://github.com/anthropics/anthropic-sdk-python/blob/v1.14.0/tests/test_open_telemetry.py)
