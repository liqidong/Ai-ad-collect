---
type: capture
review_state: 草稿
pending_revision: false
date: 2026-11-12
---

# QuickJS文件接入详解：计算留在REPL，权限边界必须落到后端

核查时间：2026-11-12T00:06:50Z

来源时点：PR #7637/#7638/#7668/#7669于2026-11-11 01:23:48–01:23:51 UTC依次合入main；本条固定最后合并提交94657a47f8d47acfc17c19157f823b7bdd902994。QuickJS 0.3.12于11月7日发布，Deep Agents 0.7.35于11月9日发布，均早于新增。

开放范围：11月11日源码合并；beta/alpha接口，未正式发行；代码与测试审阅

## 阅读范围与证据边界

已读取四个PR的说明、合并日期与完整变更；阅读固定提交完整_fs.py、_fs_shim.js、test_fs.py，以及middleware开关/绑定/权限说明和_repl桥接、SDK graph绑定相关实现。读取许可证、依赖、README及发行记录，并逐文件对照两个正式版本。核对PR头提交CI状态而非执行测试。去重读取公开日报2026年10月及11月1–11日全部日报JSON，未发现同一fs新增已报道。未全库审计、未安装运行、未调用真实模型或接入用户文件。 本期最终核查时间：2026-11-12T00:06:50Z。

## 核心问题与已有瓶颈

原有代码模式已经可以在JavaScript中组合工具、保存变量和调度子代理，但没有直接把后端文件作为数据读入的fs接口。若经文件工具返回大段文字，内容容易进入模型上下文；早期把fs叠在文件工具中间件上的方案又需要特殊原始结果与审批处理。本次改为并列的后端桥接，简化数据路径，同时明确不继承原文件工具的策略检查。

## 输入

构建时传入CodeInterpreterMiddleware(fs=True)，由create_deep_agent绑定其实际后端；也可明确把后端对象作为fs参数。None或False不开放fs。输入数据是后端可访问的UTF-8文本文件、JS代码、路径与字符串写入内容。默认StateBackend处理当前图状态内的文件；FilesystemBackend和CompositeBackend按各自配置提供文件与路由。没有训练阶段或模型权重。

## 输出

eval中获得文本、单层目录文件名、存在检查成功/错误，或写入/删除副作用；模型最终只收到代码选择返回或打印的结果。fs提供readFile、writeFile、readdir、access、rm及对应fs.promises形式。文件内容不会自动成为fs子运行的trace输出，但代码仍可把内容返回、打印或传给其他已开放工具。

## 逐步流程

### 1. 绑定正确后端

create_deep_agent检查用户中间件是否实现bind_backend，并传入与FilesystemMiddleware相同的backend；声明式子代理中间件也会绑定。fs=True必须完成绑定后才允许首次模型调用；明确传入fs对象时该对象优先，不会被Agent后端替换。

### 2. 按开关注入有限JS接口

开启fs才向系统说明和QuickJS上下文注入文件API。默认没有fs；这不是Node.js，不能require模块，也不支持同步方法、二进制、appendFile、递归readdir或stat。writeFile只创建/替换UTF-8文本。

### 3. 将调用调度回Agent事件循环

JS异步host桥接先消耗与程序化工具调用共享的调用预算，再在eval的运行配置和Agent事件循环上调用后端。这样StateBackend能访问当前图状态，而不是另一个事件循环的错误状态。

### 4. 执行后端操作并统一错误

readFile调用adownload_files后严格UTF-8解码；writeFile调用awrite；目录和存在检查借助als；rm先检查路径类型，再调用adelete。路径先进行POSIX规范化，错误映射为ENOENT、EISDIR、ENOTDIR等；这不是完整OS权限验证。

### 5. 返回受控结果并保留恢复边界

fs子运行只记录空输出，文件值经桥接直接回到JS。代码可过滤、聚合后再输出小结果。快照恢复时重新安装当前fs接口；存储是否跨进程持久仍由后端与LangGraph checkpointer决定，新增fs本身不提供持久事务。

## 关键机制与真实增量

关键增量是把文件数据通道和模型可见的文件工具通道分开，并用构建期backend绑定避免两个通道指向不同文件集。中间件不能随意重绑另一个具体后端，否则抛ValueError；同一个后端对象可再次绑定，两个恰为StateBackend类型的实例也视为可互换，因为它们读取各自运行图的状态。该校验避免部分误配置，但不等于多租户安全证明。权限并未沿文件工具中间件自动继承，能力范围由实际后端决定。

## 来源报告的效果与证据

### 新接口已在main源码中，正式版本尚不含。

固定提交的middleware.py含fs参数、bind_backend和显式危险说明；对照langchain-quickjs==0.3.12相同文件没有fs参数，deepagents==0.7.35的graph.py没有_bind_backend。不能只pip升级到这两个正式版本就期待新行为。

[原始证据](https://github.com/langchain-ai/deepagents/pull/7669)

### 读写、路由和绑定有可检查的离线回归。

test_fs.py使用FakeChatModel与假文本。读取用例在invoke和ainvoke中断言JSON内150行；写入用例检查新内容、目录列表和删除后ENOENT。fs=True另有StateBackend、FilesystemBackend、CompositeBackend三种后端乘两种调用方式的6组参数组合。

[原始证据](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/tests/unit_tests/test_fs.py)

### 内存和调用预算边界有明确反例测试。

16 MiB文本配8 MiB堆的测试期望当前eval出现MarshalError，下一次1+1仍为2；max_ptc_calls=2时第三次readFile期望PTCCallBudgetExceeded。它们验证错误传播，不是大文件吞吐或延迟基准。

[原始证据](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/tests/unit_tests/test_fs.py)

### 快照恢复与trace输出控制并非完整数据防泄漏保证。

恢复用例用InMemorySaver保存读取函数，重建Agent后仍读到预期字符串；trace测试仅断言fs.readFile子运行输出为None，不能推出最终eval结果、stdout、错误和其他工具均不泄露内容。

[原始证据](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/tests/unit_tests/test_fs.py)

### 公开CI可支持源码可测试性，但没有真实Agent质量收益。

PR #7669头提交b9936a0的最新工作流42931113974中，QuickJS的Ubuntu Python 3.11/3.12/3.13/3.14测试状态均success。旧取消运行仍有failure/cancelled记录；本轮未执行或检查全部日志，也没有任务成功率、token节省或生产隔离数据。

[原始证据](https://github.com/langchain-ai/deepagents/actions/runs/42931113974)

## 限制与未知

- fs默认关闭。开启后直接调用后端，不经过FilesystemMiddleware permission rules、create_deep_agent permissions或interrupt_on文件审批。依赖这些检查的Agent不应开启；不能把这项显式设计权衡描述为默认漏洞。
- 默认StateBackend读取图状态文件，不因此获得宿主文件系统。若显式使用FilesystemBackend(virtual_mode=False)或本地shell后端，可达范围可能扩大到宿主；受限虚拟根或真正沙箱需要由后端实现。路径中的..规范化测试不是完整符号链接、并发或跨租户安全审计。
- fs.access在该shim中仅做路径存在/类型检查，传入R_OK/W_OK等模式也不意味着真正验证操作系统读写权限。不能照搬完整Node fs语义。
- 只支持异步UTF-8文本子集；writeFile替换已有内容，rm支持显式递归删除。检查与写入/删除之间没有在本次桥接中提供事务、版本条件或并发冲突解决，失败后的副作用不自动回滚。
- 默认64 MiB QuickJS堆、256次桥接调用预算、5秒VM执行时间是配置上限，不是实测性能。VM时间不包含Python后端等待，无法保证5秒墙钟超时；rm的预检查和删除各经过一次桥接。
- 固定提交README仍有旧的“无filesystem”通用说明，新增能力以fs开关和代码文档为准；测试覆盖同步/异步fixture，但本轮未验证所有外部后端。接口标记beta，包分类仍为Alpha。
- 没有安装、运行项目或真实模型，不提供生产建议、加速百分比或恰好一次执行承诺。

## 工程判断、最小试用与建议

- 最小试验先固定94657a47f8d47acfc17c19157f823b7bdd902994，在独立环境同时使用该checkout的libs/deepagents与libs/partners/quickjs，避免旧核心包缺少bind_backend。可在quickjs目录按pyproject配置执行uv sync --group test，再运行uv run pytest tests/unit_tests/test_fs.py -q；这是建议步骤，本轮没有执行。
- 优先复用官方FakeChatModel与StateBackend假文件fixture：让eval执行JSON.parse(await fs.readFile("/data.json")).rows.length，应得到150；再写入一份新的临时文本并读回。只有明确获准的可丢弃文件才测试删除，绝不以真实工作目录作首轮验证。
- 对照组省略fs应无接口；plain create_agent配fs=True未绑定时应报错；同一中间件重绑不同磁盘后端应报错。再测试CompositeBackend路由和快照恢复，确认数据不会意外进入另一文件集。
- 若现有系统依赖文件审批，保持fs=None，继续使用受控文件工具。若确需直接数据通道，应先在后端落实最小可达范围、只读/写入策略、审计和配额，再做隔离回归；调用次数和路径规范化不能替代授权。
- 许可证为MIT，保留版权和许可声明；模型服务和数据仍各有使用条件。QuickJS包要求Python >=3.11,<4.0，依赖quickjs-rs >=0.2.6,<0.3.0、deepagents >=0.7.35,<0.8.0及LangChain/LangGraph等。quickjs-rs采用原生扩展，README说明有主流平台预编译轮子；本次新绑定还需使用同提交源码核心，版本约束下限本身不保证含新能力。
- 性能验收只比较同一公开文本、相同模型与任务：记录文件读取字节、模型可见结果大小、总tokens、错误率和端到端延迟。只有有实际测量后才能判断直接读取是否优于文件工具链。

## 原始来源

- [fs=True自动绑定后端：PR #7669](https://github.com/langchain-ai/deepagents/pull/7669)
- [新增UTF-8读取：PR #7637](https://github.com/langchain-ai/deepagents/pull/7637)
- [新增写入、列目录、存在检查和删除：PR #7638](https://github.com/langchain-ai/deepagents/pull/7638)
- [SDK向中间件传递同一backend：PR #7668](https://github.com/langchain-ai/deepagents/pull/7668)
- [固定提交：开关、绑定及权限警告](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/langchain_quickjs/middleware.py)
- [固定提交：后端操作实现](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/langchain_quickjs/_fs.py)
- [固定提交：JavaScript接口与参数边界](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/langchain_quickjs/_fs_shim.js)
- [固定提交：完整fs回归测试](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/tests/unit_tests/test_fs.py)
- [正式发行仍为langchain-quickjs 0.3.12](https://github.com/langchain-ai/deepagents/releases/tag/langchain-quickjs%3D%3D0.3.12)
- [事件循环桥接和trace输出实现](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/langchain_quickjs/_repl.py)
- [依赖与Alpha分类](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/pyproject.toml)
- [MIT许可证](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/LICENSE)
- [安装及运行背景说明](https://github.com/langchain-ai/deepagents/blob/94657a47f8d47acfc17c19157f823b7bdd902994/libs/partners/quickjs/README.md)
- [旧正式版QuickJS源码对照](https://github.com/langchain-ai/deepagents/blob/langchain-quickjs%3D%3D0.3.12/libs/partners/quickjs/langchain_quickjs/middleware.py)
- [旧正式版Deep Agents源码对照](https://github.com/langchain-ai/deepagents/blob/deepagents%3D%3D0.7.35/libs/deepagents/deepagents/graph.py)
