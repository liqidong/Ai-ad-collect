---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-04
---

# RAGFlow：文件版本提交增加目录树授权，收紧跨工作区写入边界

来源日期：PR创建2026-10-30T16:28:28Z；合并2026-11-03T11:47:47Z；2026-11-04 Asia/Shanghai核查

核查时间：2026-11-04T00:09:05Z

状态：新合入数据隔离修复；非检索方法；未实测

## 核心问题

文件版本提交入口已有目标目录访问检查，但请求中的文件ID是另一组独立输入。只检查URL中的目录，不能证明批量修改的每个文件都属于该目录。作者将此归为IDOR/CWE-639问题；本文不据此推断所有版本或部署均受影响，也不声称已有正式漏洞编号。

## 训练与推理输入

无训练或模型推理。输入包括目标目录folderID、作者、提交说明、FileChange列表，以及上一提交的tree_state。每条变更包含文件ID、操作和对应内容/名称；数据库中的文件parent_id用于祖先链判断。

## 输出与资产形式

合法提交继续生成FileCommit、FileCommitItem和目录树快照，内容写入现有对象存储；不合法文件使事务返回权限错误。handler将越界资源隐藏为应用层NotFound，测试仍期望HTTP 200，不能把它写成HTTP 404。

## 分步流程

### 1. 保留入口授权，逐文件补齐作用域

handler先确认用户与目标目录访问，再进入CreateCommit。修复不重新设计角色权限，而是在文件写入之前补上目录树归属检查。

### 2. 在事务内锁定目录及文件行

authorizeCommitFiles在保存新commit之前运行；目标必须是目录，文件ID不能为空。按ID聚合操作并使用GORM UPDATE锁读取相关文件，目标是避免验证后、更新前文件被移动。PR正文仍写“事务前检查”，与最终代码不一致，本文以固定提交为准。

### 3. 分开处理内容写入与删除历史

同一ID只要有非delete操作，就必须存在活跃文件行并通过目录树归属检查；纯delete只要求ID在此前tree_state中出现，允许记录已删除或已移走文件的历史，同时拒绝给从未属于本目录历史的ID制造删除记录。原有逻辑仍拒绝为当前目录内的活跃文件记删除。

### 4. 按层批量核查祖先链

逐层加载去重后的父节点，而非为每个文件单独查询整条链；最多64轮加载，最终用visited集合检测循环，并判断是否能够到达目标目录。缺失祖先或无法到达目标的链会拒绝。

### 5. 写入与错误返回

检查通过才创建commit、处理内容/重命名及历史条目。事务错误回滚数据库写入；handler沿用权限错误转换，避免把越界输入显示为内部服务错误。

## 关键机制

增量是把批量文件引用的对象级授权约束放到写事务内，并对历史删除使用不同的归属规则。它依赖文件父链和既有目录权限作为边界，不是逐行新增tenant_id相等检查，也不新增RAG召回或答案指标。

## 来源事实与报告效果

- 新增测试覆盖跨租户、同租户兄弟工作区、越界重命名和不存在的add对象。：四类构造用例断言提交被拒绝，目标文件名称、位置、大小保持不变，且没有保存新commit。作者称修复前红、修复后绿；本报告仅阅读，未复现。 [原始证据](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_authorization_test.go)
- 合法直接子文件和嵌套子文件仍能提交。：正例构造两个文件修改，断言FileCount为2、大小和存储位置更新。数字为夹具断言，不是性能/质量实验。 [原始证据](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_authorization_test.go)
- 删除历史规则有独立回归断言。：新测试拒绝无历史的未知ID及其他目录文件ID，检查未生成对应FileCommitItem；原有例子保留已移走但有历史文件的删除记录。 [原始证据](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_delete_test.go)
- 测试环境不能等同真实多连接事务验证。：测试夹具使用内存SQLite、内存对象存储，MaxOpenConns和MaxIdleConns都为1；作者提供Go测试通过说明，但本次未执行，也未核验生产数据库并发移动。 [原始证据](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_page_test.go)

## 限制与未知

- 公开Apache-2.0代码已合并；正式稳定包、容器部署与用户安装状态未验证。nightly发布说明指向11月3日较新的主分支提交，不等于本文验证该镜像可用。
- 检查范围是Go文件提交路径，不是整个RAGFlow权限体系、Python路径或所有租户数据一致性的全面证明。
- SQLite单连接夹具不验证真实数据库的SELECT FOR UPDATE、并发移动、死锁与吞吐。按层批量查询是实现方式，没有实测性能收益。
- tree_state在事务开始前读取，补丁没有据此证明并行提交历史的全局串行一致性；历史JSON损坏时会回退为空，可能拒绝本应合法的历史删除。
- 外部对象存储Put不是数据库事务资源，不能由数据库回滚推出对象存储副作用也会撤销。本文未审计或验证失败后的对象清理。
- 64轮祖先加载给损坏父链设限，极深合法目录也需专项边界测试；本次未运行。

## 工程建议，非复现实验

- 先确认自己使用的Go版本、文件提交入口与部署SHA，再决定升级或回补；不要仅看到仓库合并就认定线上已修复。
- 最小验收全部使用自有测试账户与虚构数据：同目录直接/嵌套修改成功；兄弟工作区与另一测试租户文件修改被拒绝且文件不变；无历史删除不生成版本记录。
- 增加真实后端上的并发移动、重复混合操作、缺失祖先、循环父链、深层目录及存储写入失败测试。建议记录应用层错误码，不能只检查HTTP状态。

## 阅读范围

已阅读PR正文、固定提交全部五个改动文件补丁，以及CreateCommit、authorizeCommitFiles、祖先链检查、handler权限/错误处理与SQLite夹具的相关完整函数和许可证。非论文，无实验表。未安装、构建、运行单测或攻击真实服务；核查日期2026-11-04 Asia/Shanghai。

## 原始来源

- [合并提交：文件提交限定在目标目录树](https://github.com/infiniflow/ragflow/commit/33806977d1e57c1e7bb0f817d0c795ba54adeb46)
- [PR #21795：问题、设计与作者测试说明](https://github.com/infiniflow/ragflow/pull/21795)
- [事务内授权与祖先链检查源码](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit.go)
- [跨工作区、跨租户和合法子目录测试](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_authorization_test.go)
- [删除历史记录测试](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_delete_test.go)
- [HTTP handler及错误转换](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/handler/file_commit.go)
- [SQLite测试夹具](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/internal/service/file/file_commit_page_test.go)
- [Apache-2.0许可证](https://github.com/infiniflow/ragflow/blob/33806977d1e57c1e7bb0f817d0c795ba54adeb46/LICENSE)
