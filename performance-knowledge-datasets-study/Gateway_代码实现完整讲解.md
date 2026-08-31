# PerfDog Service Gateway：代码实现完整讲解

> 本文仅整理 Gateway 的两部分讲解内容：
>
> 1. Gateway 的三层实现、协议、Adapter、Job/Worker、部署；
> 2. 启动与健康检查、Agent Context、卡片治理、RAG 发布、异常协议与边界。
>
> 本文不修改任何现有工程逻辑，也不替代已有的完整服务文档。

---

# 第一部分：Gateway 的主体实现

## 1. Gateway 是怎么实现的？

可以把 Gateway 的代码实现概括成：

> **三层结构 + 两类 Adapter（适配器）+ 一条异步 Job 任务链路。**

```text
HTTP REST / HTTP MCP / stdio MCP
            ↓
      api.py / mcp.py          协议层
            ↓
        runtime.py             统一业务分发层
            ↓
   ┌───────────────┴───────────────┐
   ↓                               ↓
dual_rag adapter              wiki_starter adapter
直接调 Python 函数             白名单 + 子进程执行脚本
   ↓                               ↓
检索、Agent Context、建库       抓取、抽卡、校验、Wiki
            ↓
      workspace_jobs.py
            ↓
MySQL Job + lease + 单写锁 + Worker
```

---

## 2. 第一层：协议层——解决“外部怎么调用”

Gateway 同时提供三种入口：

| 入口 | 适用对象 | 含义 |
|---|---|---|
| REST HTTP | 后端、脚本、网页、自动化系统 | 普通 Web API，例如 `POST /api/retrieve` |
| stdio MCP | 本地 IDE、桌面 Agent | 通过标准输入输出传 JSON-RPC 消息 |
| HTTP MCP | 远程 Agent、MCP Client | 通过 HTTP 调用 MCP 工具 |

### 2.1 什么是 REST HTTP？

REST HTTP 就是最常见的 Web 接口调用形式，例如：

```text
POST /api/retrieve
```

客户端发送 JSON 请求，服务端返回 JSON 结果。

例如调用检索能力时，调用方传入查询、平台、指标等条件；Gateway 转发给 Dual RAG，返回相关知识卡和证据。

### 2.2 什么是 MCP？

MCP 可以理解成一种让 AI Agent “发现并调用工具”的协议。

普通 HTTP 更像：

```text
调用方明确请求某个 URL
```

MCP 更像：

```text
Agent 先问：你有哪些工具？
Gateway 返回工具列表
Agent 再调用：我要调用检索工具
```

因此，MCP 特别适合 IDE 或智能体调用知识库、构建任务等能力。

### 2.3 `api.py` 做什么？

`api.py` 使用 Python 标准库的 `ThreadingHTTPServer` 启动 HTTP 服务。

`ThreadingHTTPServer` 的意思是：每个 HTTP 请求可以由独立线程处理，因此不会因为一个短请求阻塞其他短请求。

它负责：

1. 读取 HTTP 请求体中的 JSON；
2. 根据 URL 判断要执行哪个功能；
3. 返回 JSON 响应；
4. 处理 CORS；
5. 将参数错误返回为 `400`；
6. 将未知路由返回为 `404`；
7. 将未捕获异常返回为 `500`。

例如：

```text
POST /api/retrieve
→ 同步检索知识

POST /api/wiki-starter/ingest-source
→ 将“新增资料并构建知识”的任务放入后台队列

POST /api/dual-rag/build-assets
→ 将“构建 RAG 语料或向量索引”的任务放入后台队列
```

这里要区分：

- **检索**是读操作，通常比较快，因此同步返回；
- **抓取、LLM 抽卡、建向量库**是长任务，因此异步执行。

### 2.4 stdio MCP 怎么实现？

stdio 是 `standard input/output`，即标准输入和标准输出。

本地 IDE 启动 Gateway 的 MCP 进程后：

```text
IDE 往 stdin 写 JSON-RPC 消息
Gateway 从 stdin 读取
Gateway 往 stdout 返回 JSON-RPC 响应
IDE 从 stdout 读取
```

消息通过 `Content-Length` 分帧。它的作用是告诉接收方：

> 接下来这一段 JSON 消息有多少字节，避免连续多条消息粘在一起无法区分。

MCP 支持的主要方法是：

```text
initialize
tools/list
tools/call
ping
```

其中：

- `initialize`：初始化协议；
- `tools/list`：查询有哪些工具；
- `tools/call`：调用具体工具；
- `ping`：健康检查。

重点是：**MCP 层不写检索或建库逻辑。**

它收到 `tools/call` 后，最终仍然会调用 `runtime.call_mcp_tool()`，由统一业务层处理。

### 2.5 HTTP MCP 怎么实现？

HTTP MCP 仍然使用同一个 HTTP Server，只是走：

```text
POST /mcp
DELETE /mcp
```

基本流程：

```text
客户端发送 initialize
→ 服务端创建 session
→ 返回 Mcp-Session-Id
→ 客户端后续请求带上该 session id
→ tools/list 或 tools/call
→ DELETE /mcp 时清理 session
```

这里：

- `Mcp-Session-Id`：一次 MCP 会话的临时编号；
- `MCP-Protocol-Version`：双方协商使用的 MCP 协议版本；
- `session`：用于记录客户端已经初始化过的状态。

当前 session 存在 Python 进程内存的字典中，并通过线程锁保护。

这意味着：

> 当前 HTTP MCP 适合单实例运行；如果部署多个 Gateway 副本，请求被转发到不同副本时，另一个副本的内存里没有原 session，就会出问题。

---

## 3. 第二层：`runtime.py`——统一业务分发

`runtime.py` 可以理解为 Gateway 的“总调度台”。

它主要解决一个问题：

> HTTP、stdio MCP、HTTP MCP 都能调用同一项业务能力，但不能让三套协议各自写一份业务逻辑。

例如“知识检索”这项能力：

```text
REST:
POST /api/retrieve
→ runtime 的检索处理函数
→ Dual RAG adapter
→ 返回检索结果

MCP:
tools/call(perfdog_retrieve_knowledge)
→ runtime.call_mcp_tool()
→ 同一个检索处理函数
→ Dual RAG adapter
→ 返回同一份检索结果
```

这叫做**协议层与业务层分离**。

好处是：

- REST 改参数时，MCP 不会漏改；
- MCP 改返回结构时，HTTP 不会出现不同结果；
- 核心业务逻辑只维护一份；
- 后续增加新协议也可以复用 runtime。

### 3.1 MCP Tool 是什么？

MCP Tool 就是暴露给 Agent 的“可调用函数”。

例如：

```text
perfdog_workspace_health
perfdog_retrieve_knowledge
perfdog_build_agent_context
perfdog_dual_rag_build_assets
perfdog_wiki_starter_ingest_source
```

其中：

- `perfdog_retrieve_knowledge`：检索知识；
- `perfdog_build_agent_context`：构造给下游 Agent 使用的结构化证据上下文；
- `perfdog_wiki_starter_ingest_source`：新增一条资料并跑知识构建；
- `perfdog_dual_rag_build_assets`：构建 RAG 文档或 Qdrant 索引。

代码中用一个类似 `_TOOL_HANDLERS` 的映射表：

```text
MCP Tool 名
→ 对应的业务处理函数
```

因此，未知工具名会被拒绝，而不会执行任意代码。

---

## 4. 第三层：为什么要用两类 Adapter？

Adapter 的意思是“适配层”。

因为两个下游项目的代码组织方式不同，Gateway 不能用完全相同的调用方法。

### 4.1 Dual RAG Adapter：直接调用 Python 函数

`perfdog_dual_rag` 已经是一个结构清晰的 Python 包，核心能力已经封装为 Python 函数。

因此 Gateway 可以直接调用它，例如：

```text
构建检索结果
构建 Agent Context
构建 RAG corpus
构建 Qdrant 向量索引
```

这种方式的特点：

```text
Gateway
→ 直接 import Dual RAG 的函数
→ 传 Python 参数
→ 获得 Python 返回值或异常
```

优点：

- 不需要额外启动子进程；
- 调用路径短；
- 返回值和异常更容易处理；
- 性能更好；
- 可以共享同一进程里的本地锁。

你可以说：

> Dual RAG 已经是包化能力，因此 Gateway 把它当作内部 Python 服务调用，而不是把所有功能都包装成 shell 命令。

### 4.2 Wiki Starter Adapter：任务白名单 + 受控子进程

`perfdog_llm_wiki_starter` 是脚本型工程，核心流程分散在多个脚本：

```text
01_build_sources.py
02_scrape_sources.py
03_validate_scrapes.py
04_extract_cards.py
05_validate_cards.py
06_render_wiki.py
```

它没有天然统一的 Python Service API。

所以 Gateway 不会让用户传：

```text
“请执行任意 Python 命令”
```

而是采用受控方案：

```text
调用方传 task 名
→ Gateway 检查 task 是否在白名单
→ 根据 task 拼接固定脚本命令
→ 只透传允许的参数
→ subprocess.run() 启动子进程
→ 设置超时
→ 截断日志
→ 返回结果与产物摘要
```

例如允许执行的任务包括：

```text
add_source
scrape_sources
fetch_video_transcripts
validate_scrapes
extract_cards
validate_cards
render_wiki
delete_cards
set_rag_visibility
```

其中：

- `task`：想执行的受支持任务名；
- **白名单**：只允许执行预先写好的任务，拒绝未知任务；
- `subprocess.run()`：Python 启动外部进程的方法；
- `timeout`：脚本最长允许运行多久，超时就中断；
- `stdout/stderr`：子进程的正常日志和错误日志；
- **日志截断**：只保留末尾有限字符，避免大日志塞满 HTTP 响应或 MySQL。

这种设计的价值是：

> Gateway 把脚本工程封装成可控服务能力，同时避免 HTTP 请求直接变成“远程任意命令执行”。

---

## 5. 为什么写操作不直接执行，而要 Job + Worker？

这是 Gateway 最关键的工程设计。

### 5.1 哪些操作属于长任务？

例如：

```text
网页抓取
视频字幕获取和 ASR
LLM 抽卡
批量知识卡校验
Wiki 渲染
RAG 语料构建
Qdrant 向量建库
```

这些操作可能持续几十秒、几分钟，甚至更久。

如果 HTTP 请求直接同步执行，可能发生：

```text
浏览器一直等待
→ 反向代理超时
→ 客户端断开
→ 用户不知道任务到底成功没有
→ 多个请求同时改同一份卡片或索引
```

所以写接口不会立即跑完整任务，而是：

```text
创建 Job 记录
→ 写进 MySQL
→ 立即返回 job_id
→ Worker 在后台慢慢执行
```

这里：

- `Job`：一条后台待办任务记录；
- `job_id`：这条任务的唯一编号；
- `Worker`：不断从队列领取任务并执行的后台进程；
- `payload`：用户提交的原始任务参数，例如 URL、任务类型、构建模式；
- `result`：Worker 最终执行完成后写入的结果；
- `status`：任务状态。

写接口最初返回：

```text
queued=true
job_id=...
```

它只表示：

> 任务已成功写入 MySQL 队列。

**不表示**：

> 网页已经抓取成功，知识卡已经生成，RAG 已经更新。

调用方之后需要通过：

```text
GET /api/jobs/<job_id>
```

轮询任务状态。

任务状态主要是：

```text
queued
→ 已入队，等待 Worker

running
→ 已被 Worker 领取，正在运行

succeeded
→ Worker 已完成，结果写入 result

failed
→ Worker 捕获异常，错误写入 error
```

---

## 6. Worker 如何执行 Job？

Worker 启动后，会持续轮询 MySQL，例如每两秒尝试一次：

```text
claim_next_job()
→ 没有任务：等待
→ 有任务：领取并执行
```

`claim` 的意思是：

> 某一个 Worker 正式声明“这条任务由我处理”。

领取后，整个过程是：

```text
queued
→ Worker claim
→ status 变为 running
→ 写入 worker_id
→ 写入 lease_expires_at
→ 获取全局单写锁
→ 启动 heartbeat 线程续租
→ 执行对应 Adapter
→ 刷新 cards_index
→ 必要时发布 RAG release
→ status 变为 succeeded 或 failed
→ 释放单写锁
```

### 6.1 Lease 是什么？

`lease` 可以理解为“带过期时间的任务处理权”。

当前逻辑大致是：

```text
Worker 领取任务
→ 获得 300 秒处理权
→ 每 60 秒 heartbeat 一次
→ heartbeat 续长处理权
```

这里：

- `lease_expires_at`：当前处理权在哪个时间点过期；
- `heartbeat`：Worker 定期告诉 MySQL“我还活着、还在处理这条任务”；
- `attempts`：该任务被成功领取过多少次；
- `worker_id`：哪一个 Worker 在执行，通常类似“机器名:进程号”。

为什么要这么做？

因为 Worker 可能崩溃：

```text
Worker 已领取任务
→ 机器宕机或进程被杀
→ Job 一直是 running
→ 没有 lease，就永远没人再处理它
```

有 lease 后：

```text
Worker 崩溃
→ 不再 heartbeat
→ lease 过期
→ 后续 Worker 发现该任务超时
→ 将任务重新放回 queued
→ 新 Worker 可以再次处理
```

所以：

> Lease 解决“Worker 崩溃后任务永久卡死”的问题。

注意，`attempts` 增加表示任务被重新领取过，**不等于所有外部 API 都自动重试过**。

### 6.2 Single Writer Lock 是什么？

即使有多个 Worker，多个写任务也不能同时修改共享资产：

```text
cards_validated.jsonl
wiki/
rag_docs.jsonl
Qdrant 索引
```

否则可能发生：

```text
任务 A 正在写 cards
任务 B 同时删除 cards
→ 文件覆盖、状态错乱、索引与文件不一致
```

因此，Gateway 在执行写任务前，会通过 MySQL 获取全局的 `single-writer lock`。

它相当于一把全局写锁：

```text
Worker A 获取锁
→ A 可以修改知识资产
→ Worker B 即使已经领到任务，也要等待锁释放
→ A 成功或失败后释放锁
→ B 才能继续
```

要区分：

- **lease**：决定“哪一个 Worker 有资格处理某一个 Job”；
- **single-writer lock**：决定“同一时间能否有多个 Job 改共享资产”。

两者解决的问题不同。

---

## 7. `ingest_source` 为什么比普通脚本调用更复杂？

`ingest_source` 是“新增一条 URL，并跑完整知识构建链路”的组合任务。

它大致执行：

```text
add_source
→ 抓网页或获取视频字幕
→ validate_scrapes
→ extract_cards
→ validate_cards
→ render_wiki
```

其中第二步会根据来源的抓取策略决定：

```text
普通网页
→ scrape_sources

视频来源
→ fetch_video_transcripts
→ 必要时 ASR
```

### 7.1 为什么不能只看脚本 `exit code=0`？

`exit code=0` 只代表：

> Python 脚本进程没有以异常退出。

但不代表：

```text
这条 URL 抓到了正文；
正文通过质量校验；
抽出了知识卡；
最终有 validated card；
已经真正进入 RAG。
```

例如批处理脚本可能处理 100 个来源：

```text
脚本正常结束
→ 99 条成功
→ 当前这 1 条失败
→ exit code 仍然可能是 0
```

所以 Gateway 会在关键阶段后做**业务产物核验**：

```text
抓取后
→ MySQL 中该 source 的抓取状态是否正确

正文校验后
→ source 是否变成 validated 或 needs_manual_review

抽卡后
→ cards_draft.jsonl 是否真的出现该 source_id 的卡片

卡片校验后
→ cards_validated.jsonl 是否真的出现该 source_id 的正式卡
```

这里：

- `source_id`：一条来源的稳定唯一编号；
- `cards_draft.jsonl`：LLM 初步抽出的草稿卡；
- `cards_validated.jsonl`：通过规则和语义校验后的正式卡；
- `artifacts`：某次任务实际生成或影响的文件、卡片、索引等产物。

你可以这样说：

> 我们不只判断“进程是否成功”，还判断“当前来源是否真的生成了预期业务产物”，避免出现脚本执行成功但资料未进入知识库的假成功。

---

## 8. Gateway 与 Docker Compose 的关系

在 Docker Compose 中，Gateway 相关能力会被拆成三个运行角色：

```text
app
→ 运行 HTTP API 和 HTTP MCP

worker
→ 消费 MySQL Job，执行后台长任务

webapp
→ 提供人工管理来源、知识卡的页面
```

它们使用同一个镜像：

```text
perfdog-workspace-gateway
```

但通过不同 `command` 启动不同程序。

好处：

```text
同一套源码
+ 同一套 Python 依赖
+ 同一套模型缓存
+ 不需要维护三份 Dockerfile
```

完整 Compose 同时启动：

```text
MySQL
Qdrant
Gateway app
Worker
Webapp
```

职责分别是：

| 组件 | 作用 |
|---|---|
| MySQL | 保存来源、Job、lease、发布记录、卡片管理索引 |
| Qdrant | 保存向量索引，支持向量检索 |
| Gateway app | 提供 REST 与 HTTP MCP |
| Worker | 后台执行长 Job |
| Webapp | 提供人工管理界面 |

---

# 第二部分：Gateway 的补充功能与边界

## 9. Gateway 如何启动、定位项目、做健康检查？

Gateway 不是写死路径启动的，而是先通过 `bootstrap.py` 自动找到整个工作区的结构：

```text
perfdog_knowledge_datasets/
├─ perfdog_service_gateway/
├─ perfdog_dual_rag/
└─ perfdog_llm_wiki_starter/
```

代码会根据 `bootstrap.py` 自己所在的位置，反推出：

```text
WORKSPACE_ROOT
→ 整个工作区根目录

GATEWAY_ROOT
→ perfdog_service_gateway

DUAL_RAG_ROOT
→ perfdog_dual_rag

WIKI_STARTER_ROOT
→ perfdog_llm_wiki_starter
```

这样做的目的：

> 无论你从哪个当前目录启动 Gateway，它都能定位到两个兄弟项目，而不是依赖命令行运行时所在的目录。

之后，`ensure_workspace_sys_path()` 会把：

```text
perfdog_service_gateway/src
perfdog_dual_rag/src
```

加入 Python 的 `sys.path`。

`sys.path` 可以理解成 Python 的“模块搜索目录列表”。加入后，Gateway 才能直接：

```python
import perfdog_dual_rag
```

而 Wiki Starter 是脚本型项目，不直接作为 Python 包导入，因此仍通过受控子进程调用。

### 9.1 健康检查做什么？

Gateway 提供：

```text
GET /healthz
GET /api/workspace/overview
GET /api/dual-rag/health
GET /api/wiki-starter/status
```

其中工作区总健康检查会检查三类信息：

```text
1. 工作区目录是否完整
2. Dual RAG 是否可用
3. Wiki Starter 的关键资产是否存在
```

例如会检查：

```text
工作区根目录是否存在
Gateway 目录是否存在
Dual RAG 根目录与 src 是否存在
Wiki Starter 根目录是否存在
```

同时，Dual RAG 健康检查还会确认：

```text
build_report.json 是否存在
rag_docs.jsonl 是否存在
本地 Qdrant 目录是否存在
是否配置远程 Qdrant URL
```

这里要理解几个概念：

- `build_report.json`：最近一次构建 RAG 语料时生成的报告，记录输入、输出、统计信息；
- `rag_docs.jsonl`：RAG 实际消费的结构化文档集合；
- `Qdrant`：向量检索库；
- `qdrant_url`：若配置，表示使用远程 Qdrant 服务；
- `qdrant_path`：若未走远程服务，则是本地 Qdrant 数据目录。

所以健康检查不是只回答“HTTP 服务进程还活着吗”，而是回答：

> 工作区目录、上游知识资产、RAG 语料和检索后端是否具备可运行条件。

---

## 10. Gateway 如何给下游 Agent 提供 `Agent Context`？

这一点很重要。

Gateway 不只提供：

```text
“帮我搜几篇相关文档”
```

还提供：

```text
“把检索到的知识整理成下游 Agent 可直接使用的证据上下文”
```

入口是：

```text
POST /api/agent-context
POST /api/dual-rag/agent-context
```

MCP 对应工具是：

```text
perfdog_build_agent_context
```

调用链是：

```text
HTTP / MCP 请求
→ Gateway runtime
→ dual_rag adapter
→ Dual RAG service_facade
→ 查询理解
→ local 或 Qdrant 检索
→ 规则精排
→ final_reporter
→ agent_context
```

### 10.1 检索请求如何结构化？

调用方不是只传一句自然语言，而是可以传结构化条件，例如：

```json
{
  "platform": "Android",
  "engine": "Unity",
  "metrics": ["avgFps", "cpuAvgPct"],
  "symptoms": ["持续掉帧", "CPU 占用偏高"],
  "anti_signals": ["GPU 未持续满载"]
}
```

这些字段的含义是：

- `platform`：问题发生的平台，例如 Android、iOS、Windows；
- `engine`：业务或游戏使用的引擎，例如 Unity、Unreal；
- `metrics`：用户已观测到的具体性能指标；
- `symptoms`：现象描述，例如掉帧、卡顿、内存持续上涨；
- `anti_signals`：反向证据，即“理论上如果某根因成立应出现、但实际没有出现”的信号。

例如：

```text
症状：掉帧
CPU：高
GPU：未满载
```

这类输入会让检索与精排更倾向于 CPU、主线程、脚本逻辑、GC 等方向的知识卡，而不是盲目把 GPU 当根因。

### 10.2 `agent_context` 与普通检索结果有什么区别？

普通检索接口更接近：

```text
返回哪些知识卡命中
→ 每张卡的分数、来源、摘要、调试信息
```

而 `agent_context` 更接近：

```text
把命中的卡片按“现象、根因、验证、优化、风险”等角色整理好
→ 形成一份适合 LLM / Agent 阅读的证据包
```

可以把它理解为：

```text
普通检索
= 搜索引擎结果页

Agent Context
= 给诊断 Agent 准备好的案卷材料
```

最终 Agent 可以结合：

```text
当前性能报告里的真实指标
+ Gateway 返回的历史知识证据
+ 自己的推理能力
```

形成诊断结论。

但一定要准确表达：

> Gateway 和 Dual RAG 提供的是“结构化证据与上下文”，不是 Gateway 自己已经自动做出了最终根因裁决。

也就是说，Gateway 不应被夸大成：

```text
输入 FPS 低
→ Gateway 自动百分之百确认根因
```

真实能力是：

```text
输入结构化症状和指标
→ 返回相关、可追溯、有反证信息的知识证据
→ 下游 Agent 或人类再结合当前测量事实完成归因
```

---

## 11. Gateway 如何管理知识卡？

除了抓取和检索，Gateway 还有两项重要的知识治理能力：

```text
删除知识卡
切换知识卡是否进入 RAG
```

对应接口：

```text
POST /api/wiki-starter/delete-cards
POST /api/wiki-starter/set-rag-visibility
```

对应 MCP Tool：

```text
perfdog_wiki_starter_delete_cards
perfdog_wiki_starter_set_rag_visibility
```

### 11.1 删除知识卡：`delete_cards`

删除可以按两种粒度进行：

```text
按 card_id 删除一张或多张卡
按 source_id 删除某条来源下的全部卡
```

这里：

- `card_id`：单张知识卡的稳定唯一编号；
- `source_id`：一条原始资料的稳定唯一编号；
- 一个 `source_id` 通常可以对应多张 `card_id`。

例如一篇“Unity 性能优化文章”可能拆出：

```text
一张现象卡
一张根因卡
一张验证卡
一张优化卡
```

它们属于同一个 `source_id`，但分别有不同 `card_id`。

删除链路是：

```text
delete_cards
→ 修改 cards_draft.jsonl
→ 修改 cards_validated.jsonl
→ 得到 affected_source_ids
→ 对受影响来源增量重渲染 Wiki
→ 可选重建 RAG
```

其中：

- `cards_draft.jsonl`：LLM 初步抽取出的草稿卡；
- `cards_validated.jsonl`：通过规则与语义校验的正式卡；
- `affected_source_ids`：此次删除影响到的来源；
- **增量重渲染**：不重建全部 Wiki，只更新被影响来源对应的页面。

为什么删除后要重渲染 Wiki？

因为如果卡片已经删掉，但 Wiki 页面还保留原内容，就会产生：

```text
JSONL 里没有这张卡
但 Wiki 页面还展示它
```

这种数据不一致。

因此 Gateway 的删除 pipeline 会自动：

```text
先删卡
→ 再清理旧 Wiki 页面
→ 按剩余卡片重新渲染受影响来源
```

如果一个来源下的卡片都被删光，它的 Wiki 页面自然不会再出现。

### 11.2 `rag_excluded`：不删除，但不让它参与检索

有些卡片不一定要物理删除。

例如：

```text
内容保留给人工审阅
Wiki 中仍然希望展示
但暂时不希望它影响线上 RAG 检索
```

此时使用：

```text
set_rag_visibility
```

它本质是在卡片上切换：

```text
rag_excluded = true / false
```

含义：

```text
rag_excluded=true
→ 卡片仍然保留
→ Wiki 仍可展示
→ 但后续构建 rag_docs.jsonl 时跳过它
→ 因此不会被 RAG 检索召回

rag_excluded=false
→ 撤销排除
→ 卡片恢复进入后续 RAG 构建范围
```

这个设计很有价值，因为它把：

```text
“内容是否存在”
```

和：

```text
“内容是否参与线上检索”
```

分开了。

你可以这样讲：

> 对风险知识卡，我们不一定立即删除，而是用 `rag_excluded` 作为上线开关。这样保留审计和 Wiki 可见性，但避免未经确认的知识污染检索结果。

### 11.3 为什么默认会重建 RAG？

`set_rag_visibility` 默认 `rebuild_rag=true`。

原因是只改 JSONL 中的 `rag_excluded` 字段还不够：

```text
卡片字段已变
但旧 rag_docs.jsonl 还保留它
甚至 Qdrant 旧索引还保留它
```

所以必须重新构建，改动才能进入检索结果。

不过默认的 `rebuild_mode` 是：

```text
corpus
```

即：

```text
只重建 rag_docs.jsonl
不立即重建更耗时的 Qdrant 向量库
```

如果需要同时更新向量索引，才使用：

```text
rebuild_mode=full
```

### 11.4 `cards_index` 是什么？它不是 Qdrant

Gateway 还会把知识卡建立到 MySQL 的 `cards_index` 表中。

注意：

> `cards_index` 不是向量索引，也不是 Qdrant。

它是为了 Web 管理、分页、筛选和简单文本查询建立的关系型索引。

它大致保存：

```text
dataset
release_id
card_id
source_id
card_type
title
canonical_key
review_status
rag_excluded
platforms_text
search_text
card_json
```

简单理解：

- `dataset`：这张卡来自草稿集还是正式卡集；
- `release_id`：这张卡属于哪个发布版本；
- `card_type`：卡片类型，例如根因、现象、验证、优化；
- `review_status`：是否需要人工审查；
- `platforms_text`：平台字段拼接出的可 SQL 查询文本；
- `search_text`：标题、摘要、来源等拼成的普通文本搜索字段；
- `card_json`：完整的原始卡片 JSON。

它服务的场景是：

```text
Web 页面列出卡片
按平台筛选
按来源筛选
按卡片类型筛选
按 review status 筛选
分页展示
```

而 Qdrant 服务的是：

```text
语义向量检索
```

两者不要混为一谈。

---

## 12. Gateway 如何构建 RAG 并发布版本？

入口是：

```text
POST /api/dual-rag/build-assets
```

MCP Tool：

```text
perfdog_dual_rag_build_assets
```

这个任务有三种构建模式：

```text
corpus
qdrant
full
```

### 12.1 三种构建模式的区别

#### `corpus`

```text
cards_validated.jsonl + Wiki
→ 构建 rag_docs.jsonl
```

只生成 RAG 文档，不建立或更新向量库。

适合：

```text
刚修改了卡片内容
先更新语料
暂时不急着更新 Qdrant
```

#### `qdrant`

```text
已有 rag_docs.jsonl
→ 只建立或更新 Qdrant 向量索引
```

适合：

```text
语料已经正确
只是想重新向量化
或者切换 embedding 模型、collection、Qdrant 地址
```

#### `full`

```text
cards + Wiki
→ rag_docs.jsonl
→ embedding
→ Qdrant 向量库
```

适合：

```text
从头完整构建
一批来源或卡片更新后统一发布
```

### 12.2 `build-assets` 内部做了什么？

主要调用链：

```text
Gateway Job
→ build_dual_rag_assets()
→ build_rag_corpus()
→ build_qdrant_index()
→ 可选 publish_asset_release()
```

其中：

#### `build_rag_corpus()`

负责将上游知识资产转换为：

```text
rag_docs.jsonl
```

并默认排除：

```text
review_status = needs_manual_review
```

除非调用时显式允许：

```text
include_needs_manual_review=true
```

这体现了知识治理原则：

> 未人工确认的卡默认不进入 RAG，避免它影响后续 Agent 的证据基础。

#### `build_qdrant_index()`

负责：

```text
读取 rag_docs.jsonl
→ 用 dense / sparse embedding 模型向量化
→ 写入 Qdrant collection
```

这里：

- `dense_model`：稠密向量模型，理解文本语义相似性；
- `sparse_model`：稀疏向量模型，更适合保留关键词、指标名、术语精确匹配；
- `collection_name`：Qdrant 里的一个索引集合名称；
- `batch_size`：每次批量向量化和写入多少条文档，默认 16；
- `qdrant_url`：远程 Qdrant 服务地址；
- `qdrant_path`：本地 Qdrant 的数据路径。

### 12.3 什么是 Release？为什么要发布？

构建成功不一定等于“新资产已经被线上检索使用”。

因此 Gateway 引入 `asset_releases` 表记录：

```text
当前哪一套 cards、Wiki、rag_docs、Qdrant 配置是 active
```

`release` 可以理解为：

> 一次可追踪的“当前生效知识资产版本”。

例如一次发布会记录：

```text
release_id
cards_path
wiki_path
rag_docs_path
build_report_path
qdrant_path / qdrant_url
collection_name
metadata
created_by
published_at
```

其中：

- `release_id`：版本编号，自动生成时类似 `rel-时间戳`；
- `status=active`：当前线上正在使用；
- `status=retired`：已被新版本替换；
- `metadata`：记录构建模式、发布原因、构建报告等扩展信息；
- `created_by`：哪个模块发起发布，例如 `gateway:dual_rag.build_assets`。

发布时逻辑是：

```text
先把旧 active release 改成 retired
→ 写入新 release
→ 新 release 标记为 active
```

之后 Gateway 的检索、健康检查等能力可以优先使用 active release 中保存的路径和 Qdrant 配置。

### 12.4 一个很容易讲错的点

要区分：

```text
Job.publish_release
```

和：

```text
result.published_release
```

前者只是：

> 这个 Job 有发布意图。

后者才表示：

> 实际已经得到发布结果。

特别是 `dual_rag.build_assets` Job：

- Job 入队时，Job 表里的 `publish_release` 默认可能是 `true`；
- 但真正调用 `build_dual_rag_assets()` 时，是否发布仍由 payload 里的 `publish_release` 决定；
- 如果 payload 没有带这个字段，adapter 内部默认是 `false`。

所以不能只看 Job 记录里的发布标记。

正确判断是：

```text
任务是否真的发布成功
→ 看 job.status 是否为 succeeded
→ 再看 job.result.published_release 是否非空
```

### 12.5 Wiki 任务后为什么还可能自动触发 full 发布？

对这些成功的写任务：

```text
wiki.ingest_source
wiki.delete_cards
wiki.set_rag_visibility
```

Worker 会在任务执行成功后，根据 Job 的发布意图，触发一次 follow-up：

```text
build_mode=full
publish_release=true
```

即：

```text
Wiki / 卡片变更
→ 再重建 rag_docs
→ 再重建 Qdrant
→ 再发布为 active release
```

目的是让知识更新后，检索和 Agent Context 尽快使用新资产。

但也要知道一个边界：

> 直接在 Python 内调用 `run_ingest_source_pipeline()` 时，`rebuild_rag` 默认仍是 `false`；只有走 Gateway 的 Job + Worker 编排、且满足 follow-up 发布条件时，才会追加完整发布。

因此面试时要先说清调用面：

```text
直接调用 Adapter
还是
通过 HTTP / MCP → Job → Worker
```

不能笼统说“新增资料一定自动更新 Qdrant”。

---

## 13. Gateway 的错误处理与协议边界

### 13.1 普通 REST HTTP 错误

Gateway 的 REST 层主要映射：

```text
400 Bad Request
→ 参数或 JSON 格式错误

404 Not Found
→ URL 路由不存在

500 Internal Server Error
→ 未捕获的服务执行异常
```

例如：

```text
请求体不是 JSON 对象
→ 400

访问不存在的 /api/xxx
→ 404

下游执行时抛未处理异常
→ 500
```

当前返回的业务错误主体一般是：

```json
{
  "success": false,
  "error": "可读错误文本"
}
```

这对于人工调试足够，但当前还没有：

```text
稳定 error_code
trace_id
统一错误分类
```

因此生产化后应补：

```text
机器可读错误码
请求追踪 ID
统一异常中间件
结构化日志
```

### 13.2 MCP 错误和普通业务失败不完全一样

MCP 使用 JSON-RPC 2.0。

JSON-RPC 的意思是：客户端和服务端通过 JSON 表达“调用方法、传参数、返回结果或错误”。

MCP 中有两类错误：

#### 第一类：协议错误

例如：

```text
不是 jsonrpc: "2.0"
缺少 Mcp-Session-Id
MCP 协议版本不匹配
调用未知 method
JSON 不能解析
```

这类问题会走 JSON-RPC 的 `error`：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "请求格式非法"
  }
}
```

常见 code：

```text
-32700：JSON 解析失败
-32600：请求格式非法
-32601：调用的方法不存在
-32603：服务内部错误
```

#### 第二类：工具业务失败

例如：

```text
调用了不存在的 Tool
检索参数不合法
脚本执行超时
没有配置 OPENAI_API_KEY
RAG 构建失败
```

这时 JSON-RPC 协议本身可能仍然成功，只是 Tool Result 中：

```text
isError=true
structuredContent.success=false
```

这里：

- `content`：给人类或普通 MCP Client 看的格式化文本；
- `structuredContent`：给程序读取的结构化业务 JSON；
- `isError`：该 Tool 的业务调用是否失败。

可以这样记：

```text
JSON-RPC error
= “你连协议都没有按要求说话”

Tool isError
= “协议没问题，但你调用的业务任务失败了”
```

### 13.3 Wiki 子进程失败如何处理？

Gateway 启动 Wiki 脚本时会记录：

```text
command
cwd
exit_code
timed_out
duration_seconds
stdout
stderr
artifacts
```

这些字段的作用：

- `command`：最终运行的命令；
- `cwd`：子进程工作目录；
- `exit_code`：脚本退出码，`0` 通常表示进程未异常退出；
- `timed_out`：是否超过 Gateway 设定的最长执行时间；
- `duration_seconds`：脚本执行耗时；
- `stdout`：正常输出日志；
- `stderr`：错误输出日志；
- `artifacts`：任务生成或检查到的文件/记录摘要。

而且 Gateway 会提前检查部分前置条件。

例如：

```text
extract_cards
→ 必须配置 OPENAI_API_KEY

validate_cards 且 semantic_mode=required
→ 也必须配置 OPENAI_API_KEY
```

如果缺失，会在启动子进程之前返回失败，而不是跑到一半才报错。

---

## 14. Gateway 的安全边界、当前限制与生产化方向

### 14.1 当前已经实现的安全控制

#### 任务白名单

外部调用方不能传任意 shell 命令。

只能传预定义的：

```text
add_source
scrape_sources
fetch_video_transcripts
validate_scrapes
extract_cards
validate_cards
render_wiki
delete_cards
set_rag_visibility
```

好处：

> 避免把 HTTP 接口变成任意远程命令执行入口。

#### 参数枚举和类型归一

例如：

```text
build_mode 只能是 corpus / qdrant / full
semantic_mode 只能是 auto / required / off
priority 只能是 P0 / P1
```

不符合就拒绝，而不是把任意字符串透传给下游。

#### 超时与输出截断

- `timeout_seconds`：限制脚本最多运行多久；
- `max_output_chars`：限制返回日志长度；
- `_TASK_RUN_LOCK`：同一 Gateway 进程内，Wiki 脚本任务串行执行；
- MySQL `single-writer lock`：多个 Worker 间也避免并发修改共享资产。

### 14.2 当前有限实现，不能夸大

#### 没有鉴权与权限控制

当前 HTTP 接口没有：

```text
JWT
API Key
OAuth
RBAC
```

也就是说，暴露到不可信网络前必须加安全层。

#### 没有请求幂等键

幂等键的意思是：

> 客户端因为超时重试同一请求时，系统能识别“这是同一件事”，不重复创建多个 Job。

当前没有 `Idempotency-Key`。

因此如果调用方连续提交两次：

```text
POST /api/wiki-starter/ingest-source
```

可能生成两个独立 Job。

虽然单写锁会让它们不同时写资产，但不会自动消除重复任务。

#### 没有统一自动重试策略

当前：

```text
Worker 崩溃
→ lease 到期
→ Job 可以重新变为 queued
```

这是“Worker 故障恢复”。

但不等同于：

```text
Firecrawl 429 自动指数退避重试
LLM 服务失败自动重试
所有 Job 按统一策略多次重跑
```

外部服务的重试逻辑主要在具体 Wiki 脚本里；Gateway 没有统一的 Job retry policy。

#### 没有事件流或回调

当前调用方主要只能：

```text
不断 GET /api/jobs/<job_id>
```

轮询状态。

尚未提供：

```text
SSE 实时进度流
WebSocket
任务完成回调 webhook
阶段级事件日志
```

#### HTTP MCP Session 只在单进程内存中

当前 HTTP MCP 会话保存在 Python 字典里。

因此：

```text
Gateway 重启
→ session 丢失

多副本 Gateway
→ 请求被负载均衡到另一副本
→ 找不到原 session
```

生产化需要：

```text
共享 Redis session store
或负载均衡粘性会话
```

#### 本地 Qdrant 不适合多副本共享

本地 Qdrant 文件模式需要进程内锁保护。

它适合：

```text
单机
开发环境
单实例部署
```

不适合：

```text
多个 Gateway 副本同时访问同一块共享文件系统
```

生产环境更适合：

```text
独立远程 Qdrant 服务
```

---

# 最终完整面试表达

> 我们在两个子工程之上实现了一个 Workspace Gateway，作为统一服务与编排层。协议层同时支持 REST、stdio MCP 和 HTTP MCP，但它们都复用 `runtime.py` 的统一分发逻辑，避免多协议实现漂移。对 Dual RAG 这种包化能力，Gateway 直接调用 Python 函数，负责检索、构建 Agent Context、RAG 语料和 Qdrant 索引；对脚本型 Wiki Starter，则通过任务白名单、固定参数、超时和受控子进程进行编排。
>
> 在写链路上，抓取、LLM 抽卡、卡片校验、Wiki 渲染和向量建库不会阻塞 HTTP，而是先写入 MySQL Job 队列，由 Worker 通过 300 秒 lease、60 秒 heartbeat 和 MySQL single-writer lock 异步串行执行。除了检查脚本退出码，我们还会检查 source 的抓取状态、JSONL 是否真实生成卡片等业务产物，避免假成功。
>
> Gateway 还支持按卡或来源删除知识、通过 `rag_excluded` 控制知识是否进入 RAG、构建并发布 active release。它当前已具备单机或共享环境的核心编排能力，但尚未实现鉴权、幂等键、事件流、统一 Job 自动重试、共享 MCP session 和多副本 Qdrant 方案；这些是后续生产化的主要方向。
