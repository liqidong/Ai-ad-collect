---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-03
---

# AB-Attention 首次开放代码：加速视频扩散采样，不等于改善高斯训练数据质量

来源日期：论文：2026-10-29；代码：2026-11-02T18:47:34Z（上海11月3日02:47）

核查时间：2026-11-03T00:03:13Z

状态：首次代码开放｜非论文今日首发

## 核心问题

视频扩散把时空 token 两两计算注意力，序列变长时成本近似平方增长。粗略挑选重要块容易丢掉低权重但累计显著的长尾信息。AB-Attention 解决的是推理成本与近似误差的权衡；它不是人物身份保持模型、姿态控制器，也不直接训练 Gaussian。相对上一期只见待开放说明，本次实际增量是 11 月 2 日的首个可检查代码提交，论文仍是 10 月 29 日版本。

## 训练与推理输入

本方法不新增训练过程或专属权重。视频入口读取已有 Wan2.1 T2V 1.3B/14B 或 HunyuanVideo 1.0 13B 权重、提示词、尺寸/帧数、随机种子或 float32 初始 latent。底层输入是同设备 BF16 Q/K/V，形状 [B,H,N,128]，N 必须能被 4 整除，采样 token 数小于 N。代码接入已有模型的注意力，不接收 Gaussian 模型，也不接收带相机/SMPL-X 标注的训练集。

## 输出与资产形式

底层返回与 Q 同形状的近似注意力结果，可选 softmax 归一化量；上层输出生成视频 MP4。没有输出多视角同步标定、SMPL-X、mask、可驱动高斯或新动作泛化保证。

## 分步流程

### 1. 固定模型与稠密对照

generate.py 载入 Diffusers 模型及调度器，准备提示词和同一份初始噪声；安装 Wan 或 Hunyuan 的注意力处理器。--dense 保留相同前处理而用完整 FA3，输出可比较的基线视频。每次新视频重置采样计划，不能跨样本继承上一个人物的计划。

### 2. 按步骤决定是否近似

video.py 的前 10 个去噪步及每步第 0 层保持稠密。从第 11 步开始，其余层每 4 步重建计划，中间步骤复用索引与权重而非复用旧注意力输出。输出是当前层应使用的稠密路径或采样计划。

### 3. 局部聚类给细块评分

输入当前 Q、K，IBCS 在每个 128-query 块内用 8 个代表簇估计其关注方向，对 4-token key 块打分。这样不是用一个平均 query 代表全部需求，也不需要对整个序列做全局重排。输出每个 query 块对候选 key 块的分数。

### 4. 概率采样并记录校正

SIP 把分数转成包含概率，不放回抽取固定数量的块；高分块可能必选，低分块仍有非零机会。build_plan 输出块 ID 及 log2(1/概率) 权重，按 [rows,pages,32] 保存。每个 page 收容 32 个四 token 块。

### 5. 分页执行并回到去噪流程

flashinfer_backend.py 将选中块映射为分页 KV 地址，并在 logits 上加入校正量，再执行 softmax 和 V 聚合。视频/文字交互在 Hunyuan 路径保持精确；采样主要用于视频对视频。结果进入剩余 transformer 与 VAE，最终导出视频。

## 关键机制

核心不是把大量注意力直接置零，而是用带逆包含概率权重的样本近似完整注意力。需要严格区分：对固定激活，分子与分母估计无偏，不代表两者相除后的注意力输出无偏；论文明确承认这个比值通常有偏。细粒度选择与大 tile 执行分离，避免因仅选择 4-token 块就让 GPU 逐小块低效运行。公开实现通过 FlashInfer FA3 分页后端完成执行，本项目提供选择/采样及权重适配。

## 来源事实与报告效果

- 作者表 1：在视频生成质量接近稠密参照的条件下加速，未证明生成数据能改善数字人训练。：600 个 Penguin 提示词，720p、50 步，H20/BF16；Wan1.3B、Wan14B、Hunyuan13B 的端到端加速分别 2.01×、1.84×、2.23×。后两者 PSNR 为 30.788/34.347，SSIM 0.9169/0.9480，LPIPS 0.0964/0.0700；PSNR/SSIM 越高、LPIPS 越低表示更接近同提示词/种子的稠密生成，不是真实人体重建精度。表内对比包括 Radial、DFSAttn、SVG2。 [原始证据](https://arxiv.org/html/2610.39663v1#S4.T1)
- 作者消融支持概率校正的重要性，而不是只要随机删块就有效。：表 2 使用 Wan14B 的 100 提示词子集，相同 89.16% 稀疏度；完整法 PSNR 29.705，无权重采样为 25.706，top-k 为 25.711。表 3 固定采样预算下，4-token 与128-token选择分别为29.705与28.686。子集结果不能与表1的600条均值直接相减。 [原始证据](https://arxiv.org/html/2610.39663v1#S4.T2)
- 首发代码已提供自定义输入与匹配基线入口；可检查不等于本轮已跑通。：generate.py 支持 --prompt、--negative-prompt、--model-path、--latents、--dense；计时覆盖文本编码/去噪/VAE，不包含模型载入和视频导出。README 明示小尺寸功能测试不是论文速度/质量复现。 [原始证据](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/generate.py)

## 限制与未知

- 2026-11-03 核查：仓库 About 仍写 Code coming soon，但固定提交已存在 README、CUDA 源码与 Python 入口；以实际文件及提交时间判断开放，不把陈旧简介作为当前状态。
- 只支持前向推理；不能用作视频模型微调反向算子或高斯训练加速。当前编译目标 sm_90a，文档面向 Hopper H20/H100；没有验证消费级 RTX 或其他架构可用。
- 公开模型集成是三种 T2V 模型，不是 Wan2.2、MimicMotion、姿态条件 I2V 的直接兼容证明。迁移需要独立接入并核查 token 布局、文本分区和 scheduler。
- 作者的注意力跳过比例只统计稀疏去噪步，不是整条管线减少 89–92% 成本；端到端提升远小于该比例，不能混为一谈。
- 表1 VBench 并非全面超过稠密版，例如 Wan1.3B imaging quality 63.92 对稠密65.48；近似会改变结果，不是无损提速。
- 没有人体身份、手指、遮挡边界、跨视角几何一致性或下游 Gaussian 未见动作指标。生成得更快不能推导出监督更可靠。
- 代码 MIT；底模、SVG2、FlashInfer 和模型权重各自许可仍需分别确认。Hunyuan默认下载使用社区 Diffusers 转换，复现应固定转换版本。
- 本轮未安装依赖、编译内核、下载权重或跑实验；无独立复现和生产可靠性结论。

## 工程建议，非复现实验

- 最小试用（建议，未执行）：按固定提交 README 在隔离 Linux/Hopper 环境安装 PyTorch2.6/CUDA12.4、视频依赖及指定 SVG2 revision；先运行 python generate.py --model wan1.3b --prompt 'A person raises one arm.' --height 256 --width 256 --frames 17 --steps 15 --sample-pages 1 --output smoke.mp4，再加 --dense 比较同噪声结果。此例仅验证路径，不代表可控动作或质量验收。
- 如目标是生成训练数据，先决定身份和动作/相机控制生成器，再评估能否移植该注意力算子；不能直接用自由 T2V 输出替代同角色、多视角一致监督。必须先有原始数据基线与伪帧筛选。
- 工程判断：最可能复用的是 build_plan/apply_plan、每层计划刷新与稠密 warmup。新增成本包括 Hopper 编译环境、模型 attention 接口适配、固定噪声公平对照和错误回退；不是装一个包就直接接入任意 avatar 训练。
- 建议双层验收：先比较生成端时延/显存/身份与动作漂移，再固定高斯训练预算比较纯真实数据、稠密生成数据、稀疏生成数据三组。最终以关闭生成器后的新视角/新动作渲染判断收益，而非展示修饰过的视频。
- 可选 --cube-layout 是首发代码中的额外布局选项；README 说明论文表格使用原 raster 顺序。先不启用，避免把新增扩展效果冒充论文证据。

## 阅读范围

已读 arXiv v1 HTML 的正文第1–6节、方法公式/算法、实验表1–3、局限与附录A–D；核对固定首发提交 README、attention.py、video.py、flashinfer_backend.py、generate.py、setup.py、pyproject.toml、bindings.cpp 与 LICENSE。没有逐行审计全部 CUDA 内核、下载视频逐帧检查或运行代码。论文日期2026-10-29；代码提交2026-11-02T18:47:34Z；核查2026-11-03T00:00:47Z。

## 原始来源

- [AB-Attention 论文 v1（正文、三张表及附录 A–D）](https://arxiv.org/html/2610.39663v1)
- [arXiv 提交日期：2026-10-29](https://arxiv.org/abs/2610.39663)
- [首次代码提交：2026-11-02 18:47:34 UTC](https://github.com/InfRec-Cardinal/AB-Attention/commit/75e01af44ba850e4ee62375ead12d901a53cb483)
- [固定提交 README：安装、模型、复现条件](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/README.md)
- [采样计划 API](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/ab_attention/attention.py)
- [视频模型接入与稠密/稀疏调度](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/ab_attention/video.py)
- [FlashInfer 分页执行和输入约束](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/ab_attention/flashinfer_backend.py)
- [自定义提示词/模型目录/初始噪声运行入口](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/generate.py)
- [依赖及 CUDA 编译目标](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/pyproject.toml)
- [MIT 许可](https://github.com/InfRec-Cardinal/AB-Attention/blob/75e01af44ba850e4ee62375ead12d901a53cb483/LICENSE)
