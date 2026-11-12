---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-12
---

# MinerU续读协议深读：目标范围与已消费位置必须分开

核查时间：2026-11-12T00:06:50Z

来源时点：PR #5749于2026-11-10提出，2026-11-11 07:22:37 UTC合入dev（261f1fe2c57755fff15d2d0088e5a17e17752379）；mineru-4.2.1-released于2026-11-11 13:53:12 UTC正式发布，已核标签源码包含修复。

开放范围：11月11日已正式发行；代码及回归审阅，未安装运行

## 阅读范围与证据边界

已读PR #5749说明、维护者补充及全部5文件diff；读4.2.1完整续读测试及缓存夹具，核SDK/CLI完整相关文档。对照4.2.0和4.2.1读取计划、执行分发及下一请求生成代码；读取正式发行元数据、许可证、pyproject和CI运行状态。已读取公开归档10月9日至11月11日的日报JSON并按条目去重。本轮未安装或执行MinerU，未读取PR复现附件、全库或全部CI日志。 本期最终核查时间：2026-11-12T00:06:50Z。

## 核心问题与已有瓶颈

输出预算会把一次阅读拆成多次请求。4.2.0把最后消费的块/字符游标直接作为下一次locator，导致原来的整页阅读变成只读一个块；该块读完后，旧逻辑建议下一页，同页后续内容便被跳过。原PDF、解析缓存和完整Markdown仍可能完整，因此仅对OCR或导出文件做检查发现不了这个问题。

## 输入

不涉及新增模型训练。输入是Doclib已有的解析缓存、文档/页locator、输出软字符预算limit及context；默认limit=30000、context=0。后续请求携带上次next_request返回的locator和after。after可以精确到已读完的页、块或块内字符偏移，并须匹配同一文档及解析档位。尚未入库/解析的文件不能靠read自动创建解析任务。

## 输出

返回Markdown内容、实际content_ranges、truncated状态、request_scope及next_request。页内截断时，next_request以文档locator保留阅读语义，以after记录消费进度；当前页完成后才建议下一页。显式页范围读取继续保持原范围，结束时next_request为空。没有新增请求/响应字段，但过去忽略after的客户端必须调整。

## 逐步流程

### 1. 确认原目标及游标范围

读取计划先解析文档、档位、页/块目标，并验证after。新代码允许没有page选择的文档locator搭配after，但仍拒绝块目标加after、非Markdown和非零context；跨文档、跨档位游标不被接受。

### 2. 从消费位置恢复未完成页

文档级after若含块号，就定位回该页并继续块后或字符偏移后的内容；若after只到页，表示该页已经完成，从下一页开始。服务使用已有缓存，不重新调用解析模型。

### 3. 按软预算生成实际消费区间

既有渲染器按内容边界截断，记录content_ranges和是否页内截断。预算不是必须凑满的硬长度：默认30000字符的回归中，一页三个12000字符块会先返回前两个块，再续读第三块。

### 4. 生成与原语义一致的下一请求

直接块/字符目标沿用块内续读，不扩展成同页遍历；整页/文档在页内截断时返回文档locator加after；页间截断则使用渲染器提供的next_page_no。显式范围仍由原范围locator约束。

### 5. 客户端原样推进并验收

SDK应把next_request的两个字段一起传回read_content；CLI的Next提示也包含必要的--after。循环检查状态是否重复、是否到达空next_request，并将累计内容与已知完整样本核对，不能自行猜测下一个块号或页号。

## 关键机制与真实增量

这次把“要读什么”与“已经读到哪里”重新分开。locator表达阅读目标，after表达消费进度。仅把最后一个块当新目标，会丢失整页范围；改成page:1加after也不等价，因为已有协议会将这种组合视为有界页选择，读完后不再建议下一页。因此修复选择文档locator加after，同时保留显式范围和直接块目标的旧语义。源码改动集中在读取计划与下一请求生成，不改变渲染启发式、字符预算、模型或依赖。

## 来源报告的效果与证据

### 新增29项续读回归覆盖内容完整性及边界；本轮未执行。

已读完整测试文件：12组文档/单页×3种预算×标记开关，1组默认预算，6组直接块/字符，3组文档after，2组跨文档/档位拒绝，以及空块和编号跳跃、默认十页窗口、CLI标记、HTTP路由等用例。合成内容要求按顺序拼接为ALPHA、BRAVO、CHARLIE、DELTA，检查重复或遗漏，而非只看HTTP成功。

[原始证据](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/tests/unittest/test_doclib_read_continuation.py)

### 默认预算也能触发缺陷，不仅是极小测试预算。

新增test_default_budget_continuation_preserves_all_blocks构造第一页A/B/C各12000字符、第二页D为10字符；首次请求省略limit，断言默认30000，第一次返回A+B，后续为C再D。它直接验证每块只消费一次。

[原始证据](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/tests/unittest/test_doclib_read_continuation.py)

### 作者与维护者报告均为定向验证，不是识别质量或业务成功率提升。

PR原作者报告初版28项回归从12过/16失败变成28过；维护者补1项后报告29项续读用例、Doclib/CLI合计660项通过，这些数量不可相加。原作者另报告7场景PDF复现从2过变7过，完整导出哈希未变。本轮核到PR头提交对应CI运行42977027525为completed/success，但未逐项阅读全部日志。

[原始证据](https://github.com/opendatalab/MinerU/pull/5749)

### 4.2.1已经包含修复，但不覆盖带邻域的所有阅读方式。

已对照4.2.0与4.2.1标签的_next_read_request和读取计划，确认新target分流及文档after逻辑；发行说明明确context=0。CLI文档和PR都保留context>0块/字符窗口截断仍重复/遗漏的限制。

[原始证据](https://github.com/opendatalab/MinerU/releases/tag/mineru-4.2.1-released)

## 限制与未知

- context>0的块或字符上下文窗口仍存在已确认的截断续读缺陷，不应据此版本宣称“所有长文读取都无损”。
- 直接指定块时不会自动把同页其余块全部纳入；这是保留的目标语义，不是该修复再次漏掉整页。需要整页覆盖时从页/文档入口开始，并使用context=0。
- 服务已升级但调用方只转发locator、丢弃after，仍会破坏续读。协议字段未增加不代表客户端假设无需检查。
- 测试主要使用人工构造的PageInfo、真实SQLite/JSON缓存与进程内ASGI路由；验证的是读取协议，不衡量扫描件OCR、复杂表格解析、检索召回或最终回答正确率。
- 本轮未安装、运行回归或下载模型，也未复核PR附件PDF和复现压缩包。作者报告的环境与数量不能写成独立复现。
- Doclib本地读取和远端parse-server是不同层；此修复不自动授予上传文档、获取数据或绕过访问控制的权限。

## 工程判断、最小试用与建议

- 最小离线验证：在隔离环境固定mineru-4.2.1-released，按test依赖准备，执行python -m pytest -o addopts='' -q tests/unittest/test_doclib_read_continuation.py。先用测试缓存，不需要真实用户文档或模型推理；本轮没有执行该命令。
- SDK最小调用：已有缓存后执行result = client.read_content('doc:实际ID/tier:flash/page:1', context=0, limit=1000)；若next_request非空，下一次使用client.read_content(result.next_request.locator, after=result.next_request.after, context=0, limit=1000)。重复直到结束，并限制最大步数和重复的(locator, after)状态。实际文档ID由服务返回，不能照抄示例ID。
- 验收至少包含短预算、默认30000字符长页、页间截断、块内字符截断、显式不连续页范围、空块和缺号、非法跨文档游标，以及CLI/SDK/HTTP三种调用路径；将累计证据与完整导出作差异检查，不以任务完成提示代替内容完整性。
- 部署前提：固定版pyproject要求Python>=3.11且<3.15，DocVortex>=0.6.21且<0.7、pypdfium2>=5.13.0且<6，并有SQLite/HTTP服务等依赖。复用已解析缓存的续读不需要GPU；新增解析的算力取决于所选档位，不能由这项修复推断。
- 本地SDK通过UDS或loopback服务连接，需服务已运行及已有认证配置；DoclibClient构造本身不启动服务。官方建议保持loopback，不把本地文档库直接暴露到不可信网络。解析请求remote默认False，应保持该边界，试验只使用公开或获授权样本。
- 许可证不是无附加条件的Apache-2.0：4.2.1的LICENSE.md包含MinerU附加条款，合并口径MAU超过1亿或月总收入超过2000万美元需另行商业许可，向第三方提供在线服务有显著标识义务。实际应用还须分别检查依赖、模型和数据条款。
- 将本次变化视为Agent证据传输完整性修复：仍需在下游记录来源页/块、检查解析质量与引用覆盖，不能以修复续读推导更高幻觉抑制率或更强推理能力。

## 原始来源

- [MinerU 4.2.1正式发行说明](https://github.com/opendatalab/MinerU/releases/tag/mineru-4.2.1-released)
- [PR #5749：问题、修复范围、作者和维护者验证](https://github.com/opendatalab/MinerU/pull/5749)
- [4.2.1读取计划与续读实现](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/mineru/doclib/services/content_read_svc.py)
- [4.2.1完整续读回归测试](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/tests/unittest/test_doclib_read_continuation.py)
- [4.2.1 SDK读取与认证约定](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/docs/next/sdk/doclib-client.md)
- [4.2.1 CLI续读语义与已知限制](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/docs/next/cli/mineru-read.md)
- [修复前4.2.0读取实现](https://github.com/opendatalab/MinerU/blob/mineru-4.2.0-released/mineru/doclib/services/content_read_svc.py)
- [4.2.1许可证及附加条款](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/LICENSE.md)
- [4.2.1依赖与Python版本声明](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/pyproject.toml)
- [PR头提交CI运行结果](https://github.com/opendatalab/MinerU/actions/runs/42977027525)
- [合成页与真实缓存测试夹具](https://github.com/opendatalab/MinerU/blob/mineru-4.2.1-released/tests/unittest/test_doclib_page_selection.py)
