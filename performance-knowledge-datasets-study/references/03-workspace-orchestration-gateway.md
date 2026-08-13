# 3. `workspace_orchestration_gateway`：统一服务、协议适配与任务执行

## 3.1 服务化状态标记

- **[当前代码已实现]**：REST 写路由入 MySQL Job；Worker claim/lease/heartbeat/单写锁；HTTP/stdio MCP；active release 与 follow-up 发布。
- **[当前有限实现]**：Job 通过轮询查询，不含回调/事件流、请求幂等键或统一失败自动重试；HTTP MCP session 仅进程内；无认证/RBAC。
- **[推荐生产化演进]**：分队列 Worker、指数退避、Idempotency-Key、trace/event、共享 session store、远端 Qdrant。这些不是当前已上线行为。

字段、状态机和 API 失败合同统一查 `09-api-mcp-job-contract.md`；术语查 `10-glossary-and-state-labels.md`。

## 3.2 服务化策略

Gateway 不重复实现知识构建或 RAG 算法，而是把已有两类资产暴露成一致服务：


- `hybrid_retrieval`：已包化，有明确 Python API，adapter 直接函数调用；
- `asset_pipeline`：编号脚本型、没有稳定 service API，adapter 通过任务白名单构造固定命令，以子进程执行。

这是一种渐进式重构：避免为了接口化而大规模侵入已有 ETL 脚本，同时避免把 HTTP 请求直接拼成任意 shell 命令。

## 3.2 启动与 bootstrap

- `bootstrap.py` 从自身路径推导工作区根，将编排网关与 `hybrid_retrieval_service` 的 `src` 放进 `sys.path`。

- `serve_api.py` 启动 HTTP 服务，并预热 dense/sparse embedder，避免首个检索请求承担 ONNX/模型冷启动。
- `serve_worker.py` 启动轮询 Worker，默认约每 2 秒 claim job。
- `serve_mcp.py` 启动 stdio MCP。

## 3.4 Runtime：唯一业务装配层


`runtime.py` 是 HTTP 与 MCP 共用的服务层：

- 聚合 workspace health 与 overview；
- 处理混合检索的 retrieve / agent context / build assets；
- 调用 asset pipeline adapter 的 status、单任务和编排流程；
- 定义 MCP 的 9 个 tool schema，使 REST 与 MCP 不会各自演进出不同业务逻辑。

九个工具：

1. `knowledge_workspace_health`
2. `retrieve_knowledge`
3. `build_agent_context`
4. `hybrid_retrieval_build_assets`
5. `asset_pipeline_get_status`
6. `asset_pipeline_run_task`
7. `asset_pipeline_ingest_source`
8. `asset_pipeline_delete_cards`
9. `asset_pipeline_set_rag_visibility`

## 3.5 HTTP：`api.py`


基于标准库 `ThreadingHTTPServer`，不依赖 Flask/FastAPI：

- JSON body 解析与响应序列化；
- CORS；
- 路由分发；
- 错误映射为 HTTP status；
- REST 和远程 HTTP MCP 共用同一进程。

主要路由：

```text
GET  /healthz
GET  /api/workspace/overview
GET  /api/hybrid-retrieval/health
GET  /api/asset-pipeline/status
POST /api/retrieve                         # hybrid_retrieval 兼容路由
POST /api/agent-context                    # hybrid_retrieval 兼容路由
POST /api/hybrid-retrieval/retrieve
POST /api/hybrid-retrieval/agent-context
POST /api/hybrid-retrieval/build-assets
POST /api/asset-pipeline/run-task
POST /api/asset-pipeline/ingest-source
POST /api/asset-pipeline/delete-cards
POST /api/asset-pipeline/set-rag-visibility
POST /mcp
DELETE /mcp
```

## 3.6 MCP：`mcp.py` + HTTP `/mcp`


### stdio MCP

- 使用 `Content-Length` framing 收发 JSON-RPC；
- 处理 `initialize`、`tools/list`、`tools/call`；
- 直接复用 runtime，而不是另写一套工具实现。

### HTTP MCP

- `POST /mcp` 接收单条 JSON-RPC；
- `initialize` 后返回 `Mcp-Session-Id`；后续带 session id 和协商的 protocol version；
- `DELETE /mcp` 显式清理；
- 不提供 SSE，因此 `GET /mcp` 返回 405。

限制：会话保存在进程内存。多实例负载均衡若不做粘滞会话，会出现 session not found；生产应使用共享 session store 或把 MCP gateway 固定单副本。

## 3.6 `adapters/hybrid_retrieval.py`

职责：

- 解析默认路径、active release；
- 调用 `build_rag_corpus`、`build_qdrant_index`、检索和报告函数；
- 提供 `corpus` / `qdrant` / `full` build mode；
- 组合发布相关元数据。

优点：无需 subprocess，异常和返回值可保留 Python 结构。风险：与包内部 API 有耦合，需用集成测试约束版本升级。

## 3.7 `adapters/asset_pipeline.py`

### 安全与可控性

- 定义允许任务白名单，HTTP 输入的 task 不能变成任意脚本路径。
- 对每个 task 建固定 command builder，只允许预定义参数映射。
- 子进程有 timeout；stdout/stderr 截断，防止日志无限塞进 API 响应。
- 对关键任务检查 `.env` 所需 key 等 prerequisite，尽早给出可理解错误。

### Pipeline 编排

- `run_asset_pipeline_task`：执行一个白名单任务；
- `run_ingest_source_pipeline`：串 add→fetch→validate→extract→validate cards→render；每阶段检查真实产物；默认 fail-fast；
- `run_delete_cards_pipeline`：删除后重渲染受影响 Wiki；
- `run_set_rag_visibility_pipeline`：切换 `rag_excluded` 后默认重建语料。

这里的核心工程点：**exit code 0 不代表业务成功**。adapter 会核验 source/card/文件等阶段产物，阻断“脚本正常退出但没有生成结果”的假成功。

## 3.8 异步 Job：`workspace_jobs.py`

写操作或资产构建可能长、会修改共享目录，网关引入 MySQL job 表与 Worker：

1. API 提交 job；
2. Worker claim 未处理 job；
3. 续租 lease，防止 Worker 崩溃后永久占有；
4. 获取全局写锁，串行执行会改 cards/wiki/rag/qdrant 的任务；
5. 成功/失败及错误落库；
6. 刷新卡片索引、发布资产。

读检索可以同步；写/构建更适合异步 job。注意仓库文档的早期“同步 HTTP 长任务”描述可能和当前 Worker 机制并存，面试应说明：**应以代码中的 API→Job→Worker 流程为准，直接同步编排仍有超时风险。**

## 3.9 典型故障排查

| 现象 | 首查点 | 原因模型 |
| --- | --- | --- |
| 任务提交后无结果 | job status、Worker 日志、lease | Worker 未启动/不健康/未 claim |
| ingest 成功但检索不到 | `rebuild_rag`、`rag_excluded`、build report | 默认不重建或可见性关闭 |
| Qdrant lock 错误 | 并发 build/retrieve、容器副本数 | 本地目录模式冲突 |
| MCP session not found | request headers、负载均衡 | 进程内 session 未粘滞 |
| 子进程“成功”但无卡片 | adapter stage verification | 上游脚本没有产出/筛选掉来源 |

## 3.10 安全与扩展

当前没有鉴权，且有 delete/build 等写工具。生产最小改造：

- API gateway / OAuth/JWT 或内部 token；
- RBAC：只读检索与运维写操作分权；
- 对 URL 抓取做 SSRF allowlist/denylist、速率和配额；
- 将现有可查询 Job 状态扩展为阶段级进度、事件流、回调和审计日志；

- Qdrant 替换为远端集群，使用外部锁/队列支撑多副本。
