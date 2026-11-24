---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-24
verified_at: "2026-11-24T00:04:59.620Z"
---

# STFR：用多视角地标给 FLAME 一个更可靠的起点

## 问题

手机绕拍可以重建一张脸的局部细节，但自动重拓扑若从不稳定的地标或拟合起点出发，在眼睑、嘴角或轮廓可能留下返工。11月18日增量是注册阶段的初始化与稳健拟合，不是新的2DGS或贴图模型。

## 输入

已授权的手机绕拍视频。比较更新本身时固定视频帧、掩膜、COLMAP相机、2DGS网格、初始外观及注册模板。运行需Linux/CUDA栈，DAViD、面部地标权重、FLAME 2020 `generic_model.pkl`，完整注册另需 Faceform WrapCmd。

## 输出

详细网格与相机参数，以及注册后的 ICT 拓扑 OBJ、独立 UV 图；可另输出 UE5 规范坐标 OBJ 和变换。不是 MetaHuman DNA、角色绑定、口型模型或自动重光照材质。

## 流程

- **固定几何底稿**：默认先从视频取帧并估计人像掩膜，COLMAP估相机，2DGS输出几何。选16张较清楚的视图拟合固定几何的外观，用于后续地标渲染。
- **三角化地标、再稳健拟合**：共享标定相机与地标观测，先三角化成3D地标并初始化FLAME；退化几何、无穷点等拒绝，代码要求至少三个三角化地标。三角化的soft-L1与后续稳健重投影采用4px量级尺度，减少异常观测权重；这不保证每种遮挡都能稳定。
- **从正面重投影，生成 ICT 拓扑**：拟合将几何带到规范空间，再用正面渲染识别地标并投回3D；WrapCmd执行模板配准。注册之后才固定网格优化UV纹理，并导出OBJ及贴图。
- **把颜色问题留给第二组**：若目标是去阴影，冻结通过验收的网格和UV求解器；另测试逐帧去光照处理后的RGB，不用换几何证明颜色改善。

## 关键机制

同一位置由多视角观测约束，稳健重投影比直接让少数异常点牵引参数更适合估计初始化。重拓扑由外部Wrap执行，UV仍是对拍摄外观的拟合；其许可、输出与精度应分别判断。

## 作者结果与可核对事实

- **注册更新已在作者主分支**：固定提交7ef2d358于2026-11-18T01:56:06Z纳入三角化初始化和稳健拟合；有真实源码，不是模型权重首发。 [来源](https://github.com/yxuhan/STFR/commit/7ef2d358b2a8047b4519f8b389783bce0e091ca4)
- **作者示例速度不是返工测量**：作者给80帧、长边960的示例，单RTX 3090约21分钟。不是本轮实测，也没有新版相对旧版的人工修模时间、失败率或Mac性能数据。 [来源](https://github.com/yxuhan/STFR/blob/7ef2d358b2a8047b4519f8b389783bce0e091ca4/README.md)
- **纹理有单独基线**：旧 OpenDelight 的公开 runner 输出逐帧 RGBA diffuse，不直接输出融合 UV。其论文表1中，3DRFE PSNR 34.27 vs SwitchLight 31.08 dB，而 FaceOLAT 为32.88 vs 32.98；指标逐图匹配色彩且仅计皮肤，不证明颜色跨视角一致，也不证明无需人工。 [来源](https://arxiv.org/html/2605.05636v1)

## 边界与未知

- GPL-3.0 源码不覆盖第三方组件。WrapCmd需要有效许可证，FLAME 2020与各权重有各自条款；不要把源码开放写成整条链可自由商用。
- 眼部模板有close/open/fill分支；同一 A/B 固定模板。无兼容平台或许可证时可以只核验前段或离线资产，不把跳过注册写成通过。
- 人物要保持表情，遮挡、光亮过曝、少视角和低分辨率可能让纹理或地标异常。是面部路线，不承诺完整头部、侧后发型或极端卡通。

## 工程建议（未实测）

- 同一授权视频固定帧/相机/掩膜/网格/模板，比更新前后注册；第一次可只跑 reg 阶段，记录鼻翼/眼睑/唇线位置、模板穿插、一次通过率和盲评修模分钟数。画质与用时都要过门槛才扩样。
- 再固定几何、相机和UV算法，比较原始与去光照贴图；使用未参与该次拟合的视图检查暗影残留、肤色漂移和雀斑保留。两组数据分别报告。

## 阅读范围

已核对固定补丁、稳健拟合实现、版本化使用/环境/输出和独立注册脚本、Faceform 文档；OpenDelight 只用于对照纹理，核了公开推理和论文对比。未安装、下载权重或运行。

## 原始来源

- [11月18日官方补丁](https://github.com/yxuhan/STFR/commit/7ef2d358b2a8047b4519f8b389783bce0e091ca4)
- [固定版本 FLAME 拟合代码](https://github.com/yxuhan/STFR/blob/7ef2d358b2a8047b4519f8b389783bce0e091ca4/registration/align/AlbedoMMFitting/robust_fitting.py)
- [固定版本项目使用说明](https://github.com/yxuhan/STFR/blob/7ef2d358b2a8047b4519f8b389783bce0e091ca4/README.md)
- [阶段、导出和运行配置](https://github.com/yxuhan/STFR/blob/7ef2d358b2a8047b4519f8b389783bce0e091ca4/pipeline.py)
- [依赖与单独模型文件](https://github.com/yxuhan/STFR/blob/7ef2d358b2a8047b4519f8b389783bce0e091ca4/ENV.md)
- [Faceform 命令行许可](https://faceform.com/docs/Wrap/CommandLineInterface/CommandLineInterface.html)
- [OpenDelight 公开推理](https://github.com/yxuhan/OpenDelight/blob/main/doc/TEST.md)
- [OpenDelight 论文及表1](https://arxiv.org/html/2605.05636v1)
