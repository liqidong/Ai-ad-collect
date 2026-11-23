---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-23
verified_at: "2026-11-22T23:59:59.187Z"
---

# Disney分层高斯：从稀疏卡通参考图做可改、可绑定的角色

## 问题

卡通设定图可能跨视角不一致，而直接从图生成视频难以保留稳定、可编辑的角色资产。

## 输入

单张或多张角色图，可选人工网格；如需线稿与光照分层，作者方法需要有/无对应效果的参考。

## 输出

可绑定的代理网格和分层外观高斯，支持指定视角和姿态渲染、单独编辑样式；不是已验证的精密头部Mesh、口型或语音驱动。

## 流程

- **先建可控代理**：从 TRELLIS 或人工网格起步，高斯以重心坐标锚到三角面，中心、朝向、尺度随三角面形变；论文实现用表面一层及沿法线的内外各三层。可用 Mixamo 或 RigAnything 给网格绑骨骼。
- **补视图，而非把生成图当真值**：以初拟网格渲染的转台深度和原参考联合条件化 LTX-Video，再以合成角度作伪监督。阶段 I 联合拟合网格与底色；优化时约束高斯的不透明度并固定内禀尺度，促使光度误差反推网格形变，而不单靠高斯遮住误差。
- **固定几何后分层编辑**：内层/表层保留底色；外层承担纹理叠加和轮廓。用成对参考拟合视角相关的线稿或光照模块，允许逐层人工修订。 阶段 II 不再移动网格；各样式层可单独改、再合成。

## 关键机制

关键组合是共用一个可变形骨架代理，把低可信的生成监督放在拟合阶段，把线稿/光照拆出底色。不是单张图直接输出最终动画。

## 作者结果与可核对事实

- **四个合成角色、每个四张输入，报告的转台图像指标更好**：表4：PSNR 23.8219 vs LTX 22.5696，LPIPS 0.0337 vs 0.0405。正文称对真值转台图定量，但未明确这些数字是否剔除输入视角，不写成严格留出新视角成绩。图7则明确区分展示的输入图与生成图，三组分别用3、4、1张输入；两项证据不是同一个协议。 指标是完整方法对比，并非只加合成视图的消融。 [来源](https://studios.disneyresearch.com/app/uploads/2026/11/Designing-Animatable-Stylized-Characters-with-Multi-Layer-3D-Gaussians-Paper.pdf#page=8)
- **预览和最终后处理速度不同**：表7作者单RTX 4090实验：准备25–60分钟；无Difix 0.05秒/帧，有Difix 2.5秒/帧。不要把预览速度写成最终输出速度。 [来源](https://studios.disneyresearch.com/app/uploads/2026/11/Designing-Animatable-Stylized-Characters-with-Multi-Layer-3D-Gaussians-Paper.pdf#page=8)
- **人工修订能纠正单图初版，但不是自动无误**：图11显示骑士与三角龙：单张分层图、初版、画师修正、更新结果。流程测试仅两名画师，不能外推为普遍提速结论。 [来源](https://studios.disneyresearch.com/app/uploads/2026/11/Designing-Animatable-Stylized-Characters-with-Multi-Layer-3D-Gaussians-Paper.pdf#page=12)

## 边界与未知

- 论文以可控渲染为目标，不保证代理网格拓扑正确；图10报告复杂遮挡及插值伪影。强冲突参考与偏离视频模型训练分布的极端风格仍会失败。
- Disney官网只有论文与演示入口；定向未找到官方代码、权重、训练数据或相关软件许可。论文版权许可不等于工程许可。
- 未独立运行；不将案例外推到嘴部遮挡、中文口型、多视角头部精确几何或Mac实时。

## 工程建议（未实测）

- 工程建议：先用一位有授权的风格化角色做对照。固定三个原始观察视角，遮挡处人工标记；仅把LTX补的角度作为低权重候选，留一张真实图评估身份和轮廓。
- 优先把头/脸/头发/线稿分开评估。多层脸部或发际遮挡重叠时，先手工建立掩膜或拆层再重拟合，不让生成补图替代几何约束。这是推荐兜底，不是论文已提供自动分割。
- 不急着替换人物驱动链。先用同一代理比较「原图拟合 / 加生成视图 / 加一次人工修订」并记录几何、底色、换角度和动作下的失真。

## 阅读范围

已读官方12页PDF方法、消融、评价、讨论与实验表，目视核对第12页图7/10/11；核对发布页、议程及最近七份公开归档未见该论文条目。未复现。

## 原始来源

- [Disney发布页](https://studios.disneyresearch.com/2026/11/16/designing-animatable-stylized-characters-with-multi-layer-3d-gaussians/)
- [论文PDF](https://studios.disneyresearch.com/app/uploads/2026/11/Designing-Animatable-Stylized-Characters-with-Multi-Layer-3D-Gaussians-Paper.pdf)
- [SIGGRAPH Asia官方议程](https://sa2026.conference-schedule.org/presentation/?id=papers_1786&sess=sess150)
- [ACM DOI](https://doi.org/10.1145/3829340.3842267)
- [图7、10、11：对照、失败和人工改图](https://studios.disneyresearch.com/app/uploads/2026/11/Designing-Animatable-Stylized-Characters-with-Multi-Layer-3D-Gaussians-Paper.pdf#page=12)
