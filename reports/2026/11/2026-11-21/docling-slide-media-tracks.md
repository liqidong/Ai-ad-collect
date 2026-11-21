---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-21
verified_at: "2026-11-21T00:02:37Z"
---

# Docling：PPTX/ODP 内嵌音视频可保留为轨道并转写

## 问题

只读取幻灯片文本、图片或视频封面，无法把视频讲解、语音备注作为可定位的知识来源。前一期 Nemotron MLX 是页面解析引擎，此次增量处理的是演示文稿内部挂载的媒体。

## 输入

PPTX/ODP 演示文稿（含可识别的视频/音频封面、图标和媒体引用）；可选转换需 ASR 依赖和模型，视频还依赖 ffmpeg。合并源码将 docling-core 下限升为 2.108.0。

## 输出

默认保留媒体轨道与封面/引用，不默认转写。显式转写后在轨道中得到带时间戳的文字、视频采样帧及所属幻灯片位置；可选保留媒体字节进 DCLX，普通 JSON 仅保留引用。

## 流程

- **识别轨道及其来源**：PPTX/ODP 后端把可识别的媒体记录为 TrackItem 和 MediaRef，再标记封面、所属幻灯片及媒体内部路径或链接。不可识别容器可只剩封面。
- **按需加载或转写**：只有显式启用 keep_media 或 do_media_conversion 才进入媒体处理；默认不加载 ASR 模型。取内部媒体或按显式规则处理链接；同文档内按媒体位置去重加载和转换。音频交 ASR，视频交视频管线，未改变原有默认视频采样策略。
- **保留溯源与错误**：把结果挂到各自的幻灯片轨道，按轨道复制幻灯片位置及内容层，保留原时间戳。真实转换失败记录错误并保留轨道；规则不允许的外链、某些未支持转写的旧视频格式可能只是跳过，不应只用最终 success 来判断媒体是否都已转写。

## 关键机制

轨道把时间维度和页面维度同时保留下来。DCLX 可装媒体字节，普通 JSON 不自包含。使用最终代码与固定文档为准：合并版已有 --convert-media，而旧 PR 描述说尚无 CLI 且仍使用图片元数据，不能作为最终行为。

## 作者结果与可核对事实

- **幻灯片溯源与去重有覆盖**：合并测试使用合成演示文稿与桩 ASR，断言流转后的幻灯片页码/坐标、不可见内容保持、同一媒体两页只转换一次、DCLX归档保留原始字节、普通JSON不含字节。 [来源](https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/tests/test_simple_pipeline_media.py)
- **真实模型测试定义覆盖含声与无声视频**：带模型标记的测试使用约10秒 MP4/MP3，检查非空转写、时间戳、页码及视频采样图；无声视频只应产出帧。测试可在缺 ffmpeg 时跳过；本轮读源码未运行，不宣称 CI 已通过或准确率提升。 [来源](https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/tests/test_simple_pipeline_media_asr.py)
- **主分支合并尚非发行确认**：2026-11-20T19:09:46Z 合入，SHA 5182e556；本轮核到的最新 Docling 发布 v2.151.1 在 11 月 19 日，早于合并。 [来源](https://github.com/docling-project/docling/pull/4752)

## 边界与未知

- 不是通用视频语义或说话人识别评测；没有转写准确率、长演示文稿耗时、端侧峰值内存实测。
- 旧格式可能只能保留引用或原素材，不能转写或取帧；含音轨与否也影响输出。
- 远程、本地外链分别需显式选项，绝对本地链接不会加载；处理不受信材料时宜保持外链关闭，不将转换成功或轨道存在理解为内容完整。

## 工程建议（未实测）

- 最小验证：在含该提交的隔离环境，用公开或合成 PPTX（一个内嵌短音频和短视频）先不开任何选项确认轨道，再加 --convert-media 验证时间戳与页码。只有需要自包含素材时另加 --keep-media 并输出 DCLX。
- 知识库索引建议按页码、轨道ID和时间段做引用，并单列缺失或未转换媒体。发布前确认转写文本、媒体原件及其保留授权，避免将原始媒体误公开。

## 阅读范围

已核原始 PR、最终处理链代码与文档 diff、CLI 数据约定、关键桩和真实模型测试源码及发布记录。不声称逐行审计44个变更文件；未安装或运行。

## 原始来源

- [原始PR与合并记录](https://github.com/docling-project/docling/pull/4752)
- [固定版使用文档](https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docs/usage/processing_audio_media.md#video-and-audio-in-slides)
- [媒体转换与溯源实现](https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/pipeline/simple_pipeline.py)
- [离线桩测试](https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/tests/test_simple_pipeline_media.py)
- [真实模型测试源码](https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/tests/test_simple_pipeline_media_asr.py)
- [Docling发行记录](https://github.com/docling-project/docling/releases)
