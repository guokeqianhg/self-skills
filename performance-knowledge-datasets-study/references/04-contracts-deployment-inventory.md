# 4. 数据合同、部署运行时与全源码索引

## 4.1 核心数据合同

### MySQL `sources`

用途：来源注册和流程状态真源。字段按责任分组理解：

| 分组 | 典型内容 | 写入者 |
| --- | --- | --- |
| 身份/来源 | source id、URL、标题、平台类型 | 01/单条新增 |
| 自动推断 | platform、engine、topic、mapped metrics | 01 规则 |
| 人工编目 | enabled、priority、include in wiki、override、notes | webapp / 人工 |
| 抓取 | fetch status、strategy、时间、错误、路径 | 02/02b/02c |
| 审核 | 正文/卡片 review 状态、版本 | 03/05/人工 |

数据库 schema 位于 `knowledge_asset_pipeline/db/schema.sql`；不要以 JSONL 字段猜测 MySQL 真相。来源更新要遵循自动字段与人工字段分离、乐观锁的约束。

### Card JSONL

卡片记录的关键识别字段为 `card_id`、`source_id`、`card_type`。围绕检索和审计还有：

- `card_priority`、`review_status`、`rag_excluded`；
- platform/engine/metrics；
- signal、source claims、applicability、risk/anti-signal；
- verification / optimization 相关结构。

`cards_draft.jsonl` 是模型抽取中间物；`cards_validated.jsonl` 是下游正式输入。`rag_excluded=true` 表示“仍保留资产，但不进入本次 RAG 语料”，而不是删除。

### `rag_docs.jsonl`

由 builder 生成，是检索面的归一化快照。每条 document 结合 card 元数据、Wiki 文本、`dense_text`、`sparse_text` 与下游证据字段。其价值是将下游从上游文件布局中解耦；以 `build_report.json` 记住本次构建的输入/输出路径。

### Qdrant manifest

`qdrant_manifest.json` 固化 collection、dense/sparse model、索引路径等，让查询端加载与建库端一致的模型配置。没有 manifest 一致性会导致“索引用 A encoder，query 用 B encoder”的隐蔽质量事故。

## 4.2 检索请求和响应心智模型

典型检索输入：

```json
{
  "backend": "qdrant",
  "platform": "Android",
  "engine": "Unity",
  "metrics": ["avgFps", "low1pct", "p99FrameTimeMs"],
  "symptoms": ["平均帧率看起来还可以", "转场时有明显长尾卡顿"],
  "anti_signals": ["GPU利用率并不持续拉满"],
  "card_priorities": ["P0"],
  "top_k": 3,
  "rerank_mode": "local_hybrid"
}
```

处理顺序：payload normalization → `QuerySpec` → query understanding → backend retrieve → optional rerank → final report package。响应不只是 doc IDs，还会按诊断角色组织证据，并包含可给 Agent 使用的 context。

## 4.3 环境变量与依赖

### 上游能力

| 变量 | 作用 |
| --- | --- |
| `DATABASE_HOST/PORT/USER/PASSWORD/DATABASE` | 来源和 job 数据库 |
| `CRAWLER_API_KEY` | 网页抓取 |
| `MODEL_API_KEY` | 抽卡、语义校验、可选 ASR |
| `MODEL_BASE_URL` | 通用模型兼容接口 网关 |
| `MODEL_DEFAULT_NAME` | 抽卡默认模型 |
| `MODEL_VALIDATE_NAME` | 语义验证模型覆盖 |
| `MODEL_ASR_NAME/LANGUAGE` | 视频语音转写 |
| `ASR_FALLBACK`、`VIDEO_PLATFORM_COOKIE` | 视频抓取/转写策略 |

### RAG 与部署

| 变量 | 作用 |
| --- | --- |
| `QDRANT_URL` | 外部 Qdrant 服务 URL；未配置可用本地路径模式 |
| `FASTEMBED_CACHE_PATH` | embedding 模型缓存 |
| `HF_HUB_OFFLINE`、`TRANSFORMERS_OFFLINE` | 运行期离线读取缓存，防止集群网络超时 |
| `WEBAPP_HOST/PORT/DEBUG` | 管理端运行参数 |

主要依赖类别：PyMySQL/数据库、Flask 管理端、网页抓取服务、通用模型兼容接口 client、Pydantic、Playwright/ffmpeg/ASR、Qdrant client、FastEmbed/ONNX、Markdown/HTTP 工具。准确版本以三个 `requirements.txt` 为准。

## 4.4 Dockerfile 的关键思路

- 基础镜像 `python:3.11-slim`；安装 `libgomp1` 以满足推理相关运行依赖。
- 先复制三个 requirements，先装依赖，再复制完整代码，改善 Docker layer cache。
- 在构建期打包 `asset_pipeline/data`、`asset_pipeline/wiki`、`hybrid_retrieval/data` 到 `/opt/knowledge-seed`。
- 构建期临时允许 HuggingFace 联网并预热 `thenlper/gte-large` 与 `Qdrant/bm25`；运行期设置离线开关，避免受限集群首次请求卡在下载。
- 默认启动 `serve_api.py:8787`。

## 4.5 Compose 拓扑

`docker-compose.yml` 用于本地独立体验：

```text
mysql(8.0) ← app(gateway)
          ← worker
          ← webapp
qdrant(1.18) ← app / worker
```

- MySQL 挂 `schema.sql` 初始化；
- app、worker、webapp 共用同一镜像，区别仅是 command；
- cards、wiki、RAG data、Qdrant、MySQL 分别映射命名 volume；
- `depends_on` 等 MySQL healthcheck 与 Qdrant 启动；
- `down` 不带 `-v` 不删数据。

`docker-compose.shared-mysql.yml` 用于共享实例：不启动本地 MySQL，连接外部共享 MySQL；`asset_seed` 在 volume 为空时将镜像内 seed 解包到持久卷；Qdrant 仍是单实例本地存储，不能横向扩副本。

## 4.6 自研 Python 代码全量职责索引

### Root

| 文件 | 说明 |
| --- | --- |
| `tmp_count_sources.py` | 临时/辅助统计脚本；不应作为主链路设计依据 |

### `knowledge_asset_pipeline/scripts`

| 文件 | 说明 |
| --- | --- |
| `00_migrate_registry_to_mysql.py` | Excel/JSONL 历史注册表迁 MySQL |
| `01_build_sources.py` | Markdown 清单解析、推断、URL 规范化、MySQL upsert |
| `01b_add_single_source.py` | 单 URL 来源登记，动态复用 01 |
| `02_scrape_sources.py` | 网页抓取服务 网页抓取、资产落盘、状态回写 |
| `02b_fetch_video_transcripts.py` | 字幕、视频 API、音频/ASR 兜底 |
| `02c_retry_failed_fetches.py` | 按 fetch strategy 分流失败重试 |
| `03_validate_scrapes.py` | 清洗正文确定性质量校验 |
| `04_extract_cards.py` | LLM+Pydantic 知识卡抽取 |
| `05_validate_cards.py` | 本地规则、批量 LLM 语义校验、缓存/分层 |
| `06_render_wiki.py` | 卡片到 Wiki 页面、索引渲染 |
| `cards_admin.py` | 卡片删除与 `rag_excluded` 可见性管理 |
| `metric_aliases.py` | 指标别名统一 |
| `models.py` | Pydantic 数据模型/校验 |
| `registry_db.py` | MySQL DAO、兼容映射、乐观锁 |
| `scrape_cleaning.py` | 抓取文本清洗 |
| `sync_sources_table.py` | 两 MySQL sources 表同步 |
| `workspace_store.py` | job/lease/锁/发布/索引等运行时存储基础能力 |

### `knowledge_asset_pipeline/webapp`

| 文件 | 说明 |
| --- | --- |
| `app.py` | 人工来源编目、状态浏览、卡片/来源管理 UI 后端 |

### `hybrid_retrieval_service/src/hybrid_retrieval_service`

| 文件 | 说明 |
| --- | --- |
| `__init__.py` | 公开 API 聚合 |
| `builder.py` | cards/Wiki 对齐和语料构建 |
| `text_utils.py` | 清洗、别名、切词、Counter cosine |
| `query_understanding.py` | 自由文本到 EvidenceFrame/QuerySpec 的规则理解 |
| `retriever.py` | local BM25+cosine+规则检索 |
| `reranker.py` | Qdrant 候选的 local hybrid 二阶段排序 |
| `vector_store.py` | dense/sparse embedding、建库、manifest、切换 |
| `qdrant_retriever.py` | dense/hybrid search、filter、fusion、rerank 接入 |
| `final_reporter.py` | 命中回表、agent context、Markdown/JSON 报告 |
| `service_facade.py` | 请求归一、health、服务 payload、MCP schema、Qdrant 互斥 |

### `hybrid_retrieval_service/scripts`

| 文件 | 说明 |
| --- | --- |
| `build_rag_docs.py` | 语料构建 CLI |
| `build_qdrant_index.py` | 索引构建 CLI |
| `build_full_rag.py` | 全量构建 CLI |
| `retrieve_cards.py` | 检索 CLI、路径解析与 fail-fast |
| `generate_final_report.py` | agent context/报告 CLI |
| `generate_nl_test_report.py` | 固定自然语言回归和 Markdown 报告 |
| `smoke_test_retrieval.py` | local/raw/rerank smoke/一致性检验 |
| `serve_api.py` | 兼容入口，转 gateway HTTP |
| `serve_mcp.py` | 兼容入口，转 gateway stdio MCP |

### `workspace_orchestration_gateway`

| 文件 | 说明 |
| --- | --- |
| `bootstrap.py` | monorepo path/bootstrap |
| `runtime.py` | REST/MCP 共用业务门面、9 tool 定义 |
| `api.py` | `ThreadingHTTPServer` REST/HTTP MCP 实现 |
| `mcp.py` | stdio Content-Length JSON-RPC MCP server |
| `workspace_jobs.py` | MySQL job、lease、全局写锁、Worker 执行 |
| `adapters/hybrid_retrieval.py` | direct Python adapter、资产构建/发布 |
| `adapters/asset_pipeline.py` | 任务白名单、subprocess、阶段核验、编排 |
| `scripts/serve_api.py` | HTTP 启动+模型预热 |
| `scripts/serve_worker.py` | Worker 启动 |
| `scripts/serve_mcp.py` | stdio MCP 启动 |
| `scripts/seed_runtime_assets.py` | 首次部署安全解包 seed，防 path traversal |

## 4.7 提交与运行时文件边界

应维护：源码、README、Docker/compose、schema、`.env.example`、必要可交付资产快照。

不应提交：真实 `.env`/cookie/key、`.venv`、`__pycache__`、运行锁、临时备份、私有日志。Qdrant/JSONL/Wiki 是否提交取决于交付策略，但必须理解它们是可以从上游重建的派生资产。
