# 0. 先讲主线：从一条资料到一次可解释回答

> **先读本篇，再读其它 reference。** 目标不是记住文件名，而是能顺着同一条故事讲清输入、硬逻辑、模型判断、输出和价值。

## 0.1 一句话、30 秒、2 分钟三个版本

### 一句话

这是一个将分散的 性能采集平台 性能资料加工成**可追溯、可审核、可检索，并能输出诊断证据包**的知识工程：上游治理知识，下游混合检索，网关统一提供 HTTP/MCP 与异步任务入口。

### 30 秒版本

系统的输入是专家链接清单或单条 URL。它先用规则登记来源并在 MySQL 保存人工编目和处理状态；再抓网页或获取视频字幕/ASR 文本，做质量校验；LLM 从正文抽取结构化性能知识卡，规则和批量 LLM 复核后生成 Wiki。下游按 `card_id` 把卡片和 Wiki 合并为 RAG 文档，既提供 BM25+词频余弦+领域规则的 local 检索，也提供 Qdrant dense/sparse 候选召回和规则精排。最终返回带来源、验证步骤、优化动作和风险的 Agent context，而不是只返回几段文本。

### 2 分钟版本

它解决的不是“没有向量库”，而是性能知识来源分散、质量不一、人工编目会被脚本覆盖、检索结果缺少证据和验证路径的问题。架构上分为：`asset_pipeline` 负责来源到知识卡/Wiki 的治理；`hybrid_retrieval` 负责统一语料、混合检索和报告打包；`orchestration_gateway` 负责 API/MCP 和长写任务编排。

上游首先从 Markdown 清单或单条 URL 获取来源。URL 归一化、来源类型、平台/引擎/指标初判都是确定性规则；MySQL `sources` 是运行时真源，Web 管理端只写人工字段并通过乐观锁保护。网页走 网页抓取服务，视频优先字幕，缺失时才走音频和 ASR。正文先经规则质量校验，再由 通用模型兼容接口 LLM 按 Pydantic schema 抽取现象、根因、验证、优化等知识卡；05 阶段将确定性问题留给规则，仅将边界样本按批交给 LLM 语义打标，并使用版本化缓存。

下游构建时按 `card_id` 对齐 validated cards 与 Wiki，过滤 `rag_excluded`，生成带 `dense_text`、`sparse_text` 和证据字段的 `rag_docs.jsonl`。local 后端是可解释的 BM25、Counter cosine 和领域规则；Qdrant 后端用 FastEmbed 做 dense/sparse 候选召回，默认把候选交回 `local_hybrid` 做二阶段精排。最终 `final_reporter` 回表补齐来源主张、验证步骤、优化动作和风险，供 Agent 使用。服务层对包化 RAG 直接调用函数，对脚本型上游用白名单子进程适配；所有修改共享资产的 API 都是入队，Worker 通过 lease 和写锁执行，避免长 HTTP 请求和并发写冲突。

## 0.2 一条主线：输入 → 分析 → 模型 → 输出

```text
输入 A：Markdown 专家资料清单       输入 B：单条 URL       输入 C：检索诊断请求
          │                               │                         │
          └─────── 来源登记（硬逻辑） ──────┘                         │
                           │                                          │
                  MySQL sources（真源）                               │
                           │                                          │
       网页抓取 / 视频字幕优先、ASR 兜底（外部服务 + 硬逻辑）            │
                           │                                          │
            raw + meta + cleaned Markdown（可追溯中间资产）            │
                           │                                          │
              正文质量校验（硬逻辑）                                   │
                           │                                          │
       LLM 抽卡（模型）→ Pydantic/后处理（硬逻辑）                      │
                           │                                          │
    规则短路 + 批量 LLM 语义复核（硬逻辑决定是否调用模型）                │
                           │                                          │
          validated cards + Wiki（正式上游产物）                        │
                           │                                          │
  按 card_id 对齐、排除 rag_excluded、构建 RAG 文档（硬逻辑）            │
                           │                                          │
  local：BM25/词频余弦/规则       qdrant：embedding/候选召回             │
                           └──── local_hybrid 二阶段精排 ─────────────┤
                                                                        │
                                                  结构化证据包/Agent context
```

## 0.3 必须牢记的“硬逻辑 / 模型 / 外部依赖”边界

| 环节 | 分类 | 代码做的事 | 不应夸大成 |
| --- | --- | --- | --- |
| URL、source id、平台/引擎/指标初判 | 硬逻辑 | 归一化、SHA-1、关键词/章节启发式 | LLM 自动理解来源 |
| 网页获取 | 外部服务 | 网页抓取服务 抓取并存 raw/meta/Markdown | 系统自己理解网页 |
| 视频文本 | 外部服务 + 模型 | 字幕/API 优先；必要时音频+ASR | 对视频内容做完整语义理解 |

| 正文校验 | 硬逻辑 | 空/长度/有效性等规则 | LLM 质量判断 |
| 知识卡抽取 | LLM | Chat Completions 生成结构化卡草稿 | 所有字段绝对真实 |
| 卡片结构与明显问题 | 硬逻辑 | Pydantic、规则、启发式短路 | 由模型决定全部质量 |
| 边界语义标签 | LLM | 批量语义打标、版本化缓存 | 逐卡实时调用模型 |
| query understanding | 硬逻辑 | 别名、标签、EvidenceFrame | 通用 LLM 意图理解 |
| local 检索 | 硬逻辑 | BM25 + Counter cosine + 领域规则 | 真正 dense embedding |
| Qdrant 检索 | 向量模型 + 硬逻辑 | FastEmbed 向量、filter、fusion、规则 rerank | 直接产出最终根因 |
| 最终 context | 硬逻辑 | 命中回表、字段分组、风险补齐 | LLM 自动生成结论 |

## 0.4 输出是什么，价值在哪里

| 输出 | 消费者 | 价值 |
| --- | --- | --- |
| MySQL `sources` | 管理端/流水线 | 来源状态可查询、人工编辑不被覆盖 |
| raw/meta/Markdown | 排障/重跑 | 保留原始证据与处理轨迹 |
| draft/validated card JSONL | 审核/下游 | 将散文资料变成可检索诊断单元 |
| Wiki | 人 | 可阅读、可审阅、可 Git diff |
| `rag_docs.jsonl` + manifest | RAG | 固定一次检索构建的输入和模型合同 |
| 检索 package / Agent context | Agent/服务 | 返回来源、验证、优化和风险，不只是片段 |
| job/release 状态 | 运维 | 长任务可跟踪、失败可恢复 |

## 0.5 45 分钟阅读路径：从看懂到能讲

1. **第 1–8 分钟：**读本篇的 30 秒和 2 分钟版本，复述一次主线。
2. **第 9–16 分钟：**读 `00-workspace-architecture.md`，记住四类真源/派生资产和 `rag_excluded`。
3. **第 17–27 分钟：**读 `01-knowledge-asset-pipeline.md`，回答“资料如何变成知识卡，LLM 在哪里介入”。
4. **第 28–36 分钟：**读 `02-hybrid-retrieval.md`，回答“local 与 Qdrant 各做什么，为什么两阶段”。
5. **第 37–42 分钟：**读 `03-workspace-orchestration-gateway.md`，回答“为什么分 adapter、为什么写任务入队”。
6. **第 43–45 分钟：**读 `05-interview-review.md` 的陈述和最后速记。

需要追问时，不重新通读：函数/调用链查 `06-code-level-function-map.md`；硬逻辑与模型判断查 `07-decision-reliability-map.md`；字段、接口和部署查 `04-contracts-deployment-inventory.md`。
