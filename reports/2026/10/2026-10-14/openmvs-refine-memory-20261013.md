---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-10-14
---

# OpenMVS：精化降内存与贴图更新已合入，静态人脸先关自动分块

来源日期：2026-10-13
核查时点：2026-10-14T00:12:19Z（UTC）
状态：公开来源研读草稿，未独立复现实验。

## 核心问题
高分辨率多视图的摄影测量链在 RefineMesh 中保留每视图梯度/深度缓冲，并为每个 CPU 图像对建立多张全幅统计图，容易让视图数与分辨率放大内存压力。TextureMesh 又以两倍图像宽高光栅化、旧梯度评分选择贴图，增加内存与遮挡边缘污染风险。10月13日合入的 #1314 把这些工程改动、按内存分块、PLY/OBJ I/O 与 halfmesh 迁移一并带入 develop；它不是新的静态人脸精度算法。

## 输入
沿用同一组 GS 渲染图、固定且已校验坐标/尺度的相机、同一份 MVS scene.mvs、深度图或初始网格。精化对照需要复用完全相同的初始 Mesh 与深度图；仅让新旧版本分别重跑深度估计不足以隔离改动。依赖 halfmesh v0.5.1，项目端口已固定下载哈希。

## 输出
按光度一致性精化后的几何网格、重新分配/排布贴图的纹理 Mesh，或按预算分块后合并的 Mesh/点云。内存优化主要改善可运行规模；贴图 PSNR/SSIM反映表观，不能证明鼻翼、眼睑、唇缘或脸部外轮廓更准确。分块合并不会缝合接缝。

## 分步流程

### 1. 锁定版本与不分块基线
#1314 在2026-10-13T22:12:03Z合入 develop，提交610562a4；仍不能视作稳定版。MvgMvsPipeline.py 的新默认是 --sub-scene-memory auto；首轮使用 --sub-scene-memory off，若直接用 MvsScalablePipeline.py 则是 --memory off。分开输入/输出目录，避免 --resume 复用旧产物误判版本收益。

### 2. 在采样位置取梯度，缩小 CPU 窗口缓存
默认中央差分与双线性采样可交换顺序：在投影采样点左右/上下各一像素采样再求差，不再为每视图存两张 float 梯度图。其他模板模式0/2仍需预计算。CPU 的六张7×7窗口和改为64行带状缓冲加边界halo；延迟移除被门限拒绝的像素，以保持下一带的统计输入。

### 3. CUDA 按参考视图合并光度累加
同一参考视图的图像对方向先把光度项写入像素和，再对该视图拥有的面累加一次；每组最多32个方向，用64位(group,bits)标记保留顶点方向计数。另一个改动取消各视图常驻深度图：参考深度从face map每组重算到共用缓冲，配对视图在采样点按面重算。

### 4. 贴图优先可见分辨率与遮挡完整度
面片在每个视图的质量改为可见像素数，并按该面自己的最佳视图归一化。可见面积不足投影的90%且缺失超过2像素时，质量乘0.1；不到最佳值0.2的候选被剔除。使用原分辨率光栅、Potts LBP线性标签计算、footprint图集与push-pull边缘填充；unsharp mask默认权重由0.5改0。

### 5. 只有内存确实不足时再独立测试拆分
先估计全场景深度图，再按深度样本递归二分，模型分别预测fusion和Delaunay mesh峰值。每块独立融合/建网/精化，最终按面重心所属箱体保留并拼接，再用全体图像贴图。该合并没有几何缝合；人脸的轮廓、鼻唇沟和眼睑不宜承担未经验证的分块边界。

### 6. 先测几何，再单独测表观
建议保留旧版不精化、旧版精化、新版同参精化三组，以固定相机渲染的silhouette与深度误差检查几何；有独立扫描真值时再报表面距离分位数、法线误差与多阈值F-score。面部ROI单独汇总，并记录主存、显存和wall time。贴图留出视图PSNR另列，不作为Mesh精度通行证。

## 关键机制
本轮主要是减少中间量与重排等价计算，而不是增加观测证据。默认梯度的内存换取额外采样，CPU带状统计减少线程叠加的全图scratch，CUDA分组降低重复面累加并以重计算替代常驻深度。浮点求和/采样重排仍可能改变优化轨迹，不能将所有改动都称作逐字节一致。贴图视图选择确实改变了目标评分，更重视面片有效可见分辨率与遮挡，但只改变纹理来源。新分块把场景容量问题转化为独立子问题，代价是块间边界未缝合、内存预算只为两个阶段建模。

## 来源结果与证据

### 新增状态是 develop 合入，不是稳定软件发布。
PR created_at=2026-10-12T20:20:47Z，merged_at=2026-10-13T22:12:03Z；base=develop、merged=true，45条提交、52个文件。10月14日核查releases，最新正式软件发布仍为v2.4.0（2026-01-20），另有9月RoMa模型预发布。
来源：https://github.com/cdcseacave/openMVS/pull/1314

### 默认梯度按需计算减少每视图缓冲，几何F1均值变化很小且并非各场景都提升。
作者同输入CUDA测EPFL/Strecha的Herz-Jesu-P8（HJ8）、fountain-P11（F11）、Herz-Jesu-P25（HJ25）：主存1.60/2.17/3.41→1.28/1.64/2.51GB；该单项每视图显存21→13B/px。F1旧0.5174/0.3479/0.6473，新0.5186/0.3490/0.6457，平均约+0.0002，HJ25下降。该提交未给出完整硬件/这些F1的距离阈值，不能外推到人脸或毫米精度。
来源：https://github.com/cdcseacave/openMVS/commit/d219c18f334f87e9eaca3b22eeaff0032027cbe1

### CPU带状统计在HJ8约减半峰值RAM。
作者报告8.3–9.1GB→4.1–4.3GB；F1新0.5171/0.5188，旧0.5185/0.5180，属于CPU运行顺序扰动量级；wall time未超出噪声显示变化。
来源：https://github.com/cdcseacave/openMVS/commit/54779eef3927fa8224e5687cd686e9a11f96227c

### CUDA光度分组减少单次evaluation时间。
按参考视图分组，在F11/HJ25/Tanks & Temples Truck最细尺度evaluation用时降低20%/28%/52%，Truck wall 82.8→71.5s，accept/reject序列相同。结果来自逐项提交实验，不应与其他提交的计时基线混合相乘为总加速。
来源：https://github.com/cdcseacave/openMVS/commit/3393d2e0bd7585294d97bae6213d55de13e86728

### 取消CUDA常驻深度图节省显存，且本单项实验输出逐字节一致。
该独立提交在HJ8、Truck输出逐字节一致，Truck显存8.85→6.86GB；wall 66.5→68s略增加。它采用重计算换常驻缓冲，不能称此单项同时加速。
来源：https://github.com/cdcseacave/openMVS/commit/fff11c5fe2ce20531618359dbcc133860861e968

### 新贴图默认在四场景留出视图上改善表观指标。
每第4个视图留出、同一精化网格：旧默认→新默认PSNR/SSIM为HJ8 25.84/.9088→26.55/.9222，F11 28.25/.9332→28.80/.9451，HJ25 24.15/.8811→24.52/.8989，T&T Ignatius 16.81/.7234→17.47/.7527。footprint packing图集64→32MTexels。这些是纹理/渲染指标，不是几何误差。
来源：https://github.com/cdcseacave/openMVS/blob/610562a4f033be2aaf4f240512c93e1038186768/docs/design/MeshTexturing.md

### 分块在大场景有价值，但小视图集合的接缝会降低几何结果。
T&T Church 507视图、50M点、同深度图、fusion --number-views 24：12GB内存拆4块F1 0.6085，旧面积拆5块0.5967。EPFL HJ8、--refine '--scales 1'、1cm阈值：不拆分0.5225，强制4.2GB拆2块0.5133；设计文档把约0.009 F1损失归于接缝。Church表未给出本次F1距离阈值，不能与HJ8数值跨集比较。
来源：https://github.com/cdcseacave/openMVS/blob/610562a4f033be2aaf4f240512c93e1038186768/docs/design/SceneSplit.md

### 自动预算不是全流程硬性内存上限。
设计文档报告拟合模型中位相对误差fusion14%、mesh4%，且明确未计RefineMesh或最终全图TextureMesh；边界不缝合，若继续拆分不能有效降低预测成本会给出警告。
来源：https://github.com/cdcseacave/openMVS/blob/610562a4f033be2aaf4f240512c93e1038186768/docs/design/SceneSplit.md

### 上游已有测试，但没有本用户人脸数据验证。
PR报告Tests.exe 0/1/2在RelWithDebInfo与Debug通过，halfmesh桥接覆盖volume、plane split、SubMesh、Join、texture split与OBJ往返。PLY重写提交另报告4750变异文件在ASan/UBSan下无故障。此处均为作者报告，本次未执行代码、编译或复现实验。
来源：https://github.com/cdcseacave/openMVS/pull/1314

## 局限与未知
- 不是人脸专项论文，也不是SOTA几何结论。公开实验集中在EPFL/Strecha及Tanks & Temples场景；没有静态人脸毫米级误差、轮廓/鼻翼/眼睑ROI或GS渲染图输入实验。
- 总体PR、逐项提交与设计文档的测试口径不同；设计文档的现行默认F1表也不同于单项提交对照，不可拼成一个严格统一benchmark。部分关键内存/时间结果缺硬件、完整复现命令及置信区间。
- 精化设计文档承认从ground-truth网格出发仍可能漂移，粗尺度光度迭代会放大初始减面噪声；自动decimate会先丢掉几何，后续优化无法凭空恢复。此为当前方法局限，不是本轮新发现的人脸效果。
- 新自动拆分只估计fusion/mesh，两阶段预测有误差；它既不保证GPU显存足够，也不覆盖精化和全局贴图峰值。
- CPU多线程光度累加存在运行次序噪声；CUDA深度估计亦非逐次可复现。比较应复用同一深度图/初始网格，不要把两次重建的随机差异算作改动收益。
- halfmesh 0.5.1与52文件迁移扩大了回归面；应固定提交和依赖验证I/O往返、材质路径、UV、拓扑与法线。PR里的300视图PLY列表支持曾加入后撤回，最终仍以每点最多255个view为断言契约，不能误报为已支持任意视图列表。
- 本次仅作原始来源阅读和代码核验，没有运行外部代码，亦未验证用户当前安装是否包含这些改动。

## 工程建议（非复现结论）
- 首选小范围验证RefineMesh内存收益，不直接升级整条生产链。锁定旧版与610562a4，匹配halfmesh0.5.1，同一组图、相机、mask、初始网格、深度图和显式参数。
- 关闭新驱动的自动split：MvgMvsPipeline.py使用--sub-scene-memory off；MvsScalablePipeline.py使用--memory off；直接调用DensifyPointCloud的该参数是数值预算，0表示关闭，不能把字符串off照搬到二进制。
- 对高精度静态人脸，设旧版不精化基线；对照decimate=1、simplify-tolerance=0等几何保护选项应作为独立消融而非默认保证，并明确记录是否仍有remesh或其他几何处理。--scales 0在本次修复后可返回原输入Mesh。
- 鼻唇、鼻翼、眼睑、耳缘与外轮廓ROI必须独立检查：建议比较无纹理法线渲染、轮廓偏移、深度差及真实扫描误差，避免更锐的贴图掩盖几何收缩。
- 如果内存并不构成瓶颈，这一更新没有足够证据要求替换现有人脸基线；若显存限制阻止当前分辨率/视图数，再把可运行规模收益与真实几何验证一起衡量。
- 新增一键脚本的--resume以输出存在为跳过条件；A/B使用独立目录并记录每一步是否真正执行。Windows传入各阶段options仍按POSIX shlex拆分，路径需遵循脚本帮助中的引号规则。
- GS渲染图的伪细节、视角依赖颜色和轮廓不一致不会由这类内存/贴图工程改动自动消除。固定GS、相机和渲染配置，才有可解释的摄影测量A/B。

## 实际阅读范围
已核验PR及合入元数据、45条提交说明、halfmesh依赖与软件release列表；阅读RefineMesh CPU/CUDA的关键改动、TextureMesh视图选择/图集改动、PointCloud最终view-list契约、两条Python驱动；精读三份设计文档中本轮涉及的机制、验证与局限章节。未对全部52个文件做逐行审计，未执行上游代码/测试。本项为代码工程核查，不是论文全文细读。

## 原始来源
- [OpenMVS #1314：45条提交合入 develop，含作者实验报告](https://github.com/cdcseacave/openMVS/pull/1314)
- [合入提交 610562a4](https://github.com/cdcseacave/openMVS/commit/610562a4f033be2aaf4f240512c93e1038186768)
- [MeshRefinement：精化机制、基准与几何漂移局限](https://github.com/cdcseacave/openMVS/blob/610562a4f033be2aaf4f240512c93e1038186768/docs/design/MeshRefinement.md)
- [MeshTexturing：新视图选择与留出视图贴图评测](https://github.com/cdcseacave/openMVS/blob/610562a4f033be2aaf4f240512c93e1038186768/docs/design/MeshTexturing.md)
- [SceneSplit：内存模型、无缝合合并与接缝退化](https://github.com/cdcseacave/openMVS/blob/610562a4f033be2aaf4f240512c93e1038186768/docs/design/SceneSplit.md)
- [halfmesh 0.5.1 依赖发布](https://github.com/cdcseacave/halfmesh/releases/tag/v0.5.1)
- [默认梯度按采样计算的单项实验](https://github.com/cdcseacave/openMVS/commit/d219c18f334f87e9eaca3b22eeaff0032027cbe1)
- [CPU带状统计单项实验](https://github.com/cdcseacave/openMVS/commit/54779eef3927fa8224e5687cd686e9a11f96227c)
- [CUDA参考视图累加单项实验](https://github.com/cdcseacave/openMVS/commit/3393d2e0bd7585294d97bae6213d55de13e86728)
- [CUDA取消常驻深度图单项实验](https://github.com/cdcseacave/openMVS/commit/fff11c5fe2ce20531618359dbcc133860861e968)
