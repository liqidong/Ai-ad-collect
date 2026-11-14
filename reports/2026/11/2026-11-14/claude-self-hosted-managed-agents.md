---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-14
---

# Claude Managed Agents 整体自托管：服务与会话状态可内置，模型推理仍需单独审查

2026-11-14核查官方发布说明，并研读公开总览、quickstart、功能差异、数据边界，以及Kubernetes前提/部署/入口安全、provider/gateway、API事件读取、升级备份与重连等相关章节。未取得受限发行包，未读取服务端源码或包内operator-guide，不称代码审计或独立运行复现；没有论文实验表可引用。

> 来源事实、作者测试与工程建议分列；未独立复现。

## 核心问题与新增变化

原有 self-hosted sandboxes 只把工具执行环境放在客户基础设施上，Agent 服务及会话历史仍由 Anthropic 托管。11月12日官方新增的 self-hosted Managed Agents 允许同时部署服务和状态存储，适合必须掌控会话数据落点、推理供应商及运维策略的团队。增量是部署责任边界移动，不是新模型、开源权重，也不是给现有 Claude Code 换一个参数即可获得。

## 输入

无训练输入或新权重。运行输入包括 Agent 定义、用户会话事件、环境配置、工具结果和三类不同用途的凭据：应用访问服务的 server API key、服务调用模型的 inference credential、worker领取环境任务的 environment key。模型请求送往配置的推理供应商，默认Claude API。

## 输出

系统产生会话事件流、Agent回复、工具调用和执行产物，并把agents、sessions及history存入自己管理的数据库。工具运行在worker的容器/VM，不在服务进程中。持久会话状态与沙箱文件持久性不是同一个问题，不能由数据库已保存推断工作目录一定保留。

## 逐步流程

### 1. 先取得发行包并建立本地服务状态

需要联系Anthropic account team取得下载资格。开发验证使用发行包的Docker Compose，把服务、PostgreSQL及worker组件一起启动；先检查 /cma-server/readyz，而不是仅看容器正在运行。没有数据库配置时状态只存内存，重启会丢失。

### 2. 客户端连接自托管控制面

SDK或ant CLI把base URL改为本地服务地址，API key改为server key；不要混用模型API key。创建agent与environment后，创建session并发送user.message。服务接受请求、管理会话并调用模型，不意味着工具执行已经完成。

### 3. 服务向指定模型端发起推理

provider可选anthropic、bedrock、vertex、foundry或gateway。配置负责凭据与模型ID映射；官方要求映射前后是同一模型，因为请求按agent声明的模型构造。自定义gateway的三类协议格式不等于已证明任意GPT/开源模型可以无损替换。

### 4. worker在独立环境执行工具并回传

worker主动轮询环境任务，执行shell/文件工具，把结果交回会话供下一次模型调用。Kubernetes示例使用独立dispatcher及每session一个sandbox pod；工具算力与控制面分离，可分别配置隔离和资源。

### 5. 应用消费事件并处理等待状态

客户端应先订阅event stream再发送消息，避免错过新事件。遇到requires_action不能简单当成完成；错误事件也可能后续重试。若环境没有worker，模型仍能开始推理，但工具调用会等待，必须另做worker存活和任务进展监控。

### 6. 为升级和恢复维护状态

官方升级文档要求按版本变更处理schema和备份；跨较多版本可能需要停服。替换服务时事件流会断开，客户端需要重连。数据库备份包含会话内容，应加密并与密钥分离；这仍不构成外部副作用exactly-once的证明。

## 关键机制

这是服务API/agent loop、持久数据库、工具worker、推理端四部分分离的部署方式。相较只自托管sandbox，客户取得控制面与history存储的部署权，同时承担安全、版本、备份和隔离责任。模型能力本身没有因部署位置改变而提升。工程上可把它看成可自管的官方Harness产品，而不是任意BYOM的开源框架。

## 来源事实与证据

### 官方已宣布提供受限申请的发行下载，不是人人可公开拉取的代码/权重。

官方release notes日期为2026-11-12；overview和quickstart均要求联系account team。公开页面未给本次可下载包的精确版本、资产哈希或首次上传时间，本轮没有取得包。

[原始证据](https://platform.claude.com/docs/en/release-notes/overview#november-12-2026)

### 公开文档给出可检验的最小部署链路，但没有独立复现。

Quickstart用Docker Compose、readyz及hostname对照容器名验证工具确实在sandbox运行。该验证是执行位置检查，不是任务质量、长时稳定性或吞吐基准。本报告未执行。

[原始证据](https://platform.claude.com/docs/en/managed-agents/self-hosted/quickstart#run-your-first-session)

### 生产部署有具体基础设施要求，不是一个CLI进程的轻量替代。

Kubernetes文档要求1.30+、PostgreSQL16+、gVisor节点及可执行NetworkPolicy的网络插件；两张Helm chart分离服务与sandbox，示例运行两个server副本。未提供规模压测、成功率、恢复时间或费用对照表。

[原始证据](https://platform.claude.com/docs/en/managed-agents/self-hosted/kubernetes#prerequisites)

### 自托管会话存储不等于内容完全不出网。

数据说明明确：每次模型请求包含当前会话、工具输入和结果，交给选定推理端；模型侧保留规则依供应商协议。worker网络权限独立于server的出站清单。

[原始证据](https://platform.claude.com/docs/en/managed-agents/self-hosted-data-and-access#what-leaves-your-network)

## 限制与未知

- 下载须申请；受专用Software License Agreement约束，不能称MIT/Apache开源。没有读到发行包内部operator-guide或服务器源码，没有安装验证。
- 功能不等价于托管版：缺scheduled deployments、dreams和webhooks；自建调度器与轮询/事件流须另接。多Agent仅支持预定义coordinator，1至20个子Agent，不支持动态定义子Agent或advisor。
- 不提供Messages/Models/Files API；不支持file会话资源和mcp_oauth自动刷新。迁移不能只替换URL，也不会自动复制原有会话/skills/vault/memory。
- 每个deployment只对应一个tenant和信任等级，environment不是信任隔离边界；不同信任域应分部署。日志/trace/备份和worker文件需单独纳入生命周期审计。
- Compose是开发方案且为HTTP；示例mock登录可能给可达者管理员权限，不应用于真实数据。worker本身不隔离shell，真正隔离依赖容器或VM。
- 缺少公开生产可靠性、取消响应、重复副作用、真实长任务成功率/token成本实验；本文只评价机制与接入约束，不宣称领先或稳定生产可用。

## 工程判断与未执行的试用方案

- 最小试用：先申请发行包并审查其版本说明/许可证；在无敏感数据的机器上配置最新Docker Compose、模型调用账户、独立密钥和PostgreSQL，启动后检查readyz；创建测试agent/session，运行hostname并对照sandbox容器。只把这当部署冒烟验证，不上传实际业务数据。
- 工程判断：若瓶颈只是代码在内网机器执行，优先保留只自托管sandbox的较小运维面；必须自管会话记录和推理出口时，整体自托管才有直接收益。
- 扩展验收应独立测四层：server与DB重启后的会话恢复、worker中断后的工具状态、事件流重连是否漏读、外部副作用重试是否重复；再测取消时延、预算和长任务成功率。官方文档未替你完成这些证明。
- 服务端推理凭据不应交给worker。模型gateway仅解决路由和协议/凭据接入，不能据此假设可随意替换模型或绕过模型数据保留约束。
- 升级成本包括自建镜像、网络策略、TLS、数据库schema、客户端重连和备份恢复。应先验证回滚，不把“两个server副本”当作无故障或零数据丢失承诺。

## 原始来源

- [官方11月12日发布说明](https://platform.claude.com/docs/en/release-notes/overview)
- [自托管服务总览](https://platform.claude.com/docs/en/managed-agents/self-hosted)
- [开发Quickstart](https://platform.claude.com/docs/en/managed-agents/self-hosted/quickstart)
- [与托管服务的功能差异](https://platform.claude.com/docs/en/managed-agents/self-hosted/differences)
- [数据、连接与租户边界](https://platform.claude.com/docs/en/managed-agents/self-hosted-data-and-access)
- [Kubernetes部署](https://platform.claude.com/docs/en/managed-agents/self-hosted/kubernetes)
- [推理provider与gateway格式](https://platform.claude.com/docs/en/managed-agents/self-hosted/inference-provider)
- [SDK/API与事件流接入](https://platform.claude.com/docs/en/managed-agents/self-hosted/call-the-api)
- [升级与回滚](https://platform.claude.com/docs/en/managed-agents/self-hosted/upgrade)
