# Crawl4AI 0.9.4：网页如何变成可检索文本，安全补丁改变了什么

核查范围：已阅读官方 0.9.4 更新日志、Quick Start、Markdown Generation 与 Crawler Result 相关章节；未安装工具、未运行基准或安全测试。版本日期由更新日志核实。

## 解决什么问题

网页的正文常混在导航、推荐卡片和复杂表格里；直接把 HTML 塞给模型，既浪费上下文，又可能把边栏当正文。Crawl4AI 是采集与内容整理工具。2026-09-23 的 0.9.4 重点是自托管 Docker 服务的安全修复，并加速正文剪枝，不是提出一种新语言模型。

## 输入

输入通常是网页 URL，加上浏览器运行配置、等待条件及抽取规则；也可输入 raw:// 前缀的 HTML。规则可指定 CSS 字段或语言模型所需的结构。SDK 与对外 HTTP 服务的配置权限不同，不能把可信进程中的配置直接照搬到网络请求。

## 输出

结果对象同时承载成功状态、错误、原始 HTML、清理后 HTML、Markdown、链接、媒体和可选结构化抽取。raw_markdown 是未经过内容过滤的 Markdown；fit_markdown 是启用过滤后的正文候选；extracted_content 则可存放按指定字段产生的 JSON。

## 完整流程

### 1. ① 先获得页面，再判断采集是否成功

异步爬虫按 URL 和运行配置获取页面。进入下游前先检查 success 与 error_message，避免把失败页当正文。比如采集产品目录，页面下载成功与商品字段完整是两项独立检查。

### 2. ② 从页面结构中区分正文与噪声

Markdown 生成器可配合剪枝过滤器去除低价值区域，也可用 BM25 按查询筛选。过滤不是事实核验：一段被删除的页脚仍可能包含关键条款。因此原始版与精简版应分别保留。

### 3. ③ 选择可读文本或字段化输出

文章入库可使用 Markdown；重复商品卡片可用 CSS 规则提取名称、价格和链接；结构不规则时才考虑 LLM 抽取。表格另有 headers、rows 等结构，需检查表头与数值是否对齐。

### 4. ④ 带着出处送入检索系统

工程上建议把页面地址、抓取时间、版本和过滤参数一起保存，再分块与建索引。这一步是应用设计建议，不是更新日志新增功能；来源链接可帮助用户回到原页核对。

## 核心机制

新 PruningContentFilterLXML 的思路是从 DOM 叶节点向上一次累计节点指标，避免对每个节点反复遍历其子树。安全修复则统一辅助 HTTP 客户端的出站路径：robots.txt 和链接预览也经过服务端代理检查；配置解析持续携带调用来源，防止不可信配置被当作可信对象。

## 效果与证据

### 剪枝更快，但不是整条爬取链路快十倍。

Added 节报告两组剪枝耗时：134→13 ms、2200→260 ms；并声明新旧过滤输出逐字节一致。此处仅转述维护者测量，未复测。

https://github.com/unclecode/crawl4ai/blob/main/CHANGELOG.md#094---2026-09-23

### 本次修复覆盖出站控制与配置信任边界。

Security 和 Tests 节列出三个安全公告、出站代理测试与配置来源测试；不等于已经证明所有部署安全。

https://github.com/unclecode/crawl4ai/blob/main/CHANGELOG.md#094---2026-09-23

## 限制与未核实项

- 性能数字只针对剪枝阶段，网络延迟、页面渲染、模型调用仍会决定端到端速度。

- 正文过滤可能误删重要内容；表格识别还依赖结构与评分阈值，抽取到表格不代表数值准确。

- 普通库调用不会自动获得 Docker 注册的出站代理保护。需要按实际部署检查边界，不能只看版本号。

## 工程判断与下一步

- 对开放给其他调用者的服务，优先安排升级并做原有采集样本回归；本次声明无破坏性变更，但仍应检查业务输出。

- 将商品页、含合并单元格表格、动态文章和失败页面纳入验收，分别统计正文保留率、字段缺失与失败率。

- 保留原文与精简文本供差异核对；抓取权限、站点条款和 robots 策略需要独立遵守。

## 原始来源

- [0.9.4 更新日志：Security、Added、Tests](https://github.com/unclecode/crawl4ai/blob/main/CHANGELOG.md#094---2026-09-23)
- [官方 Quick Start：规则与模型抽取](https://docs.crawl4ai.com/core/quickstart/)
- [Markdown Generation：Using Fit Markdown、结果字段](https://docs.crawl4ai.com/core/markdown-generation/)
- [Crawler Result：Links、Media、Tables](https://docs.crawl4ai.com/core/crawler-result/)