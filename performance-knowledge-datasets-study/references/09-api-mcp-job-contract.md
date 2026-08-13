# 9. API、MCP 与 Job 合同：字段、状态机、失败与幂等边界

> **状态标记**
>
> - **[当前代码已实现]**：以下字段、状态和行为来自当前 `runtime.py`、`api.py`、`workspace_jobs.py`、`workspace_store.py`。
> - **[当前有限实现]**：有可调用行为，但缺少请求去重、鉴权、跨副本 session 等生产能力。
> - **[推荐生产化演进]**：不是当前实现，不能表述为已经上线。

## 9.1 统一 REST 响应和 HTTP 错误

### 读接口成功响应

`retrieve` / `agent-context` / health 等同步接口均以 `success: true` 为顶层成功标记，并带 `service` 或 `gateway_service` 元信息。具体业务结果在 `package`、`agent_context`、`components`、`job` 或 `release` 中。

### 写接口成功响应：不是任务结果，而是“已入队”

`POST /api/hybrid-retrieval/build-assets`、`run-task`、`ingest-source`、`delete-cards`、`set-rag-visibility` 的当前 HTTP 响应统一近似为：

```json
{
  "success": true,
  "service": {"name": "knowledge_workspace_gateway", "version": "workspace-api-mcp-v1", "workspace_root": "..."},
  "operation": "asset_pipeline.ingest_source",
  "queued": true,
  "job": {"job_id": "UUID", "job_type": "asset_pipeline.ingest_source", "status": "queued", "payload": {}, "result": {}, "error": "", "requested_by": "...", "worker_id": "", "attempts": 0, "publish_release": true, "created_at": "...", "updated_at": "...", "started_at": null, "finished_at": null, "lease_expires_at": null}
}
```

**[当前代码已实现]** HTTP `200` + `queued=true` 只说明 MySQL 已插入 job，绝不等于资料已抓取、卡片已生成或索引已更新。必须继续 `GET /api/jobs/<job_id>`。

### REST 层错误形态

| 情况 | HTTP | body 保证字段 |
| --- | --- | --- |
| 路由不存在 | `404` | `success:false`, `error` |
| 缺少 `job_id` / JSON 无法解析 / body 非 object / `ValueError` | `400` | `success:false`, `error` |
| 未捕获服务异常 | `500` | `success:false`, `error`，前缀通常为“服务执行失败” |
| Job 不存在 | 当前仍返回 `200` | `success:false`, `operation:"workspace.job_status"`, `job_id`, `error` |

**[当前有限实现]** REST 响应没有稳定的 machine-readable `error_code`、`trace_id`、字段级错误数组或 RFC 7807 problem details；调用方目前应以 HTTP status、`success` 和中文 `error` 联合判断。

## 9.2 REST 路由与同步/异步语义

| 路由 | 当前执行模型 | 请求主体 | 响应主体 |
| --- | --- | --- | --- |
| `GET /healthz`、`GET /api/workspace/overview` | 同步 | 无 | workspace health/components |
| `GET /api/hybrid-retrieval/health` | 同步 | 无 | 混合检索健康状态 |
| `GET /api/asset-pipeline/status` | 同步 | 无 | 知识资产构建层状态 |
| `GET /api/releases/active` | 同步 | 无 | active release |
| `GET /api/jobs/<job_id>` | 同步 DB 查询 | path job id | Job 对象 |
| `POST /api/retrieve`、`/api/hybrid-retrieval/retrieve` | 同步检索 | 检索合同 | `package` |
| `POST /api/agent-context`、`/api/hybrid-retrieval/agent-context` | 同步检索+组装 | 检索合同 | `agent_context` |
| `POST /api/hybrid-retrieval/build-assets` | **异步入队** | build 合同 | queued Job |
| `POST /api/asset-pipeline/run-task` | **异步入队** | task 合同 | queued Job |
| `POST /api/asset-pipeline/ingest-source` | **异步入队** | ingest 合同 | queued Job |
| `POST /api/asset-pipeline/delete-cards` | **异步入队** | delete 合同 | queued Job |
| `POST /api/asset-pipeline/set-rag-visibility` | **异步入队** | visibility 合同 | queued Job |

## 9.3 检索 / Agent Context 合同

适用于 REST `retrieve` / `agent-context`，也适用于 MCP `retrieve_knowledge` / `build_agent_context`。

### 完整输入字段

| 字段 | 类型/默认 | 说明 |
| --- | --- | --- |
| `project_root` | string，可选 | `hybrid_retrieval_service` 根目录 |
| `backend` | `local` / `qdrant`，默认 `qdrant` | 显式选后端；不自动失败切换 |
| `platform`、`engine` | string | 目标平台/引擎 |
| `metrics`、`symptoms`、`anti_signals` | string[] 或分隔字符串 | 查询证据 |
| `card_types` | string[] | 缺省为现象/根因/验证/优化四类 |
| `card_priorities`、`review_statuses`、`exclude_warnings` | string[] | 过滤条件；review 默认 human_reviewed/auto_checked/draft |
| `strict_platform`、`strict_engine` | bool，默认 false | 严格不匹配直接排除 |
| `top_k` | int，默认 3 | 每卡类型条数 |
| `rag_docs_path`、`build_report_path`、`cards_path` | string | 显式数据资产路径 |
| `qdrant_path`、`qdrant_url`、`collection_name` | string | 本地或远程索引定位 |
| `candidate_limit` | int，默认 128 | Qdrant 一阶段候选池 |
| `fusion` | `rrf` / `dbsf`，默认 `dbsf` | hybrid fusion |
| `dense_only` | bool，默认 false | 禁用 sparse 分支 |
| `dense_model`、`sparse_model` | string | 显式模型覆盖 |
| `rerank_mode` | `none` / `local_hybrid` | 默认领域规则精排 |
| `report_title` | string | 仅 package 标题 |
| `evidence_limit` | int，默认 3 | 每卡来源摘录上限 |
| `include_debug` | bool，默认 false | 是否返回 resolved inputs/原始结果 |

### 成功输出

- `retrieve`：`success`、`service`、`operation:"retrieve_knowledge"`、`package`；默认 `package` 只含 `report_meta`、`query`、`summary`、`sections`。
- `agent-context`：`success`、`service`、`operation:"build_agent_context"`、`agent_context`；`include_debug=true` 时额外返回完整 package。
- Gateway 外层额外附 `gateway_service`、`gateway_component`。

## 9.4 写操作请求合同

### A. `POST /api/asset-pipeline/ingest-source`

**必填：**

```json
{"url": "https://example.com/article"}
```

**完整可用请求体：**

```json
{
  "url": "https://example.com/article",
  "title": "可选标题提示",
  "section_hint": "Unity 渲染",
  "enabled": "Y",
  "priority": "P0",
  "include_in_wiki": "Y",
  "title_override": "人工覆盖标题",
  "mapped_metrics_override": "avgFps,low1pct",
  "notes": "收录原因",
  "updated_by": "user@example",
  "asr_fallback": "auto",
  "asr_model": "gpt-4o-mini-transcribe",
  "asr_language": "zh",
  "asr_max_audio_mb": 20,
  "continue_on_error": false,
  "timeout_seconds": 0,
  "max_output_chars": 12000,
  "rebuild_rag": false,
  "rebuild_mode": "corpus"
}
```

字段分组：来源/人工编目（`title` 到 `updated_by`）、视频 ASR（`asr_*`）、pipeline 行为（`continue_on_error`、timeout/log）和直接 adapter 重建选择（`rebuild_*`）。`url` 是 schema 中唯一 required 字段。

**重要的当前行为分层：**

- **[当前代码已实现：直接 adapter]** `run_ingest_source_pipeline` 默认 `rebuild_rag=false`，只走 cards/Wiki；其返回有 `pipeline`、`stages_planned`、`fetch_strategy`、`url`、`source_id`、`stages`、`rebuild_result`，失败时有 `stopped_at_stage`、`stopped_reason`。
- **[当前代码已实现：HTTP/MCP Job]** REST/MCP 会先创建 `asset_pipeline.ingest_source` job，`enqueue_asset_pipeline_ingest_source` 默认 `publish_release=true`；Worker 在 pipeline 成功后会触发 `_follow_up_publish`，执行 `build_mode:"full"` 的 RAG 发布。因此“adapter 默认不重建”和“HTTP 入队默认 follow-up full 发布”是不同层的行为，不能混说。
- **[当前有限实现]** `publish_release` 被 enqueue 层读取，但没有出现在 ingest MCP input schema 中；它并非可依赖的公开稳定字段。接口使用者应按当前默认行为和 job result 验证，不应假设 `rebuild_rag=false` 就能阻止 Worker follow-up 发布。

### B. `POST /api/asset-pipeline/delete-cards`

选择目标至少应提供一类：`card_id` / `card_ids` / `source_id` / `source_ids`。其余字段：`asset_pipeline_root`、`draft_path`、`validated_path`、`timeout_seconds`、`max_output_chars`、`rebuild_rag`（直接 adapter 默认 false）、`rebuild_mode`。

直接 pipeline 结果固定关注：`removed_draft_count`、`removed_validated_count`、`affected_source_ids`、`delete_result`、`rerender_result`、`rebuild_result`、`stopped_at_stage`。HTTP Job 成功后同样可能被 `publish_release` follow-up full build 覆盖为新 release。

### C. `POST /api/asset-pipeline/set-rag-visibility`

目标字段同 delete；还包括：

- `excluded`：bool，默认 true；true 只标记 `rag_excluded`，不删除卡、不影响 Wiki；
- `rebuild_rag`：直接 adapter 默认 true；
- `rebuild_mode`：`corpus` / `qdrant` / `full`，直接 adapter 默认 `corpus`；
- `draft_path`、`validated_path`、timeout/log 参数。

直接 pipeline 成功结果含：`excluded`、`affected_source_ids`、`updated_draft_card_ids`、`updated_validated_card_ids`、`not_found_card_ids`、`toggle_result`、`rebuild_result`。

### D. `POST /api/asset-pipeline/run-task`

必填 `task`，枚举为：

```text
add_source | build_sources | scrape_sources | fetch_video_transcripts |
retry_failed_fetches | validate_scrapes | extract_cards | validate_cards |
render_wiki | delete_cards | set_rag_visibility
```

其完整字段按任务组复用：

- source：`url,title,section_hint,enabled,include_in_wiki,title_override,mapped_metrics_override,notes,updated_by`；
- target cards：`card_id,card_ids,draft_path,validated_path,excluded`；
- 路径：`input_path,output_path,output_dir,output_root,usage_report_path,report_path,semantic_usage_report_path,semantic_cache_path,source_id,source_ids,source_id_file`；
- 抓取：`timeout_ms,timeout_sec,max_attempts,delay,retry_only,asr_fallback,asr_model,asr_language,asr_max_audio_mb`；
- 抽卡：`priority,limit,include_excluded,max_chars,max_cards_per_chunk,max_cards_per_source,model,force`；
- 校验/渲染：`semantic_mode,semantic_model,semantic_debug_notes,semantic_batch_size,include_needs_review,core_only,min_chars`；
- 子进程：`timeout_seconds,max_output_chars`。

`registry_path` 是 extract/validate 的历史兼容字段，schema 明确说明运行时来源真源已迁 MySQL、该参数不再实际生效。

单任务的直接 adapter 返回至少包含：`success`、`component`、`task`、`script`、`description`、`artifacts`；成功/超时执行结果还包含 `execution.command/cwd/exit_code/timed_out/duration_seconds/serialized/stdout/stderr`。注意：`success=true` 只代表该单脚本 return code 为 0；ingest pipeline 额外做业务产物核验。

### E. `POST /api/hybrid-retrieval/build-assets`

`build_mode` 为 `corpus` / `qdrant` / `full`（默认 `full`）。完整字段：

```text
project_root, asset_pipeline_root, output_root, build_mode, cards_path, wiki_root,
include_needs_manual_review, rag_docs_path, qdrant_path, qdrant_url,
collection_name, dense_model, sparse_model, batch_size,
publish_release, release_id, release_reason
```

Worker 完成后 build result 含 `build_mode`、`release_id`、`published_release`、`resolved_inputs`、`reports`。**[当前代码已实现]** 只有 build adapter 实际收到 payload 中的 `publish_release=true` 时，才将输出转到 `project_root/releases/<release_id>` 并切换 active release。**[当前有限实现/易误读点]** enqueue 层即使把 Job 列 `publish_release` 默认写为 true，原 payload 未必含此字段；而 `hybrid_retrieval.build_assets` 不走 follow-up publish。因此不要只看 Job 的 `publish_release`，必须看 Job `result.published_release` 是否非空。


## 9.5 Job 合同与状态机

### 当前 `workspace_jobs` 对象字段

`GET /api/jobs/<job_id>` 成功时返回 `job`。字段由 `_row_to_job` 固定转换：

```json
{
  "job_id": "UUID",
  "job_type": "hybrid_retrieval.build_assets | asset_pipeline.run_task | asset_pipeline.ingest_source | asset_pipeline.delete_cards | asset_pipeline.set_rag_visibility",
  "status": "queued | running | succeeded | failed",
  "payload": {},
  "result": {},
  "error": "",
  "requested_by": "",
  "worker_id": "",
  "attempts": 0,
  "publish_release": false,
  "created_at": "ISO-8601|null",
  "updated_at": "ISO-8601|null",
  "started_at": "ISO-8601|null",
  "finished_at": "ISO-8601|null",
  "lease_expires_at": "ISO-8601|null"
}
```

### 状态机

```text
enqueue_job
  → queued
claim_next_job（原子 UPDATE 成功）
  → running + worker_id + started_at + lease_expires_at + attempts+=1
heartbeat_job
  → running（仅延长 lease_expires_at）
complete_job
  → succeeded + result_json + finished_at + lease=null
fail_job
  → failed + error_text + result_json + finished_at + lease=null
lease 过期且下一次 claim 触发检查
  → queued + worker_id=null + started_at=null + lease=null
```

事实细节：

- Gateway Worker 默认 lease 为 **300 秒**，heartbeat 间隔 **60 秒**。
- `claim_next_job` 最多竞争 8 次，按 `created_at ASC` 取最早 queued job；UPDATE 使用 `WHERE job_id=? AND status='queued'`，只有 rowcount=1 才 claim 成功。
- expired running job 只在后续 `claim_next_job` 调用时被扫描并 requeue；它不会由独立清理器即时回收。
- `attempts` 每次成功 claim 加一；过期回队不清零。
- heartbeat 仅在 `job_id + running + worker_id` 同时匹配时续租，避免旧 Worker 把已被重新认领的 job 续活。

### 幂等与重试边界

| 项目 | 当前真实行为 |
| --- | --- |
| HTTP 幂等键 | **未实现**；没有 `Idempotency-Key` 字段、唯一约束或请求去重 |
| 重复 POST | 会创建不同 UUID job，可能重复执行 |
| Worker 崩溃 | lease 到期后下一次 claim 重新入队 |
| 业务副作用幂等 | 部分脚本有稳定 source id、upsert 或重建语义，但不能把整个 ingest 当作严格 exactly-once |
| API 自动 retry | 未实现统一指数退避/最大 attempts 策略；`attempts` 是观测字段，不是当前自动失败重试策略 |
| 抓取重试 | 由 02c 或各抓取脚本参数控制，和 Job requeue 是不同层的重试 |

### `job_type` 到结果字段的稳定合同

所有 Job 失败时，外层稳定字段都是 `status:"failed"`、非空 `error`、`finished_at`，并保留对象形态的 `result`；但当前没有跨任务统一的字段级错误码。成功时优先按下表读取 `result`，不要从 `publish_release` 列推断是否真的完成发布。

| `job_type` | payload 关注字段 | `succeeded` 时 `result` 的稳定字段 | `failed` 时的诊断位置 | 发布结果判定 |
| --- | --- | --- | --- | --- |
| `hybrid_retrieval.build_assets` | `build_mode`、输入/输出路径、索引参数、`publish_release`、`release_id` | `build_mode`、`release_id`、`published_release`、`resolved_inputs`、`reports` | `error` 为主；`result` 保留已生成的阶段信息（如有） | 仅当 `result.published_release` 非空才表示已发布/切换 release。 |
| `asset_pipeline.run_task` | `task` 及任务分组字段、超时/输出限制 | `success`、`component`、`task`、`script`、`description`、`artifacts`；执行详情位于 `execution` | `error`；如子进程已启动，结合 `result.execution.exit_code/timed_out/stdout/stderr` 排查 | 不触发统一 follow-up publish；任务本身是否重建由其 `task` 与 payload 决定。 |
| `asset_pipeline.ingest_source` | `url`、编目字段、ASR 字段、`rebuild_*` | `pipeline`、`stages_planned`、`fetch_strategy`、`url`、`source_id`、`stages`、`rebuild_result` | `error`，以及 pipeline 的 `stopped_at_stage`、`stopped_reason` | Worker 成功后默认 follow-up `full` 发布；当前没有承诺稳定的嵌套 publish 字段，应以 active release 或后续 build 结果确认。 |
| `asset_pipeline.delete_cards` | card/source 目标、路径、`rebuild_*` | `removed_draft_count`、`removed_validated_count`、`affected_source_ids`、`delete_result`、`rerender_result`、`rebuild_result` | `error`、`stopped_at_stage` | Worker 成功后默认 follow-up `full` 发布；不要仅以 Job 列 `publish_release` 作为发布成功证据。 |
| `asset_pipeline.set_rag_visibility` | card/source 目标、`excluded`、路径、`rebuild_*` | `excluded`、`affected_source_ids`、`updated_draft_card_ids`、`updated_validated_card_ids`、`not_found_card_ids`、`toggle_result`、`rebuild_result` | `error`、目标未找到信息或阶段失败信息 | Worker 成功后默认 follow-up `full` 发布；最终可见性以新 release 的 `rag_docs.jsonl` 为准。 |

**[当前有限实现]** follow-up publish 的过程、失败详情和嵌套结果还没有统一的字段合同。因此调用方必须区分“业务 pipeline 成功”“发布动作已尝试”“新 release 已实际激活”三件事；只有最后一件才能说明线上读取面已更新。

## 9.6 MCP 合同

### Transport

- stdio：`Content-Length` + JSON-RPC 2.0，默认协议版本 `2024-11-05`。
- HTTP：`POST /mcp` 首先发 `initialize`，响应 header 返回 `Mcp-Session-Id` 和 `MCP-Protocol-Version`；之后每个请求必须带二者；`DELETE /mcp` 关闭会话。
- 当前无 SSE；`GET /mcp` 返回 `405`。

### 九个 Tool 与输入

| Tool | 输入 |
| --- | --- |
| `knowledge_workspace_health` | `{}` |
| `retrieve_knowledge` | 9.3 检索合同 |
| `build_agent_context` | 9.3 检索合同 |
| `hybrid_retrieval_build_assets` | 9.4-E build 字段 |
| `asset_pipeline_get_status` | `asset_pipeline_root,wiki_root,registry_path,draft_cards_path,validated_cards_path,validation_report_path,catalog_path` 均可选 |
| `asset_pipeline_run_task` | 9.4-D task 字段 |
| `asset_pipeline_ingest_source` | 9.4-A ingest 字段 |
| `asset_pipeline_delete_cards` | 9.4-B delete 字段 |
| `asset_pipeline_set_rag_visibility` | 9.4-C visibility 字段 |

MCP `tools/call` 的 JSON-RPC 结果统一包装为：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [{"type": "text", "text": "<完整业务 JSON 的格式化文本>"}],
    "structuredContent": {"success": true},
    "isError": false
  }
}
```

业务失败不是一定走 JSON-RPC error：工具 handler 返回 `success:false` 时，会仍返回 JSON-RPC `result`，但 `isError:true`。协议/消息错误才走 JSON-RPC `error`，常见 code：`-32700` JSON parse、`-32600` invalid request、`-32601` method not found、`-32603` internal error。

## 9.7 当前实现与生产演进的边界

| 标记 | 内容 |
| --- | --- |
| **[当前代码已实现]** | REST 写接口入队；MySQL job/lease/heartbeat；MySQL single writer lock；active release；HTTP/stdio MCP；单进程内 MCP session。 |
| **[当前有限实现]** | 无 auth/RBAC；无幂等 key；错误主要为字符串；Job 无统一 retry policy；MCP session 无 TTL 清理且只存进程内；schema 是 MCP tool 描述但 REST handler 不做 JSON Schema 验证。 |
| **[推荐生产化演进]** | Idempotency-Key + request hash；状态枚举/错误码；指数退避和 retry policy；trace/job event；JWT/RBAC；共享 session；远端 Qdrant 与多 Worker 分队列。 |
