# 0. 工作区架构与端到端数据流

## 0.1 定位：三个工程而不是一个普通 RAG Demo

`performance_knowledge_workspace` 把 性能采集平台 性能知识加工分成三层：

```text
人工资料清单
  → 上游知识资产构建（knowledge_asset_pipeline）
  → 下游检索/报告（hybrid_retrieval_service）
  → 统一服务与任务编排（workspace_orchestration_gateway）
```

| 层 | 责任 | 关键输出 | 不负责什么 |
| --- | --- | --- | --- |
| `asset_pipeline` | 来源注册、抓取、清洗、卡片抽取/验证、Wiki | `cards_validated.jsonl`、`wiki/`、MySQL `sources` | 不做 embedding、Qdrant、在线检索 |
| `hybrid_retrieval` | 卡片/Wiki 对齐、语料、local/Qdrant 检索、报告 | `rag_docs.jsonl`、Qdrant collection、agent context | 不写入来源和卡片治理状态 |
| `orchestration_gateway` | REST/MCP、adapter、异步 job、部署入口 | HTTP JSON / MCP JSON-RPC / job/release | 不重写上游 ETL 算法 |

## 0.2 真源与可重建产物

- 批量来源的人工入口：根目录 `专家知识库资料链接清单.md`。
- 运行时来源注册真源：MySQL `sources` 表。它包含人工编目、抓取状态和审核相关状态。
- 上游正式知识资产：`data/cards/cards_validated.jsonl` 与 `wiki/`。
- 下游语料快照：`hybrid_retrieval_service/data/rag_docs.jsonl`。
- 向量索引：Qdrant（本地目录或外部 Qdrant URL）。
- 历史 `source_catalog.xlsx`、`sources.jsonl` 是迁移/排障快照，而非日常主注册表。

这一区分回答了一个高频问题：**修改 Markdown 来源清单并不等于线上来源状态已经变了**；需要 `01_build_sources.py` 才会把候选来源 upsert 到 MySQL，且下游检索还需后续构建。

## 0.3 全链路

```text
专家资料链接清单.md
  │  01_build_sources.py：解析、URL 归一化、启发式标签、upsert
  ▼
MySQL sources
  │  webapp：人工启用/优先级/覆盖字段；乐观锁更新
  ▼
02 scrape / 02b video transcript / 02c retry
  ▼
raw + meta + markdown
  │  03：确定性正文质量校验
  ▼
04：通用模型兼容接口 LLM + Pydantic 抽取知识卡
  ▼
cards_draft.jsonl
  │  05：结构规则 + 批量语义验证 → P0/P1 与 review 状态
  ▼
cards_validated.jsonl ── 06 → wiki/
  │
  │  builder.build_rag_corpus()：card_id 对齐、过滤 rag_excluded
  ▼
rag_docs.jsonl + build_report.json
  │  vector_store.build_qdrant_index()：dense/sparse embedding + manifest
  ▼
Qdrant / manifest
  │
  ├─ local：BM25 + token-counter cosine + 业务规则
  └─ qdrant：dense/sparse 候选召回 + local_hybrid 二阶段精排
  ▼
final_reporter：证据回表、agent_context、Markdown/JSON 包
  ▼
gateway：REST / stdio MCP / HTTP MCP
```

## 0.4 “新增一条 URL” 的真实控制流

Gateway 的 `ingest_source` 不是魔法接口，而是组合了受控阶段：

```text
add_source
→ 按来源类型路由 scrape_sources 或 fetch_video_transcripts
→ validate_scrapes
→ extract_cards
→ validate_cards
→ render_wiki
→ 可选 build RAG
```

它对每阶段除检查子进程 exit code 外，还会核验实际产物/状态。这里必须分层：**直接调用** `run_ingest_source_pipeline` 时默认 `rebuild_rag=false`，所以只更新来源、卡片和 Wiki；但 **HTTP/MCP 写接口** 会先入队，当前 Worker 对成功的 `asset_pipeline.ingest_source` Job 默认执行 follow-up `full` 发布构建（`publish_release=true`）。两者不是同一个调用面，不能只背“默认不重建”。字段级合同与行为差异见 `09-api-mcp-job-contract.md`。


## 0.5 代码阅读顺序

1. 根 `README.md`：数据真源、部署与工作流。
2. `asset_pipeline/README.md` + `db/schema.sql`：先理解上游数据和 01–06。
3. `hybrid_retrieval/README.md` + `builder.py` + `retriever.py`：理解消费侧。
4. `vector_store.py` / `qdrant_retriever.py` / `final_reporter.py`：理解检索质量与输出合同。
5. `gateway/runtime.py` / `api.py` / `workspace_jobs.py`：理解服务化。

## 0.6 核心架构取舍

1. **文件资产 + MySQL 注册表**：来源工作流需要可查询/可并发编辑状态，MySQL 适合；内容产物需要可审阅、可 Git diff、可离线交付，JSONL/Markdown 适合。
2. **local + vector 双轨**：local 结果稳定且可解释，vector 提升语义召回；不是为了“多用一个数据库”，而是兼顾质量、可解释性、依赖失败回退。
3. **两类 adapter**：`hybrid_retrieval` 有稳定 Python API，直接 import；`asset_pipeline` 是编号脚本集合，采用任务白名单+参数翻译+子进程隔离。
4. **任务队列承接写操作**：构建/抓取/LLM 任务慢且会修改共享资产；HTTP 读取可同步，写入应由 Worker claim 后串行执行。

## 0.7 当前实现边界（必须诚实说明）

- 没有认证与授权，暴露网络前必须加网关或认证中间件。
- Qdrant 本地文件模式不适合多进程/多副本共同写同一路径；共享部署应单实例或使用远端 Qdrant。
- HTTP MCP session 在进程内存，多副本需要粘滞会话或外部 session store。
- 当前 REST 写路由已入队到 Job/Worker；但尚无回调/事件流、统一自动重试或幂等键，长任务调用方必须轮询 job 状态。直接调用 adapter 仍可能是长同步执行。

- 知识质量依赖来源、LLM 和规则；RAG 是证据检索底座，不应被表述为自动根因裁决器。
