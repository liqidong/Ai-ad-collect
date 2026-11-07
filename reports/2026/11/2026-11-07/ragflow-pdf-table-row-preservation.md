---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-07
---

# RAGFlow 1.0：保住PDF表格漏识别行，不再把数据并进合并表头

来源日期：2026-11-06 10:03:00 UTC合并；v1.0.0于12:31:31 UTC正式发布

核查时间：2026-11-07T00:06:27Z

状态：公开研究草稿。来源事实、作者结果与工程建议分别标注，未独立复现。

## 问题
Go版DeepDoc把表格结构识别（TSR）输出还原为行列时，未匹配到结构行的文本框过去会保留Go默认R=0，继而混入第0行；另一个问题是用已经被跨行单元格拉大的网格行去匹配文本，可能把两层合并表头压成一层。文本虽然没有完全消失，其所在数据行或列却可能错位，给下游RAG留下错误关联。

## 输入与输出
输入：PDF页图像、DLA定位的表格区域、已有文字/OCR文本框，以及TSR给出的行、列、表头和跨行/跨列框；文本框与结构框在同一裁剪坐标系中处理。本补丁不训练或替换OCR、DLA、TSR模型。

输出：保留逻辑行序、单元格文本与跨行/跨列几何的表格网格及R/C/H/SP标注，再交给后续表格构造与渲染；输出并非最终问答答案，也不保证TSR未识别出的所有结构都能恢复。

## 流程

### 定位并裁剪表格，保留独立文字框
生产入口processOneTable对表格区域裁剪并运行TSR，将文字框转到裁剪坐标。完成行列归属前不先把相邻文字合并成段落，防止跨行混合；本次变更接在既有表格构造链路内。

### 分开原始结构行与扩展后的合并网格
GroupCells用GridRowBands构建结构网格；AnnotationRowBands另收集原始row/header行带供文字标注。两者的角色不同，不能用扩展过的rowspan网格框代替原始行带。跨行/跨列框注入使用原始行列中心，避免先处理的长跨度框改变后续匹配中心。

### 先标记未知，再匹配行列与合并起点
AnnotateBoxesWithGridAndRows先将R重置为-1，并设置HasTableAnnotations以区分真正标注与Go零值；只有匹配成功才赋行号。列边界从非跨列、非零坐标单元格恢复。跨行框中的文字即使垂直居中在下层，也将归属移到合并区域首个物理行，且使用同一套行索引，跳过表头组件自身作为起点。

### 按纵向位置保留TSR漏掉的行
sortRFirstly先依据页码、Y和X排序，再对有行号的邻接框调整顺序；没有行号的框保持其Y位置。assignRows遇页变化、行号变化，或缺失行号且Y越过当前行底部时开启新行。同一已知行中的多行OCR文字保持一行，而不是被简单Y间隔拆开。

### 重建统一列宽并保留跨度语义
同一单元格内的文字合并，整个表共享列编号映射，避免每行单独压缩导致列错位。兼具表头与跨行/跨列标记时优先保留spanning标签，后续CalSpans据此生成rowspan/colspan并归并覆盖区域文字。表头向相邻行传播也从任意重叠收紧为至少覆盖该行高度的50%。

### 维持缺标注回退并写回生产结果
完全未标注的普通文字框继续走Y/X几何分组，不能因为默认R=0/C=0就当作已识别首行。processOneTable和WriteTableAnnotations都传入原始行带；DeepDoc的FinalizeGrid在此仍为直接返回，不增加新的OCR推理阶段。

## 关键机制
核心是把“未知行”“第0行”和“跨行框覆盖范围”区分开：先保留原始结构证据与未知状态，再用几何位置补齐分组，最后处理跨度。它把Python construct_table的部分行分组语义移植到Go路径，修复结构重建的一致性；不是新表格模型，也不是已验证的召回或生成准确率提升。

## 来源结果与证据

- **漏识别的数据行不再被并进表头，且保留正确位置。** 固定提交的TestGroupBoxesByRC_MissingRowStaysInYOrder构造两层表头、深圳、缺R的成都、武汉，断言得到5个逻辑行，成都及其数值留在第4行；这是代码中的确定性回归断言，不是本次实际运行结果。 [原始来源](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/group_boxes_test.go)

- **原始行带、跨行起点和列边界有独立边界覆盖。** table_layout_test.go检查未匹配R=-1、第二物理行不被扩展首行吞掉、宽跨列表头不吸走第三列、跨行文字移至顶部物理行，以及真实PDF坐标夹具中的2层表头加1数据行。deepdoc_table_builder_test.go另覆盖仅0.2px重叠不算表头、前一个rowspan不破坏后续colspan。 [原始来源](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/table_layout_test.go)

- **作者报告真实PDF复解析恢复15行、10列、两层表头及成都行。** PR说明报告bash build.sh --test全单测通过、bash build.sh --go构建成功，并复解析一个15行/10列合并表头PDF得到正确rowspan/colspan。此为提交者报告；本次未获得独立跑出的输出，也没有多数据集准确率或延迟评测。 [原始来源](https://github.com/infiniflow/ragflow/pull/22073)

- **修复不只是主分支代码，已进入v1.0.0正式源码。** PR于11月6日10:03 UTC合并，正式release于12:31:31 UTC发布且列出#22073；GitHub compare显示tag相对修复提交ahead 7、behind 0，merge base就是该修复提交。该证据确认源码包含关系，不等于本次验证了二进制或用户部署镜像。 [原始来源](https://github.com/infiniflow/ragflow/compare/8ba216b1c198d90dc82b298354b750d2d0e9dbda...v1.0.0)

## 局限与未知
- 这是Go版DeepDoc的表格重建修复，不应宣称所有PDF解析器、Office路径或第三方OCR服务同时获得相同收益。
- 没有新OCR/TSR训练或模型质量实验。文字本身识别错、表格区域漏检、完全缺失文字框等问题仍可能存在；几何重建不会凭空恢复未知内容。
- 原始行带缺失时保留的AnnotateTableBoxes回退仍可能过度合并跨行表头；生产入口传入原始行带，但不能据此保证所有输入都有有效结构输出。
- 固定提交的parity_lock.go仍明确记录13_crosspage_table.pdf首行Go为3列、Python为5列，结构相似度98.8%，该样本不在完全一致性锁定集。不能把这次修复表述为跨页表格问题全部解决。
- 作者的真实PDF报告仅为一个具体样本，新增单测多为合成框或坐标回放；没有可据此推广的表格准确率、RAG Recall/NDCG、回答正确率或性能提升比例。
- 已核实v1.0.0源码包含关系；未下载运行发行二进制、容器、模型或测试，也未核查用户部署版本。

## 工程建议（未经实测）
- 最小实验第一步：在隔离测试目录保留修复前提交3a6b6dca78a07356c5a0ebc4ec5debfe93ac1698和修复提交8ba216b1c198d90dc82b298354b750d2d0e9dbda，保持相同模型、输入和配置；先用仓库的框级回归测试，不必先把生产知识库重建。
- 最小实验夹具至少包含：漏R的中间数据行、两层合并表头、首列跨行与后续跨列并存、同一单元格多行文字、没有任何标注的2×2零值框。分别检查逻辑行数/顺序、列归属、文字不丢不串、跨度和Y/X回退。
- 第二步只用自有或获授权的PDF，人工给出目标表格单元格真值；保留原始页与解析输出，比较修复前后结构、内容归属及跨页边界。不要只数OCR文字或只看生成回答是否流畅。
- 第三步再做RAG小样本：固定分块、嵌入、检索和回答模型，问合并表头下某一行某一列的值并核验引用。解析收益、召回变化与生成变化要分开记录；这只是建议，本次没有执行。
- 正式版源码包含修复不代表线上已升级；先确认实际构建或镜像来源，再决定回归与部署，不把本条作为未经测试的全量升级建议。

## 阅读范围
对照2026-10-09至2026-11-06共29期历史条目去重；已读PR正文、固定提交中的关键实现差异、生产入口全文、上述单测和已知差异说明，核对正式release及提交祖先关系。未执行测试、解析、训练或推理。 核查时间：2026-11-07T00:06:27Z。

## 原始来源
- [PR #22073：问题、修复范围与作者测试说明](https://github.com/infiniflow/ragflow/pull/22073)
- [11月6日固定合并提交](https://github.com/infiniflow/ragflow/commit/8ba216b1c198d90dc82b298354b750d2d0e9dbda)
- [生产PDF表格处理入口](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table_extract.go)
- [原始行带标注与跨行起点处理](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/table_layout.go)
- [漏识别行排序与分组实现](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/group_boxes.go)
- [漏行、原始零值框与多行单元格测试](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/group_boxes_test.go)
- [合并表头、跨行起点和列边界测试](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/table_layout_test.go)
- [表头覆盖与原始中心点测试](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/table/deepdoc_table_builder_test.go)
- [已知跨页表格结构差异](https://github.com/infiniflow/ragflow/blob/8ba216b1c198d90dc82b298354b750d2d0e9dbda/internal/deepdoc/parser/pdf/parity_lock.go)
- [RAGFlow v1.0.0正式发布记录](https://github.com/infiniflow/ragflow/releases/tag/v1.0.0)
- [修复提交到正式tag的祖先关系](https://github.com/infiniflow/ragflow/compare/8ba216b1c198d90dc82b298354b750d2d0e9dbda...v1.0.0)
