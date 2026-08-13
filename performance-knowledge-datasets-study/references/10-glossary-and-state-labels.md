# 10. 术语索引与状态标记

> 复习和面试时统一使用以下术语，避免把数据库真源、知识卡、RAG 文档、在线索引和发布版本混为一谈。

## 10.1 统一状态标记

| 标记 | 含义 | 面试表达 |
| --- | --- | --- |
| **[当前代码已实现]** | 当前源码已有可调用路径和持久化/响应证据 | “当前实现已经……” |
| **[当前有限实现]** | 能工作，但有明确单机、内存、无鉴权或合同不完整限制 | “当前版本已具备基础能力，但……” |
| **[推荐生产化演进]** | 合理的下一步，不在当前代码行为内 | “如果上线到多人/生产环境，我会……” |

禁止把第三类说成第一类。例如 JWT、跨副本 MCP session store、远端 Qdrant 集群、统一幂等键和自动指数重试都是演进建议，不是当前默认能力。

## 10.2 规范命名与路径映射

以下是脱敏后唯一应使用的名称。**物理目录/包名**用于理解源码布局，**逻辑简称**用于架构、Job 类型和面试表达；两者不是不同组件。

| 层级 | 规范名称 | 对应路径或运行时标识 | 使用规则 |
| --- | --- | --- | --- |
| 离线 Skill | `performance-knowledge-datasets-study` | 当前文档包根目录 | 只指本复习档案，不等同于原工程源码目录。 |
| 原工程根 | `performance_knowledge_workspace` | 审阅时的工作区根；原始绝对路径不保留 | 用于表达“三工程同属一个工作区”。 |
| 知识资产构建层 | `knowledge_asset_pipeline` | 物理目录/包根；逻辑简称 `asset_pipeline` | 负责来源、抓取、抽卡、校验和 Wiki；Job 前缀为 `asset_pipeline.*`。 |
| 混合检索层 | `hybrid_retrieval_service` | 物理目录及 Python 包 `hybrid_retrieval_service`；逻辑简称 `hybrid_retrieval` | 负责语料、索引、检索和发布；Job 前缀为 `hybrid_retrieval.*`。 |
| 编排网关层 | `workspace_orchestration_gateway` | 物理目录；运行时 service 为 `knowledge_workspace_gateway` | 负责 REST、MCP、Job、Worker 和 adapter。 |
| 单写锁 | `knowledge_workspace_single_writer` | MySQL named lock | 只串行化共享资产写入，不代表全系统没有并发。 |
| MCP 工具 | `knowledge_workspace_*`、`asset_pipeline_*`、`hybrid_retrieval_*` | `tools/call` 名称 | 以 `09` 的工具表为准；不要从目录名猜工具名。 |
| 环境变量 | `DATABASE_*`、`CRAWLER_API_KEY`、`MODEL_API_KEY`、`VIDEO_PLATFORM_COOKIE` | 部署期配置键 | 仅保留角色名称和用途；真实值、域名、环境路径不入档。 |

## 10.3 知识资产层

| 术语 | 精确定义 | 容易混淆为 |
| --- | --- | --- |
| **source** | 一条资料来源的注册记录；运行时真源在 MySQL `sources` | 一段抓取文本或一张知识卡 |
| **source key** | 规范化 URL，用于聚合候选来源 | 随机 source id |
| **source id** | `SRC-` + 规范 URL SHA-1 前缀的稳定来源标识 | URL 本身 |
| **raw asset** | 抓取/字幕的原始响应，保存到 `data/raw` | 已可供 RAG 检索的文本 |
| **meta asset** | 抓取策略、时间、状态、错误等元数据，保存到 `data/meta` | MySQL 的完整来源真源 |
| **cleaned Markdown** | 清洗后的正文，保存到 `data/markdown`，供 03/04 消费 | Wiki 页面 |
| **draft card** | 04 LLM 抽取后的候选知识卡，`cards_draft.jsonl` | 已被证明的知识 |
| **validated card** | 通过当前 05 校验策略的正式上游卡，`cards_validated.jsonl` | 实时性能问题的最终结论 |
| **card id** | 一张卡的稳定标识，也是 card 与 Wiki/RAG 对齐键 | source id；一来源可有多张卡 |
| **Wiki** | 由 cards 渲染的面向人阅读 Markdown 页面/索引 | RAG 语料快照 |
| **RAG document / rag doc** | builder 将 card+Wiki+检索/证据字段合成的下游检索记录 | 原始 card 或 Qdrant point |
| **rag_excluded** | 卡片保留但不进入新 `rag_docs.jsonl` 的可见性开关 | 删除卡片或删除 Wiki |

## 10.4 治理与质量层

| 术语 | 精确定义 |
| --- | --- |
| **review status** | 卡片审核状态；local 默认允许 `human_reviewed`、`auto_checked`、`draft`。它影响检索过滤/加分。 |
| **validation warning** | 05 产生的质量风险标识，例如外部工具依赖、指标锚点不足、语言混杂、语气可能过强；不是程序异常。 |
| **needs_manual_review** | 需人工进一步确认的审核状态/集合；RAG 构建默认不纳入，除非显式 `include_needs_manual_review`。 |
| **source claim** | 卡内可回溯到来源的主张、位置和短摘录；是 Agent context 的证据基础。 |
| **required/supporting/anti signal** | 分别表示成立前提、辅助支持和反证/排除线索；后两层检索规则会用它们加分或扣分。 |
| **P0/P1** | 卡片优先级。local 当前对 P0 有加分，并非数据库任务优先级。 |

## 10.5 RAG 与检索层

| 术语 | 精确定义 | 不应误说 |
| --- | --- | --- |
| **local backend** | 读 `rag_docs.jsonl`，执行 BM25 + Counter cosine + 领域规则 | “真正 dense embedding 检索” |
| **Counter cosine** | `dense_text` token Counter 的余弦分数；local 的可解释 dense-like 项 | FastEmbed semantic vector |
| **Qdrant backend** | 用 dense/sparse embedding、Qdrant filter/fusion 候选召回 | 自动根因裁决器 |
| **hybrid** | dense 与 sparse 共同参与 Qdrant 召回 | local BM25 与 Qdrant 的同一实现 |
| **fusion** | Qdrant hybrid 候选融合方法，当前支持 `dbsf` / `rrf` | 二阶段 rerank |
| **candidate limit** | Qdrant 一阶段候选池大小，默认 128 | 最终每类返回数 |
| **top_k** | 每个 `card_type` 返回的最终条数，默认 3 | 一阶段候选数 |
| **local_hybrid** | 将 Qdrant candidates 交给 local BM25/规则体系重排 | 另一个 LLM reranker |
| **rerank** | 候选召回后的第二阶段排序；可传 `none` 关闭 | 全库再次向量检索 |
| **EvidenceFrame** | 规则抽取的平台、指标族、正负信号、场景、趋势、干预、假说域的结构化表达 | LLM 思维链 |
| **Agent context** | final reporter 将检索结果回表、补齐验证/优化/风险后的机器可消费 JSON | 已完成最终自然语言诊断 |

## 10.6 服务、发布与运行时层

| 术语 | 精确定义 |
| --- | --- |
| **gateway** | 工作区统一 transport/编排层；不重写上游 ETL 或 RAG 算法。 |
| **adapter** | 连接 gateway 与子工程的薄层。`hybrid_retrieval` 是直接 Python 调用；`asset_pipeline` 是受控子进程。 |
| **run task** | 执行一个白名单上游脚本 task；单任务 `success` 基本表示子进程 exit code 为 0。 |
| **ingest** | `add_source → 抓取分流 → 03 → 04 → 05 → 06` 的组合 pipeline；会额外核验该 source 的真实产物/状态。 |
| **Job** | MySQL `workspace_jobs` 的异步工作单元；HTTP 写接口的成功只表示 Job 已入队。 |
| **worker** | 轮询 claim queued Job，并执行 adapter/publish 的进程。 |
| **lease** | running Job 的租约到期时间；Worker 每 60 秒续 300 秒租约；过期 job 在下一次 claim 时重新入队。 |
| **single writer lock** | MySQL `GET_LOCK('knowledge_workspace_single_writer')`；串行化共享 assets 修改。 |
| **attempts** | job 成功被 claim 的次数；不是当前自动 retry policy。 |
| **active release** | `asset_releases` 中当前激活版本，提供 cards/wiki/rag/Qdrant 路径给后续读取。 |
| **publish release** | build 成功后创建/激活 release；Gateway Worker 对 ingest/delete/visibility 成功 Job 默认会 follow-up full publish。 |
| **snapshot** | 某阶段可审计、可重建的文件或版本产物，例如 validated cards、rag docs、manifest。 |
| **rebuild** | 从上游资产重新派生下游产物；需明确是 `corpus`、`qdrant` 还是 `full`，不能笼统说“重建 RAG”。 |

## 10.7 一句话辨析

```text
source 是“资料登记”；raw/Markdown 是“资料加工中间资产”；
card 是“原子化知识”；Wiki 是“给人看的卡片页面”；
rag doc 是“给检索器吃的 card+Wiki 快照”；Qdrant 是“索引和候选召回载体”；
Agent context 是“给下游模型的结构化证据包”；Job 是“异步执行单元”；release 是“当前可读取的资产版本”。
```
