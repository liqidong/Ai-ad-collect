---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-24
---

# RAGFlow保留PDF完整图像框：别把识别到的几个字当成整张图

核查时间：2026-10-24T00:08:34Z。公开研究草稿；不是独立复现实验或采用结论。

## 核心问题与现有瓶颈

#21390于2026-10-23 09:06:47 UTC合入main，fd675ebde21a8754184e486c33f10ac7e12f55ff；#21412于11:09:59 UTC合入main，651a1c65fce44a15a1612cd6a8e0b97cf177d2bc。两者均为当日nightly a55dccee的祖先（分别ahead 9/5、behind 0）。这是Go开发分支与nightly增量，不是新稳定版效果声明；同日v0.27.3标签与这两条提交分叉。

旧Go DeepDOC虽已从DLA版面模型得到完整figure边界，却可在整理时用图内少量文字框的并集重建图像区域，造成完整图被裁成局部文字；无文字图还可能在box→section阶段被直接丢弃。另一个独立问题是媒体预览复用文字预览裁剪，在上下加120pt暗化背景带，混入邻近内容。

## 输入

1. PDF页渲染、DeepDOC/DLA检测出的figure/equation区域、图内原生或OCR文字框、页号与渲染尺度。
2. 图注、image_context_size上下文或可选VLM描述是后续生成可检索文本的独立输入，不由保留几何自动产生。
3. 媒体预览裁剪使用typed PDF positions及已解码的页图；多页区域按页面顺序拼接。

## 输出

1. 带完整DLA区域、正确页号和坐标的figure sections；即便Text为空也可序列化为doc_type_kwd:image并保留_pdf_positions。
2. 后续VLM或chunker可按需重读原PDF裁图；媒体chunk预览不再混入文字预览的上下暗带，但保留实现中的6px拼接间隔。
3. 能否进入最终索引取决于是否存在可检索文字或上下文；保留图片位置不等于默认获得向量化图像检索。

## 分步流程

### 1. 让每个图像区域携带自己的完整几何

AnnotateBoxLayouts不再只为未命中文字的figure/equation补空框，而是为每个有效DLA区域创建带FigureBounds的几何框。figure与equation的标识命名空间分开，图内文字仍以相同布局标识与几何关联。

### 2. 阻止文字合并改坏图像边界

布局合并跳过figure框；ConsolidateFigures聚合同布局文字后，applyFigureBounds再把坐标钉回DLA区域，而不是文字框并集。table匹配跳过这些figure几何框，避免同一区域被表格检测误吸收。

### 3. 保存无文字图的位置契约

BoxesToSections只跳过无文字的非figure框；无文字figure保留。解析器输出image项和_pdf_positions，正常CGO路径不在解析阶段内联base64图像，而由后续按需裁剪。页坐标在边界做0基→1基规范化。

### 4. 媒体chunk预览使用独立的干净裁剪

#21412的CropMediaSectionPositions复用buildSectionRasterPlan，仅保留非isEdge的正文段，再重新测量、渲染和编码；chunk类型为text仍走CropTextSectionPositions。由此去掉图/表预览上下暗带，不改变文字预览语义。

### 5. 显式检查能否索引

Tokenizer过滤canonical text和上下文都为空的项；token chunker也对移除坐标标签后的合并文本做空检查。所以纯图需图注、上下文或VLM描述等实际文字才能进入这条文本检索链路。

## 关键机制

核心是把“图像区域的几何基准”与“图内识别文字”解耦：DLA区域负责裁剪范围，文字仅作为附加内容，不再反过来缩小图像。第二个修复把图/表预览和文字预览的上下文策略分离。它不是新的视觉模型，也没有训练或修改DLA模型参数。

## 来源事实与报告结果

- 完整边界、页号和渲染缩放具有明确回归断言。
  - 证据：TestEnrichFigureBoundsUseRenderScale把检测区域按scale=9还原到(10,20)-(90,60)且保留page=4；TestParsePreservesFigureWithoutText验证纯图保留、Text仍为空且页号/坐标正确，未伪造文字。
  - [原始来源](https://github.com/infiniflow/ragflow/blob/fd675ebde21a8754184e486c33f10ac7e12f55ff/internal/deepdoc/parser/pdf/figure_bounds_test.go)
- 官方新增真实PDF边界检查，但它只验证选定图像的最小几何，不是广泛准确率评测。
  - 证据：TestAN0590FigureBounds要求本地an0590scv140.pdf及DeepDOC模型，检查零基page 1存在宽≥240、高≥50的图，以及page 3宽≥350、高≥100的图。未设AN0590_PDF时测试跳过。PR作者勾选已运行，本轮没有运行。
  - [原始来源](https://github.com/infiniflow/ragflow/blob/fd675ebde21a8754184e486c33f10ac7e12f55ff/internal/deepdoc/parser/pdf/figure_bounds_integration_test.go)
- 媒体预览回归锁定无暗带及跨页顺序。
  - 证据：单页100px高主体输出100×106（含6px间隔），边缘保持未暗化红色；跨两页100+100px主体输出100×212。两块媒体的预览断言由388px改为112px。数字是人工构造测试的几何断言，不是吞吐或质量提升。
  - [原始来源](https://github.com/infiniflow/ragflow/pull/21412/files)
- 没有端到端RAG召回、回答准确率、延迟或成本的量化结果。
  - 证据：两个PR提供实现、回归代码与维护者测试说明，未提供检索/问答基准。本轮未独立复现。
  - [原始来源](https://github.com/infiniflow/ragflow/pull/21390)

## 限制与未知

- DLA区域自身若漏检或边界错误，保留该边界不会修复模型错误。该变更处理图像几何与访问，不覆盖原生PDF文字恢复或补充OCR；PR将后者明确列为独立工作#21399。
- 无图注、无有效image_context_size上下文且无VLM描述时，纯图仍会被后续文本索引链路过滤。不能写成“纯图片现在自动可搜索”。
- #21412只把chunker的图/表预览路由到CropMediaSectionPositions。固定nightly的VLM按需裁图仍调用CropSectionPositionsRasterLimited，不能声称所有下游VLM图片都已去掉上下文暗带。
- 图内若只有“数据来源：…”归因行，新Go路径去文字但保留几何；PR明确这与Python把整个区域丢弃不同。涉及来源归因的资料，应另保留原文/元数据，不能把被删除文字当作许可变化。
- 真实文档测试需要外部PDF文件和原生模型，本轮未下载或运行；没有证据说明对所有中文报表、扫描件或多栏文档都有效。

## 工程建议（未复现实验）

- 最小试用建议（未执行）：固定a55dccee的隔离环境，拿一份有权使用的技术PDF，覆盖图内少量文字、完全无字图、图表重叠检测、跨页媒体四类。分别检查DLA区域、section坐标、_pdf_positions、最终预览和索引chunks，不只看页面显示成功。
- 先跑官方单元回归：bash build.sh --test ./internal/deepdoc/parser/pdf/... ./internal/parser/parser/... ./internal/ingestion/component/chunker/。真实AN0590回归需自行合法取得对应PDF并准备MODEL_DIR；未设置环境变量会跳过，不能把整体绿色当作该样例已运行。
- 为纯图分三组：无上下文/VLM、只给图注或邻近文字、启用VLM。比较是否产生chunk及证据是否完整，再自行记录人工边界覆盖和问答正确性。不要从新增测试断言推算效果增益。
- 升级不会自动重解析已入库文档，应先选择受影响小样本重跑并保留旧证据。若启用外部VLM，裁出的页面/图片会涉及数据发送与费用，应先核验授权、供应商留存政策及脱敏。
- 合规限制：软件解析能力和安全修复不授予抓取、复制、上传或使用图片的权利；遵守资料许可、站点规则、访问控制和个人数据最小化。

## 实际阅读范围

完整阅读#21390的13文件差异和#21412的6文件差异、对应PR正文/合并元数据。补读固定nightly的PDF序列化边界、Tokenizer过滤段、token chunker过滤段与VLM按需裁剪函数，核对无文字图和预览/VLM路径限制。核查nightly标签及祖先关系；未编译、未运行、未做模型/业务性能基准。

## 原始来源

- [RAGFlow #21390：完整PDF图像边界、范围与行为说明](https://github.com/infiniflow/ragflow/pull/21390)
- [完整图像框修复的13文件差异与测试](https://github.com/infiniflow/ragflow/pull/21390/files)
- [RAGFlow #21412：媒体预览不再带上下文暗带](https://github.com/infiniflow/ragflow/pull/21412)
- [媒体裁剪修复的6文件差异与测试](https://github.com/infiniflow/ragflow/pull/21412/files)
- [固定SHA：DLA几何框保留和table/figure匹配隔离](https://github.com/infiniflow/ragflow/blob/fd675ebde21a8754184e486c33f10ac7e12f55ff/internal/deepdoc/parser/pdf/table/table_annotate.go)
- [固定SHA：合并后回钉完整图像框](https://github.com/infiniflow/ragflow/blob/fd675ebde21a8754184e486c33f10ac7e12f55ff/internal/deepdoc/parser/pdf/table/table_post.go)
- [固定SHA：真实文档AN0590图像边界回归](https://github.com/infiniflow/ragflow/blob/fd675ebde21a8754184e486c33f10ac7e12f55ff/internal/deepdoc/parser/pdf/figure_bounds_integration_test.go)
- [固定SHA：无上下文带及跨页媒体裁剪回归](https://github.com/infiniflow/ragflow/blob/651a1c65fce44a15a1612cd6a8e0b97cf177d2bc/internal/deepdoc/parser/pdf/util/crop_test.go)
- [固定nightly：Tokenizer仍过滤无文本且无上下文的媒体](https://github.com/infiniflow/ragflow/blob/a55dccee4c211046b4e4d216a538bc60fae78dfd/internal/ingestion/component/tokenizer.go)
- [固定nightly：chunker以带上下文文本判定是否可索引](https://github.com/infiniflow/ragflow/blob/a55dccee4c211046b4e4d216a538bc60fae78dfd/internal/ingestion/component/chunker/token.go)
- [固定nightly：VLM按需裁图仍走受限裁剪路径](https://github.com/infiniflow/ragflow/blob/a55dccee4c211046b4e4d216a538bc60fae78dfd/internal/ingestion/component/vision_enhancement_cgo.go)
- [固定nightly：解析边界保留坐标并延迟生成图片](https://github.com/infiniflow/ragflow/blob/a55dccee4c211046b4e4d216a538bc60fae78dfd/internal/parser/parser/pdf_parser_common.go)
- [完整图像框修复已在nightly祖先链](https://github.com/infiniflow/ragflow/compare/fd675ebde21a8754184e486c33f10ac7e12f55ff...a55dccee4c211046b4e4d216a538bc60fae78dfd)
- [媒体预览修复已在nightly祖先链](https://github.com/infiniflow/ragflow/compare/651a1c65fce44a15a1612cd6a8e0b97cf177d2bc...a55dccee4c211046b4e4d216a538bc60fae78dfd)
- [nightly发布记录](https://github.com/infiniflow/ragflow/releases/tag/nightly)
