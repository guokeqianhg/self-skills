# 6. 函数级代码地图：追问时如何快速定位

> 本篇不是逐行复述全部代码，而是覆盖主链路的关键函数。每项按“调用方、输入/输出、关键控制、持久化影响、可讲亮点”复习。路径均相对于工作区根目录。

## 6.1 上游来源与知识卡

### 来源构建：`knowledge_asset_pipeline/scripts/01_build_sources.py`

| 函数 | 位置 | 调用/输入→输出 | 要讲的实现细节 |
| --- | --- | --- | --- |
| `normalize_source_url(url)` | 82 | 被 `make_source_id`、候选聚合使用；URL → 标准 URL | scheme/netloc 小写、query 排序、移除 fragment/末尾 `/`；让同资料稳定去重 |
| `make_source_id(url)` | 92 | URL → `SRC-` + SHA-1 前 10 位 | id 建立在规范 URL 上；不要说用随机 UUID |
| `infer_platforms/engines/categories/topics` | 100/116/146/164 | URL/上下文文本 → 自动 metadata | 都是关键词启发式；无命中有 Cross-platform、Unknown/General 等默认值 |
| `infer_fetch_strategy(parsed_url)` | 214 | URL → `video_transcript_first` 或 `web_crawler` | 视频平台 `/video/` 优先视频文本，其余网页抓取 |
| `infer_auto_metrics(...)` | 247 | 语义文本/主题 → 指标 | 调 metric aliases；核心 scope 再按主题补指标 |
| `parse_cases/parse_mentions` | 264/287 | Markdown 行 → case/URL mention | 同一 URL 可有多个 mention，后续按规范 URL 聚合 |
| `extract_candidates(input_path)` | 332 | 清单 → 候选 source 列表 | 组装自动字段、版权/抓取策略；无文件输出 |
| `main()` | 430 | candidates → MySQL upsert | 最终调 `registry_db.upsert_candidates`，不是写旧 JSONL |

### 注册表：`scripts/registry_db.py`

- 关键职责：MySQL row 与历史嵌套 JSONL 的映射、schema 兼容、来源查询/upsert、人工字段更新和乐观锁。
- 面试重点：`upsert_candidates` 只刷新自动推断列，人工编目和 fetch/review 状态保留；这保证反复同步清单不会覆盖人工操作。
- 追问“并发编辑”：转到人工更新函数的 expected-version 校验；冲突应返回让用户重新读取，而不是 last-write-wins。

### 抓取：`02_scrape_sources.py` / `02b_fetch_video_transcripts.py` / `02c_retry_failed_fetches.py`

| 代码路径 | 控制流 | 副作用 |
| --- | --- | --- |
| 02 网页 | 读取 enabled 来源 → 网页抓取服务 → 清洗 | 写 `data/raw`、`data/meta`、`data/markdown`，回写 fetch 状态 |
| 02b 视频 | 字幕/页面/API → 可用则存 transcript；否则音频下载 → ffmpeg WAV/分片 → ASR | 同样三类资产与来源状态 |
| 02c 重试 | 读取失败来源的 `fetch.strategy` → 分流 02 或 02b | 不改变“原本应该走哪种抓取器”的决策 |

### 校验和抽取：03/04/05

| 函数/入口 | 位置 | 输入 → 输出 | 关键点 |
| --- | --- | --- | --- |
| `03_validate_scrapes.py` 主流程 | 脚本主入口 | Markdown/meta → source 状态 + `scrape_validation.jsonl` | 规则质检先于模型，避免空/差正文消耗 token |
| `04_extract_cards.py` 主流程 | 脚本主入口 | enabled P0/P1 来源 + 已校验 Markdown → `cards_draft.jsonl` | 通用模型兼容接口 Chat Completions；Pydantic 后处理；记录 `extract_usage.jsonl` |
| `05_validate_cards.py` 主流程 | 脚本主入口 | draft cards → validated cards + validation manifests | 结构规则→启发式短路→剩余卡 batch LLM；默认 12；语义 Prompt 版本 `2026-06-23-v2` 纳入缓存 key |

| `06_render_wiki.py` 主流程 | 脚本主入口 | validated cards → `wiki/` / index | 人读 Markdown 与机读 JSONL 分层 |

### 模型与数据模型：`models.py`、`metric_aliases.py`

- `models.py`：用 Pydantic 定义和校验知识卡结构，是 LLM 输出进入正式资产前的第一道结构闸门。
- `metric_aliases.py`：统一指标别名，并由来源初判及后续处理复用；避免 `FPS`、平均帧率、`avgFps` 形成孤岛。

## 6.2 RAG 构建、检索和报告

### 语料：`src/hybrid_retrieval_service/builder.py`

| 函数 | 位置 | 输入 → 输出 | 关键分支/副作用 |
| --- | --- | --- | --- |
| `read_jsonl(path)` / `write_jsonl(path, records)` | 63/68 | JSONL ↔ records | 公共文件读写基础 |
| `build_rag_doc(card, wiki_record, wiki_indexes)` | 494 | 一张 card + Wiki → RAG document | 拼入 `dense_text`、`sparse_text`、metadata、证据字段 |
| `build_rag_corpus(asset_pipeline_root, output_root, cards_path=None, wiki_root=None, excluded_review_statuses=None)` | 574 | starter assets → build report dict | 优先 validated、缺失回退 draft；按 card id 对齐；跳 `rag_excluded`；写 `rag_docs.jsonl` 与 `build_report.json` |

### 规则查询理解：`query_understanding.py`

| 函数/对象 | 位置 | 作用 |
| --- | --- | --- |
| `EvidenceFrame` | 151 | 统一描述平台、引擎、指标族、正反信号、场景、趋势、干预和假说域 |
| `build_evidence_frame(...)` | 236 | 通用规则式文本 → EvidenceFrame |
| `build_query_evidence_frame(...)` | 265 | 请求字段 → 查询 frame |
| `build_doc_evidence_frame(doc)` | 281 | RAG doc → 文档 frame |
| `build_query_text_for_card_type(...)` | 358 | 为不同 card type 拼接检索 query text |

### Local：`retriever.py`

| 函数/对象 | 位置 | 输入 → 输出 | 解释重点 |
| --- | --- | --- | --- |
| `QuerySpec` | 38 | API/CLI 参数对象 | `__post_init__` 归一化、去重并建立 `query_frame` |
| `build_bm25_index(docs)` | 122 | docs → term/document stats | 本地 BM25 基础统计来自 `sparse_text` |
| `evidence_alignment(spec, doc)` | 235 | query/doc frame → 命中、冲突、业务加减分 | required/supporting/anti signal 是核心领域差异 |
| `score_doc(...)` | 380 | 单文档 → score dict 或 `None` | 严格平台/引擎不通过直接排除；其余结合 BM25、Counter cosine、alignment 和质量规则 |
| `retrieve(rag_docs_path, spec, top_k=3)` | 438 | 文件+spec → 按 card type 分组的 top K | 先全量读 docs/索引，再分类排序；是无向量 DB 基线 |

**Local 最终分数要能讲清：**

\[
score=((0.58\times BM25)+(0.42\times CounterCosine))\times platformMultiplier\times engineMultiplier+evidenceBoost+compositeAdjustment+priorityBoost+reviewBoost-warningPenalty
\]

- BM25 固定参数 `k1=1.5`、`b=0.75`；不是调用第三方搜索引擎。
- 平台精确匹配倍率 `1.08`、通用文档 `0.92`、一般 soft mismatch `0.68`；引擎精确 `1.05`、通用 `0.94`、一般 soft mismatch `0.72`。严格条件直接返回 `None`。
- `evidence_alignment` 对指标族、required/supporting、正负信号、场景、趋势、干预分别加分；并对 GPU/CPU/thermal/memory 假设与 `anti_signals` 的冲突扣分。
- `P0` 加 `0.14`，核心索引再加 `0.04`；`human_reviewed` 加 `0.12`、`auto_checked` 加 `0.04`。warning 逐项扣分且总上限 `0.20`。
- 返回值保留上述每个分量和命中/冲突标签，因此 local 排名可解释，而不是黑盒相似度。

### Vector：`vector_store.py` / `qdrant_retriever.py` / `reranker.py`


| 函数/对象 | 位置 | 输入 → 输出 | 解释重点 |
| --- | --- | --- | --- |
| `VectorIndexConfig` | `vector_store.py:26` | 建库参数 dataclass | rag docs、目录、collection、模型、batch、可选 URL |
| `build_qdrant_index(config)` | 212 | config → 索引构建报告/manifest | dense+sparse vectors、collection/payload、临时路径或远端发布策略 |
| `retrieve_qdrant(...)` | `qdrant_retriever.py:72` | Qdrant + QuerySpec → 按类型结果 | candidate limit 默认 128；可 dense/hybrid、DBSF fusion、filter、可选 rerank |
| `normalize_rerank_mode(mode)` | `reranker.py:19` | mode → `none` / `local_hybrid` | 不允许未定义模式悄悄生效 |
| `build_local_hybrid_rerank_map(spec, docs, query_text)` | 35 | Qdrant candidates → 规则精排分数 | 只对候选重建 local BM25 并复用 `score_doc`，不是全库二次检索 |

### 输出：`final_reporter.py` / `service_facade.py`

| 组件 | 职责 | 面试说法 |
| --- | --- | --- |
| `final_reporter` | 根据 build report 回表、按卡片角色组织命中、补 source claims/verification/optimization/risk/applicability、构造 agent context | 检索结果被提升为证据包；不替代最终诊断推理 |
| `service_facade` | payload 归一、QuerySpec、health、检索/agent context 服务合同、MCP schema | 将算法包从 HTTP/MCP transport 中解耦；本地 Qdrant 有进程内互斥 |

## 6.3 Gateway：请求、队列、Worker、协议

> **状态标记：**当前 REST 写路由会入队并由 Worker 执行；直接调用 adapter 才是同步 pipeline。当前 Job 已有 lease/lock，但缺幂等键、回调和统一 retry policy；这些演进项详见 `09-api-mcp-job-contract.md`。


### HTTP 与 MCP

| 函数/对象 | 位置 | 追问答案 |
| --- | --- | --- |
| `_McpSessionStore` | `api.py:46-74` | 内存 dict + `threading.Lock` 管 HTTP MCP session；仅单进程有效 |
| `WorkspaceApiHandler` | `api.py:77-439` | `ThreadingHTTPServer` handler，负责 JSON/CORS/路由/错误映射 |
| `create_http_server` / `serve_http_api` | `api.py:442-454` | HTTP 服务构造与启动 |
| `call_mcp_tool(tool_name, arguments=None)` | `runtime.py:430-441` | 从 9 个 tool handler 选择业务函数；未知工具返回结构化失败而非直接抛出 |
| `handle_mcp_message` / `run_stdio_mcp_server` | `mcp.py:159-205` | stdio 的 initialize/tools/list/tools/call；`tools/call` 复用 runtime |

### API 到 Worker 的真实链路

```text
POST build-assets/run-task/ingest/delete/set-visibility
→ runtime.build_*_payload
→ workspace_jobs.enqueue_*
→ MySQL job
→ serve_worker.py 轮询 claim
→ workspace_jobs 执行对应 adapter
→ 成功/失败状态与结果落库
```

读接口（health/status/retrieve/agent context）在 HTTP 请求线程内同步运行；上述五类写接口入队，不在 API handler 内执行。

### `workspace_jobs.py`

| 函数 | 位置 | 作用 |
| --- | --- | --- |
| `enqueue_hybrid_retrieval_build_assets(payload=None)` | 94 | 入队 `hybrid_retrieval.build_assets` |
| `enqueue_asset_pipeline_run_task(payload=None)` | 107 | 入队单一上游任务 |
| 其余 `enqueue_*` | 同模块 | 分别入队 ingest/delete/visibility 等写操作 |

重点复述：`_JOB_LEASE_SECONDS=300`，`_JOB_HEARTBEAT_INTERVAL_SECONDS=60`。`run_worker_iteration` claim job 后调用 `run_single_job`；后者先获取全局写锁，后台 daemon thread 每 60 秒 heartbeat，执行 adapter，刷新 live cards index，需要发布时再走 full RAG follow-up；异常调用 `fail_job`，正常完成调用 `complete_job`，finally 释放写锁。不要说所有读取也进 job。


### Adapter

| 函数 | 位置 | 作用 |
| --- | --- | --- |
| `run_asset_pipeline_task(payload=None)` | `adapters/asset_pipeline.py:739` | 白名单 task → command builder → subprocess → 产物摘要 |
| `run_ingest_source_pipeline(payload=None)` | 973 | add→抓取分流→正文校验→抽卡→卡校验→Wiki；逐阶段核验 |
| `run_delete_cards_pipeline(payload=None)` | 1158 | 删卡后重渲染受影响 Wiki |
| `run_set_rag_visibility_pipeline(payload=None)` | 1261 | 修改可见性，默认重建语料 |

## 6.4 如何用本篇回答“某函数在哪”

- **URL/来源自动推断**：6.1 `01_build_sources.py`。
- **批量 LLM 校验缓存**：6.1 03/04/05 表，详见 `07`。
- **BM25/规则分数**：6.2 `build_bm25_index`、`evidence_alignment`、`score_doc`。
- **Qdrant hybrid/二阶段**：6.2 `retrieve_qdrant`、`build_local_hybrid_rerank_map`。
- **路径 manifest/索引构建**：6.2 `build_rag_corpus`、`build_qdrant_index`。
- **任务队列和 lease**：6.3 `workspace_jobs.py`。
- **HTTP MCP session**：6.3 `_McpSessionStore`。
- **子进程安全与业务成功核验**：6.3 `run_asset_pipeline_task` 与 `run_ingest_source_pipeline`。
