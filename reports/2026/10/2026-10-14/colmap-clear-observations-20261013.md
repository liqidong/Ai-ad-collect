---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-14
---

# COLMAP：修复清点重三角化的残留观测计数

来源日期：2026-10-13
核查时点：2026-10-14T00:12:19Z（UTC）
状态：公开来源研读草稿，未独立复现实验。

## 核心问题
旧版在缩小Image的2D观测数组时直接resize，没有先移除被截断观测的2D–3D关联；Image::NumPoints3D计数因此残留。清空重建后虽然3D点数为0，汇总观测数仍可能为1000、IsValid为false；随后重建track可能触发NumPoints3D<=NumPoints2D检查。作者指出DeleteAllPoints2DAndPoints3D被point_triangulator的clear_points路径及global mapper重三角化使用。

## 输入
已含2D–3D关联的COLMAP reconstruction，随后调用删除所有2D/3D点或缩短Image的2D点数组；也包括反复清点、再建立tracks的工作流。对于GS渲染图的已知相机链，关联点是清空旧观测后重新三角化，而不是渲染GS本身。

## 输出
清空后的计数与真实状态一致，缩小时保留范围内的观测关联不丢失，后续可以重新建立track而不受旧计数污染。补丁本身不生成更高精度Mesh，不改变相机估计、MVS深度代价或人脸轮廓目标。

## 分步流程

### 1. 判断是否命中触发路径
先检查当前脚本是否对已有模型使用clear_points、delete_all_points2D_and_points3D或SetPoints2D缩小数组。如果每轮从全新reconstruction开始且不清点复用，这项对当前任务的优先级较低。

### 2. 缩小前逐项解除被移除的关联
6db3867f在SetPoints2D(vector<Eigen::Vector2d>)的resize之前，遍历[new_size, old_size)索引并调用ResetPoint3DForPoint2D。已有3D关联被置为invalid，同时num_points3D减1；原本无关联的点不改变计数。

### 3. 保留仍在数组内的关联，再调整坐标
resize后仍沿用原来的xy赋值循环。被保留索引的3D关联保持，新增索引不带3D关联；扩容和同长度赋值不会进入新增的解除循环。此补丁不是任意点重排的track重映射器。

### 4. 检查清点、重复清点与重建tracks
上游测试在全清点后检查每张图NumPoints2D/NumPoints3D和全局观测数均0、模型有效；再在两个注册图像上各建1个观测并添加共同3D点，应得到1个3D点、2条观测、模型仍有效。

### 5. 回到原摄影测量基线
若命中故障，以包含提交的构建跑同一个小样本清点/重三角化，再继续原MVS与Mesh流程。先确认任务能正确执行，不把异常消除写为几何精度提升。

## 关键机制
真正修复点在Image层的关联计数维护，而不是单独把DeleteAllPoints2DAndPoints3D的总计数硬设零。这样既覆盖完全清空，也覆盖只缩短一部分观测的情况；ResetPoint3DForPoint2D依据实际HasPoint3D决定是否递减，避免重复清空发生计数下溢。四行production改动配合image和reconstruction回归用例，使“持有关联的2D观测数”重新与Image的缓存计数一致。

## 来源结果与证据

### 补丁在观察窗内正式合入main。
#4853创建于2026-10-13T03:34:28Z，2026-10-13T14:08:57Z合入main，merge_commit为6db3867f；3个文件、50行新增，其中production修改位于image.cc。维护者于13:14:27Z给出APPROVED。
来源：https://github.com/colmap/colmap/pull/4853

### 错误可在合成重建的清点路径直接体现。
作者pycolmap最小复现：synthesize_dataset后delete_all_points2D_and_points3D，再读num_points3D/compute_num_observations/is_valid，修复前为0/1000/False，期望0/0/True。旧基线重建track触发NumPoints3D<=NumPoints2D，示例为51 vs. 1。
来源：https://github.com/colmap/colmap/pull/4853

### 上游报告104项聚焦测试通过，并对旧版确认回归测试会失败。
作者在head29c18888、WSL Ubuntu26.04、GCC15、Release、CUDA/GUI/ONNX关闭条件下，image/reconstruction/database_cache/reconstruction_io四套测试通过，共21+51+9+23=104项。仅把两组回归改动加到旧main08c03ec3会失败。clang-format23.1.3与git diff --check通过。
来源：https://github.com/colmap/colmap/pull/4853

### 新测试同时覆盖部分缩小和复用状态。
Image测试从4个2D点、3个3D关联缩到2点，计数应为1且索引0关联保留；随后扩至5点，再清空两次，最终2D与3D关联计数均0。Reconstruction测试覆盖清空有效性和两视图新track的建立。
来源：https://github.com/colmap/colmap/commit/6db3867f60771f34fa66d166dcc00fee7d5c6dc0

## 局限与未知
- 测试数字来自上游作者报告；本次仅阅读原始代码、PR和测试，没有编译、执行或在用户的GS渲染图上复现。
- 上游明确没有跑完整套件或完整重建pipeline；聚焦CPU测试不能替代pycolmap扩展、跨平台、CUDA或人脸数据端到端验证。
- 只对清空/缩小已有观测关联数组的状态维护有直接作用；不修复相机坐标约定、图像名不一致、匹配失败、稠密深度噪声或轮廓收缩。
- 合入main并不说明用户当前COLMAP/pycolmap稳定包已包含修复，需要查实际构建提交。
- 这不是上一期images.txt空格名称或角度边界修复的重复，也不是此前模型重读状态/crop误差保留更新；触发动作与状态变量均不同。

## 工程建议（非复现结论）
- 先定位有无clear_points/清点复用，再决定升级优先级。单独这项bug不值得在未触发条件时重跑全套人脸Mesh实验。
- 建议小样本断言清点后num_points3D=0、compute_num_observations=0、is_valid=true；再重三角化检查有效track计数与每图2D/3D计数关系。这里只是建议测试，不声称已经执行。
- 做版本对照时固定图像、数据库、相机与渲染配置。若旧版异常、新版可完成，可报告正确性恢复；几何精度须另用扫描真值、轮廓偏移和局部ROI检验。
- 不要把SetPoints2D的保留索引语义误用为任意重排或删除中间点的安全接口；这个补丁只处理尾部截断所移除的关联。

## 实际阅读范围
已完整阅读正式提交的3文件diff、PR描述、合入元数据及维护者评审，并阅读正式image.cc中SetPoints2D与ResetPoint3DForPoint2D实现。未本地执行代码或测试。本项为代码工程核查，不是论文全文细读。

## 原始来源
- [COLMAP #4853：清空/缩小观测后保持计数一致，已合入main](https://github.com/colmap/colmap/pull/4853)
- [正式提交6db3867f：production与两组回归测试](https://github.com/colmap/colmap/commit/6db3867f60771f34fa66d166dcc00fee7d5c6dc0)
- [Image::SetPoints2D与ResetPoint3DForPoint2D正式实现](https://github.com/colmap/colmap/blob/6db3867f60771f34fa66d166dcc00fee7d5c6dc0/src/colmap/scene/image.cc)
