---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-23
---

# MarkItDown：新增文档模型选择、查询字段和结构化输出

来源日期：2026-10-22T22:31:47Z（北京时间10月23日06:31）
核查时点：2026-10-23T00:09:00Z（UTC）
状态：公开来源研读草稿，未独立复现实验。

## 核心问题
此前 MarkItDown 的 Azure Document Intelligence 适配主要输出布局 Markdown，未充分暴露服务中的模型选择和结构化字段。知识库只保留正文便需要二次推断作者、标题等属性；如果又把多个文档字段合并，同名字段会混淆。改动解决配置入口与序列化，不是新 OCR 模型，也不训练 RAG 模型。

## 输入
文件字节流、扩展名/MIME、Document Intelligence endpoint 与授权凭据；可选 docintel_model_id、docintel_features、docintel_query_fields、API版本。默认模型仍是 prebuilt-layout。PDF/图片可以携带 OCR add-on；Office 类型跳过它们。没有训练阶段。

## 输出
DocumentConverterResult.markdown：服务正文去除 HTML 注释后，若有有效字段则前置 YAML。单文档为 fields 映射，多文档为服务顺序的 documents 列表，各自 fields 独立；部分空文档保留空映射，全部字段为空则不输出 front matter。这不是 AnalyzeResult 全量导出。

## 分步流程

### 1. 配置模型和数据出口
输入文件及 CLI/Python 配置。CLI 将模型、功能、查询字段转给 MarkItDown，再交给转换器。显式 credential 优先，否则读 AZURE_API_KEY，再退到 DefaultAzureCredential。输出配置后的客户端；上传前必须检查数据授权和成本。

### 2. 按格式限制功能
输入 StreamInfo 和功能列表；_analysis_features 根据扩展名/MIME识别 Office 并返回空列表。对OCR输入，复制显式 features；如有查询字段，自动加入 QUERY_FIELDS。输出本次允许发送的 features/query_fields，缺省不请求 add-on。

### 3. 运行云分析
读取文件字节，指定 model_id 发起 begin_analyze_document，要求 Markdown格式并等待结果。输出 content 与 documents。处理发生于Azure，不是离线本地推理；准确率、时延和费用不能从接口变化推断。

### 4. 映射字段且保留文档边界
逐文档读取最具体的 typed value，日期转ISO，数组/对象递归处理，选择组成为已选值列表，缺少类型值时回退content。空值跳过但不跨文档合并字段，形成有序结构。

### 5. 序列化并接入知识库
YAML emitter引用或转义特殊字符、数字形/布尔形字符串和控制字符，保留科学计数浮点类型；将前置信息拼到正文前。下游用安全解析器提取元数据，并另存文件ID、来源及原文定位后才切分正文。

## 关键机制
增量是适配层契约：model_id使调用从固定布局转向可选模型；features/query_fields显式控制服务能力，查询字段会启用关联add-on。字段输出采用有类型递归映射，单文档与多文档使用不同顶层形态，避免重名字段覆盖。序列化只留下字段值，未携带全部置信度、坐标等原始元信息；正确的字段边界不代表字段本身正确。

## 来源结果与证据

### 新增能力可以从固定源码核查
CLI、参数转发、转换器与README同时变更；convert将model_id、features及适用时的query_fields传给服务。无数据集、精度、延迟对比或论文表号，不能当作质量提升报告。
来源：https://github.com/microsoft/markitdown/blob/7482c092d9c0074a89803c9fdb63a969e7fe17fc/packages/markitdown/src/markitdown/converters/_doc_intel_converter.py

### 作者提供边界测试，不等于本次测试已通过
新增562行测试文件使用mock客户端，不调用网络；覆盖单/多文档、空位置、选择组、控制字符、科学计数浮点、Office跳过功能和参数转发。本次仅阅读，没有执行。
来源：https://github.com/microsoft/markitdown/blob/7482c092d9c0074a89803c9fdb63a969e7fe17fc/packages/markitdown/tests/test_docintel_converter.py

### 不默认启用分析add-on
默认features为空，提供query_fields会加入QUERY_FIELDS；README说明formulas、ocrHighResolution、styleFont和queryFields可额外计费。基础分析并不因此免费。
来源：https://github.com/microsoft/markitdown/blob/7482c092d9c0074a89803c9fdb63a969e7fe17fc/README.md

## 局限与未知
- 已核对GitHub compare：该提交在v0.1.9之后1个提交，不在10月21日发布的v0.1.9中。需固定源码版本，不应假定pip稳定版已含新接口。
- 未调用Azure或运行测试，未评估中文扫描件准确率、吞吐或实际账单。
- Office跳过OCR add-on/query fields；模型、区域和格式支持需核对Azure矩阵。
- 文件字节会上云，权限、许可、区域、保留策略和敏感信息必须先评估。
- 输出是有损字段投影；货币可能变成金额加币种的字符串，不能直接当规范财务值。
- documents顺序不等于自有文档ID，需自行保存来源映射和定位。

## 工程建议（非复现结论）
- 最小试用（未执行）：固定本提交并安装az-doc-intel可选依赖，在获准的Azure资源上先用公开合成PDF、不加收费功能转换，再单独开启query fields比较输出和费用。
- 以00123、on、科学计数、控制字符、多个同名字段文档及中间空文档建立回归。使用safe_load，不执行YAML对象构造。
- 元数据过滤可减候选，但抽错字段会假阴性；区分机器抽取字段和人工确认字段，保留原始响应供审计。

## 实际阅读范围
已读固定提交README相关配置段、完整转换器、CLI/转发diff、新增测试及Azure官方add-on支持计费段。是原始文档/代码研读，非论文全文研读；未运行服务或测试。

## 原始来源
- [10月22日官方合并提交 #1913](https://github.com/microsoft/markitdown/commit/7482c092d9c0074a89803c9fdb63a969e7fe17fc)
- [固定提交 README](https://github.com/microsoft/markitdown/blob/7482c092d9c0074a89803c9fdb63a969e7fe17fc/README.md)
- [转换器完整源码](https://github.com/microsoft/markitdown/blob/7482c092d9c0074a89803c9fdb63a969e7fe17fc/packages/markitdown/src/markitdown/converters/_doc_intel_converter.py)
- [新增无网络单元测试](https://github.com/microsoft/markitdown/blob/7482c092d9c0074a89803c9fdb63a969e7fe17fc/packages/markitdown/tests/test_docintel_converter.py)
- [Azure add-on 支持与计费边界](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept/add-on-capabilities?view=doc-intel-4.0.0)
- [v0.1.9与本次提交的官方对比](https://github.com/microsoft/markitdown/compare/v0.1.9...7482c092d9c0074a89803c9fdb63a969e7fe17fc)
