# 5. 面试复习：项目陈述、追问和改进路线

## 5.1 两分钟项目陈述

我做的是一套面向 性能采集平台 性能诊断知识的工程化知识库，不是简单把文档塞进向量库。整体分三层：上游用 MySQL 管来源状态，抓取网页和视频字幕，经过正文校验、LLM 结构化抽卡、规则和批量语义复核后输出 JSONL 知识卡和 Markdown Wiki；中间层把卡片与 Wiki 按 `card_id` 对齐为统一 RAG 语料；下游同时保留可解释的 BM25+词频余弦+领域规则 local 基线，以及 FastEmbed+Qdrant dense/sparse 混合检索。Qdrant 只负责高召回候选，默认再用 local_hybrid 根据性能信号、平台、引擎、P0、warning 做二阶段精排。最后把命中回表成带来源主张、验证步骤、优化动作和风险的 Agent context。

服务层用 HTTP/MCP 统一暴露能力。因为 RAG 是包化模块，所以直接调用 Python API；上游是编号脚本型 ETL，所以采用白名单的受控子进程 adapter，并用 MySQL job、lease 和全局写锁处理长写任务。主要工程收益是可追溯、可复现、可解释，并避免把 LLM 输出直接当成无审计知识。

## 5.2 高频深挖问答

### Q1：为什么既有 JSONL/Wiki 又有 MySQL？

MySQL 保存协作式、频繁变化的来源登记和流程状态，适合查询、乐观锁与 job；JSONL/Markdown 保存可审阅、可 Git diff、可离线交付的知识资产。两者职责不同，避免将所有内容塞进数据库而失去可读产物，也避免用纯文件承担并发状态机。

### Q2：为什么不直接“网页 → embedding → Qdrant”？

性能诊断需要来源、适用条件、反信号、验证方法和优化建议。直接 embedding 原文不利于治理和审计。中间的知识卡将文本变成结构化诊断单元，验证阶段过滤质量问题，RAG 只消费正式且可见的资产。

### Q3：`rag_excluded` 是删除吗？

不是。它是检索可见性开关。卡片仍可留在 validated JSONL 和 Wiki 供审阅，但 builder 生成 `rag_docs.jsonl` 时跳过。修改后必须重建语料；若要更新向量结果，还应重建索引。

### Q4：local 检索的“dense”是真 embedding 吗？

不是。local 使用 `dense_text` 上的 token-counter cosine，属于可解释的稀疏词频相似度；再与 BM25、领域规则融合。真实 dense vector 是 Qdrant 路径通过 FastEmbed 生成。

### Q5：双轨检索如何保证结果不冲突？

不承诺完全一致。local 是稳定、可解释的对照/fallback；Qdrant 解决语义召回。通过 smoke test 同时跑 local、Qdrant raw、Qdrant rerank，检查非空和关键 case 的合理一致性。默认 Qdrant 候选还会走 local_hybrid 二阶段精排，缩小语义漂移。

### Q6：为什么不让 Qdrant 一步出最终结果？

向量相似度只能说明语义邻近，不能完整表达性能诊断证据：平台/引擎约束、required/anti signals、P0 和 warning 的业务优先级。先高召回，再以可解释规则精排，成本低于 cross-encoder/LLM rerank，也更方便调试。

### Q7：BM25 和 sparse embedding 是否重复？

两者都关注词面信号但运行位置不同。local BM25 是离线可解释基线；Qdrant sparse embedding 用于向量库内 hybrid candidate retrieval。设计目标不是完全正交，而是在不同后端都保留关键词精确匹配能力。

### Q8：如何避免 embedding 建库和查询模型不一致？

建库写 `qdrant_manifest.json`，记录 dense/sparse 模型、collection 和路径；查询优先读取 manifest，不依赖调用方重新猜默认值。Docker 还在 build 阶段预热相同默认模型，运行期离线读取缓存。

### Q9：为什么要把 `build_report.json` 当作重要资产？

它是一次 RAG 构建的路径与版本锚点。多 output root 时，检索/报告可通过 build report 找到正确的 rag docs、cards 和索引，而不是悄悄退回根目录旧文件。显式错误路径也 fail-fast。

### Q10：抽卡如何控制 LLM 幻觉？

不能完全消除，采用分层限制：先清洗和正文质量校验；用 Pydantic 约束输出；保留 source claim；之后执行确定性规则和批量 LLM 语义复核；输出 review status、risk/anti-signals；RAG 返回证据而非声称实时结论。还可增加人工抽检和 prompt/model 版本化。

### Q11：为什么校验要“规则 + 批量 LLM”？

格式、必填字段、明显内容角色是确定性问题，使用 LLM 浪费且不稳定；边界语义需要模型判断。先用本地规则/捷径过滤，再以 batch size 12 合并边界卡，能显著降低请求数、成本和 latency；缓存避免重复判断。

### Q12：来源状态如何避免批量脚本覆盖人工字段？

`registry_db.py` 区分自动推断列、人工列、抓取列。`01_build_sources.py` upsert 时只刷新自动部分；webapp 用乐观锁写人工修改。这样重复同步 Markdown 清单不会把运营/审核判断抹掉。

### Q13：视频抓取失败如何处理？

02b 先尝试字幕和 API 路径，必要时音频下载、ffmpeg 转换与 ASR；策略和错误写回来源状态。02c 根据已记录的 strategy 分流重试，不把网页与视频来源混在一种重试逻辑里。

### Q14：Gateway 为什么不用一个框架而是 `ThreadingHTTPServer`？

当前实现选择标准库以降低依赖，核心业务不绑定框架。代价是缺少框架级验证、异步、可观测性与生产中间件；如果并发、鉴权和 streaming 需求上升，迁移 FastAPI 是合理演进。

### Q15：asset_pipeline adapter 如何防命令注入？

不接收任意命令。它将 task 限定在白名单，并为每个任务构造固定可执行脚本和允许参数；子进程有 timeout、stdout/stderr 截断和 prerequisite 检查。仍应对 URL 等外部输入做 SSRF/内容安全控制。

### Q16：exit code 为 0 就能判定 ingest 成功吗？

不能。子进程可能正常退出却没有生成 source/card/wiki。adapter 会在关键阶段查看真实产物或来源状态，属于业务级成功验证，而不仅是进程级成功验证。

### Q17：为什么需要 job/lease/全局写锁？

抓取、LLM、渲染、建库都是长且会改共享资产的操作。job 让 API 快速返回任务身份；lease 防止 worker 崩溃永久占有；全局写锁避免同时重写 cards/wiki/rag/qdrant；最终状态和错误落库供可恢复执行。

### Q18：本地 Qdrant 的并发限制如何解释？

本地目录模式依赖文件锁，多个进程/副本同时读写或建库可能冲突。进程内 mutex 只能缓解单进程，不能跨容器。单机单实例可用；多人/多副本应该使用远端 Qdrant、外部锁和更清晰的读写策略。

### Q19：HTTP MCP 和 stdio MCP 区别？

业务 tools 共用 runtime，差别在 transport。stdio 使用 Content-Length JSON-RPC framing，适合本地 IDE；HTTP `/mcp` 使用 session header，适合远程调用，但当前无 SSE、session 在内存，横向扩展要处理粘滞会话。

### Q20：当前最主要的安全缺陷？

无鉴权。任何可访问端口者都可能调用删除、构建、抓取。生产最少加认证、RBAC、审计、网络隔离，以及对 URL 抓取做 SSRF 防护、额度和超时限制。

## 5.3 可量化成果怎么说

可根据仓库实际产物陈述：有 47 份抓取 meta/Markdown、约 138 张 P0 主知识卡和数百 Wiki 页面等快照；但面试必须说明这些是**当前数据快照规模**，不是系统吞吐 SLA。性能优化可讲 `05_validate_cards.py` 的批量语义验证：本地预处理和缓存后，边界卡合并为单请求，减少模型调用次数和耗时。

## 5.4 系统设计题回答框架

如果被问“如何把它上线到多人团队”：

1. 把 MySQL 作为共享注册表与 job store；
2. 用远端 Qdrant 集群替代本地目录；
3. cards/wiki/rag 用对象存储或 PVC，发布采用版本目录/manifest/alias；
4. API gateway 加 JWT/RBAC、审计与限流；
5. Worker 分队列：抓取、ASR、LLM、索引构建隔离；
6. 构建产物用版本化与原子切换，检索只读 active version；
7. 加可观测性：每个 source/job/LLM 请求的 trace、耗时、token、失败原因；
8. 以标注 query set 做 Recall@K、MRR、NDCG、人工准确率回归。

## 5.5 项目亮点与坦诚改进

### 可强调亮点

- 从原始资料到 Agent context 的端到端可追溯链。
- MySQL 状态机与文件型知识资产的职责分离。
- 卡片级治理、`rag_excluded` 软下线。
- 可解释 local baseline + Qdrant hybrid + 二阶段 rerank。
- manifest/显式路径 fail-fast 防止索引错配和旧数据误用。
- 受控子进程、业务产物核验、job lease/锁的可靠性意识。
- Docker build 时 embedding 预热、运行时离线，解决受限网络冷启动。

### 必须坦诚的改进项

- 无系统化自动化测试和质量指标闭环；
- 本地 Qdrant 与内存 MCP session 不适合横向扩展；
- REST 写路由虽已切到 Job/poll，但仍缺回调/事件流、统一自动重试与幂等键；

- 无认证、RBAC、SSRF 防护和完善审计；
- 嵌入模型/规则权重需要基于标注集持续实验，不应凭主观调参。

## 5.6 最后一分钟速记

```text
Markdown 清单是批量种子；MySQL sources 是运行时真源。
validated cards + wiki 是上游正式资产；rag_docs 是下游语料快照。
local = BM25 + Counter cosine + 规则；Qdrant = dense/sparse 候选 + local_hybrid 精排。
rag_excluded 是软下线，改后至少重建语料。
Gateway 对 RAG 直接函数调用，对 ETL 用白名单子进程。
写任务用 job/lease/锁；本地 Qdrant 不可多副本并发；当前无鉴权。
```
