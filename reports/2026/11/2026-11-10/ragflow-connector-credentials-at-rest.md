---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-10
verified_at: 2026-11-10T00:10:33Z
---

# RAGFlow：连接器凭据可选静态加密，先补密钥与恢复流程

来源时点：PR #20241于2026-09-25提出，2026-11-09 05:44:29 UTC合入main（5d008556de66b61203b589be20cbd8fc060eceea）。最新稳定v1.0.0于2026-11-06发布，早于改动。

开放状态：11月9日合入main；正式版尚不含；源码和测试审阅

核查时间：2026-11-10T00:10:33Z

## 阅读范围与证据等级

已读官方PR正文、全部变更清单，并逐段研读新增加解密实现全文、迁移实现与测试全文、服务层变更及凭据测试全文、实体数据库序列化变更、同步工厂及重试测试；未运行代码，非独立安全审计。

## 核心问题

连接器config.credentials原先以明文JSON持久化OAuth token、client secret和API key。数据库副本泄漏会直接泄漏这些凭据；本次增加字段级静态加密，而不是更改采集授权。

## 输入

连接器config映射，其中credentials为待保护JSON；部署环境提供标准base64编码的32字节RAGFLOW_CONNECTOR_KEY。历史迁移另输入数据库中未加密的config原始文本。未配置密钥仍走原有明文模式。

## 输出

数据库只把credentials字段写为enc:v1:加base64封装的nonce、密文与认证标签；其他config仍为明文。数据库读取阶段保留密文，授权API响应或同步连接器使用时才解密，输出原凭据对象。

## 逐步流程

### 1. 启动校验密钥

密钥来自运维设置的RAGFLOW_CONNECTOR_KEY，标准base64解码后必须恰好32字节；代码拒绝换行及非法base64，启动时即校验。不自动生成或调用KMS。

### 2. 字段序列化与认证加密

把credentials JSON序列化，使用AES-256-GCM；crypto/rand生成12字节nonce，存储nonce || ciphertext || 16字节tag。AAD为nil；withConnectorCredentials复制外层map，避免改写调用者输入。

### 3. 数据库写入加密、读取保留密文

实体ConnectorConfig.Value负责写入加密；Scan只解析JSON不解密，因而权限检查、删除、任务读取/认领不依赖密钥。GORM map update显式使用ConnectorConfig，避免绕过写入加密。

### 4. 以比较后交换迁移旧记录

ragflow_server --migrate按256行批处理历史记录，跳过非法JSON、无credentials与已有enc:值。UseNumber保留大整数；更新条件同时匹配id及原始config文本，避免覆盖迁移期间的新修改。重复执行不会重加密。

### 5. 限定解密位置与客户端输入

创建、PATCH与test-connection拒绝客户端提交顶层enc:凭据，防止把拷贝密文送入API后取回明文。GET/创建/PATCH响应仍解密回传；同步只在连接器工厂需要凭据时解密。

## 关键机制

代码推导：主要降低仅获得数据库或其密文备份、但拿不到部署密钥的攻击者读取credentials的风险。不能保护已能访问应用进程/环境密钥、读取授权GET响应或客户端编辑表单的人。空AAD未把密文绑定租户/连接器ID，不能据此宣称防御可任意篡改数据库的攻击者；API的enc:输入拒绝仅封住客户端密文回送路径。 默认不设置密钥，仍以明文写入，迁移也不做任何事。已加密行遇到缺失或错误密钥会报错，不自动解密降级：GET失败；仅修改频率/状态/重调度的PATCH在写入前失败；同步尝试3次后FAIL。删除与用完整新config替换仍可执行。重要运维边界：丢密钥后不配置新密钥而直接替换新凭据，会回到默认明文存储，因此恢复流程应先布置并备份新密钥。

## 原始来源报告的结果与证据

### 公开测试向量覆盖加密格式与错误边界

新增固定公开测试向量，检查格式兼容、加解密往返、密文篡改/错钥/缺钥/短密文失败以及错误不泄漏密钥、密文或明文。

[原始证据](https://github.com/infiniflow/ragflow/pull/20241)

### 迁移测试跨越批边界且避免覆盖新数据

迁移测试覆盖257行跨越256行批界、重复迁移不变、CAS跳过过期值及9007199254740993、1234567890123456789大整数的存储保真。

[原始证据](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/dao/connector_credentials_migration_test.go)

### 服务和同步测试区分密文存储与失败重试

服务测试验证创建/PATCH数据库中只有密文而返回配置仍为明文；失钥时状态与调度不变；同步测试验证前两次重调度、第三次FAIL且日志错误不含测试secret。

[原始证据](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/service/connector_credentials_test.go)

### 作者测试结果存在明确未通过项

作者报告相关Go包通过；go test ./internal/...仍有1个TestDispatch_PDFLegacyMarkdownConfigurationEmitsJSON失败，作者称main也失败。未提供独立性能基准，未在本研究复跑。

[原始证据](https://github.com/infiniflow/ragflow/pull/20241)

## 限制与未知

- 不覆盖credentials之外的REST headers/auth_config，也不覆盖ChatChannel.config。
- 没有密钥轮换、KMS集成、GET脱敏、全链路日志脱敏或历史备份清理。新增错误路径有保密测试，不代表整套系统日志安全。
- 历史明文数据库备份、复制日志、导出文件不会被本次当前表迁移自动清除；数据库备份与部署密钥同处泄漏时保护失效。这是机制推导，不是PR实测。
- 迁移UseNumber保护存储字节，但正常DecryptConnectorCredentials用json.Unmarshal到interface{}，数字仍按float64解码；测试明确以原始解密字节验证超大整数，不能声称端到端大整数无损。
- 本功能不授予数据抓取、再分发或模型训练许可，不构成合规认证。
- 代码已合入main，但截至核查的v1.0.0正式版不含；nightly资产内容未核验。无独立攻防审计或吞吐/延迟/检索准确率基准。

## 工程建议（未实测）

- 2026-11-09T05:44:29Z合并到main，merge commit 5d008556de66b61203b589be20cbd8fc060eceea。检查时最新稳定release为v1.0.0（11/6 12:31:31Z），本变更尚未进入该正式版。nightly标签存在但其资产构建内容未核验，不作可用承诺。
- 仅建议隔离副本验证，不在生产直接迁移：锁定合并commit；用无真实权限的假token构造一条历史明文记录；由运维生成并安全备份32字节随机密钥，配置服务与worker；跑迁移并检查credentials成为enc:v1:、其他字段与大整数保持；二次迁移应不变；移走密钥验证GET失败、非config PATCH无副作用、三次同步后FAIL，再恢复密钥。当前研究只读，没有生成密钥、安装或运行。
- 对接多数据源且会备份RAG数据库的团队值得纳入下一轮升级评估。首先确定密钥保存/恢复与备份隔离，不能仅设置一个环境变量后把整个接入链路称为安全。
- 密钥属于敏感部署资产，应由受控秘密管理流程配置；日报没有生成、读取或操作任何实际连接器密钥。

## 原始来源

- [PR #20241及作者测试结果](https://github.com/infiniflow/ragflow/pull/20241)
- [固定合并提交：凭据加解密实现](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/common/connector_credentials.go)
- [固定合并提交：历史记录迁移](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/dao/connector_credentials_migration.go)
- [固定合并提交：服务层凭据回归](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/service/connector_credentials_test.go)
- [v1.0.0正式发布记录](https://github.com/infiniflow/ragflow/releases/tag/v1.0.0)

- [固定合并提交：迁移批界、CAS与大整数测试](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/dao/connector_credentials_migration_test.go)
- [固定合并提交：同步失钥与三次重试测试](https://github.com/infiniflow/ragflow/blob/5d008556de66b61203b589be20cbd8fc060eceea/internal/syncer/syncer_test.go)
