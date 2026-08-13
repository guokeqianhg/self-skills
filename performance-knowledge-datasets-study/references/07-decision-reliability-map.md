# 7. 判断边界与可靠性地图：规则、模型、缓存、并发、重试、降级

> 这篇用于避免面试中把“脚本规则”“外部服务”“LLM 判断”“向量模型”混为一谈。

## 7.1 决策归属总表

| 场景 | 执行者 | 关键实现位置 | 输入 → 输出 | 失败/降级事实 |
| --- | --- | --- | --- | --- |
| URL 规范化与 source id | Python 规则 | `01_build_sources.py::normalize_source_url`、`make_source_id` | URL → 稳定规范 URL / `SRC-` SHA-1 前缀 | 无模型兜底；同 URL 由规范键聚合 |
| 资料分类/指标初判 | Python 启发式 | `infer_platforms`、`infer_engines`、`infer_categories`、`infer_auto_metrics` | 标题/章节/URL → 自动 metadata | 人工 override 可修正，不应称“AI 分类” |
| 抓网页 | 网页抓取服务 | `02_scrape_sources.py` | URL → raw/meta/Markdown | 失败状态回写 MySQL；可由 02c 重试 |
| 视频文本 | 字幕/API/ASR | `02b_fetch_video_transcripts.py` | 视频 URL → transcript Markdown | 字幕优先；无字幕时音频、ffmpeg、ASR；这才是实际降级链 |
| 正文是否可抽卡 | Python 规则 | `03_validate_scrapes.py` | Markdown/meta → 校验状态/issues | 不调用 LLM |
| 初次知识卡生成 | LLM | `04_extract_cards.py` | 已校验正文 → draft cards | Pydantic 与本地后处理限制结构；不能保证事实正确 |
| 卡片明显质量问题 | Python 规则 | `05_validate_cards.py` | draft card → 通过/问题/标签 | 本地短路，减少模型调用 |
| 边界语义归类 | LLM | `05_validate_cards.py` | 待判卡批次 → 语义标签 | 默认批次大小 12；有语义缓存；不能描述为每卡实时模型裁决 |
| Wiki 渲染 | Python 模板/规则 | `06_render_wiki.py` | validated cards → Wiki 页与索引 | 可从正式卡重建 |
| 查询理解 | Python 规则 | `query_understanding.py::build_*_evidence_frame` | 平台/指标/症状/反信号 → EvidenceFrame | 没有 LLM query rewrite |
| local 召回和排序 | Python 算法 | `retriever.py` | QuerySpec + RAG docs → 分类型排名 | 完全不需 Qdrant/embedding 模型；不是向量语义检索 |
| Qdrant 候选召回 | FastEmbed + Qdrant | `vector_store.py`、`qdrant_retriever.py` | text → dense/sparse vectors → candidates | 模型/索引不可用时，调用方可选 local 后端；未确认代码存在自动 fallback |
| 二阶段精排 | Python 规则 | `reranker.py::build_local_hybrid_rerank_map` | Qdrant candidates → 重排结果 | `rerank_mode=none` 可关闭 |
| Agent context | Python 回表/编排 | `final_reporter.py` | 检索命中 → 结构化证据包 | 不负责生成最终自然语言诊断结论 |

## 7.2 缓存与版本：分别解决什么问题

### 语义验证缓存

- 所在阶段：`05_validate_cards.py`。
- 用途：避免同样的边界卡在重复验证时再次请求 LLM。
- 优化方式：先用本地规则/启发式处理明显卡，再把剩余卡按默认 12 张打包为一个 LLM 请求。
- 当前代码中的语义 Prompt 版本：`2026-06-23-v2`；它会进入 cache key，连同模型名和待审 payload 共同决定是否命中缓存。

- 面试准确说法：**缓存的是语义验证结果，不是所有抽卡结果，也不是通用 LLM 对话缓存。**

### FastEmbed 模型缓存

- 所在位置：`vector_store.py` 的 dense/sparse 模型加载；Dockerfile 的构建期预热。
- 默认模型：dense `thenlper/gte-large`，sparse `Qdrant/bm25`。
- Docker 在 build 时暂时允许下载，把模型缓存到 `FASTEMBED_CACHE_PATH=/opt/fastembed_cache`；运行时设置 `HF_HUB_OFFLINE=1` 和 `TRANSFORMERS_OFFLINE=1`。
- 解决问题：受限网络环境中，首个构建/检索请求不会因临时下载或 Hugging Face 网络超时卡死。
- 不应夸大：这是镜像级模型预热，不是在线 embedding 结果缓存。

### 构建 manifest

- 文件：`qdrant_manifest.json` 与 `build_report.json`。
- `build_report.json` 锚定 cards/wiki/rag docs 等本次构建路径；`qdrant_manifest.json` 锚定 collection、dense/sparse 模型和索引配置。
- 解决问题：多输出目录或模型变更时，不会悄悄读取旧语料、也不会用不同 encoder 查询已有索引。

## 7.3 并发控制分层

| 资源 | 机制 | 代码/位置 | 解决与不能解决的问题 |
| --- | --- | --- | --- |
| 单条来源人工编辑 | 乐观锁/版本检查 | `registry_db.py`、`webapp/app.py` | 防止两个编辑者静默互相覆盖；不替代流程级写锁 |
| Gateway MCP session | `threading.Lock` 保护内存字典 | `api.py::_McpSessionStore` | 同进程线程安全；多副本仍会 session 丢失 |
| 本地 Qdrant | 进程内互斥 | `service_facade.py` | 缓解同进程目录锁冲突；不能跨进程/容器 |
| 工作区写任务 | MySQL job + lease + 全局写锁 | `workspace_jobs.py`（Gateway） | Worker 失效后可重新 claim，串行化 cards/wiki/rag/qdrant 的修改 |
| 向量索引发布 | 临时目录/原子替换或远端 alias | `vector_store.py` | 避免半成品成为正式索引；本地目录模式仍不适合多副本同时访问 |

### 一个容易说错的边界

`knowledge_asset_pipeline/scripts/workspace_store.py` 有 job/lease/lock/release 基础能力，但 01–06 主链路与 webapp 并没有直接调用它的证据；实际对外写请求的入队与 Worker 控制流在 **Gateway 的** `workspace_jobs.py`。不要把两者混成“所有上游脚本自动走 job”。

## 7.4 重试、fail-fast 与实际降级

### 重试

- `02c_retry_failed_fetches.py` 依据已记录的 `fetch.strategy` 分流，网页回 02、视频回 02b。
- 价值：按失败来源原始策略重试，不把所有失败 URL 粗暴丢给同一种抓取器。

### Fail-fast

- 显式传错 `cards_path`、`wiki_root`、`rag_docs_path` 或 `qdrant_path` 时，RAG CLI/API 直接报错而不是静默使用默认旧产物。
- Gateway 的上游 adapter 还会做 prerequisite 检查和子进程后产物核验。
- 价值：宁可暴露配置错误，也不返回“看似正常、实际来自旧数据”的结果。

### 降级：已确认与未确认

| 说法 | 结论 |
| --- | --- |
| 视频无字幕时转音频 ASR | **已实现的降级** |
| `cards_validated.jsonl` 缺失时回退 `cards_draft.jsonl` | **已实现的输入回退** |
| Qdrant 失败自动改用 local | **不要宣称自动发生**；local 是可显式选择的独立后端，需调用方选择 |
| LLM 抽卡失败自动生成确定性知识卡 | **未实现** |
| 语义校验失败自动认为通过 | **不可宣称**；应以校验输出/失败状态为准 |

## 7.6 业务成功与进程成功


Gateway 的 `adapters/asset_pipeline.py` 不只依赖 subprocess exit code：

1. 先按 task 白名单构造固定命令；
2. 执行时设置超时，截断 stdout/stderr；
3. 运行结束后通过 `_verify_stage_outcome` 等检查 source 状态、JSONL 或 Wiki 等真实产物；
4. 任一阶段不满足即 fail-fast，不继续后续 pipeline。

面试表达：这避免了“脚本返回 0，但因为筛选、空结果或状态未更新，实际没有产物”的假成功。
