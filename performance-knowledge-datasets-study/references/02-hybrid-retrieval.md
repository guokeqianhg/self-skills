# 2. `hybrid_retrieval_service`：可解释检索基线与向量检索

## 2.1 模块边界

这是上游知识资产的消费层。输入是 `cards_validated.jsonl`（不存在时可回退 draft）、`wiki/` 和索引页；输出是统一语料、Qdrant 索引、结构化检索包、Agent context 与报告。包级导出收敛在 `src/hybrid_retrieval_service/__init__.py`。

```text
builder → rag_docs.jsonl
  ├─ retriever → local 检索
  ├─ vector_store → Qdrant 索引
  └─ qdrant_retriever → 向量/混合召回
                    ↓
             final_reporter → 给 LLM 的证据包
                    ↓
             service_facade → HTTP/MCP 中性合同
```

## 2.2 语料构建：`builder.py`

### 输入与对齐

- 读 JSONL，选择 validated cards，缺失时回退 draft。
- 自动发现 `wiki/index.md`、`wiki/index_core.md` 和卡片页。
- 通过 `card_id` 把结构化 card 与对应 Wiki 文本对齐；不要把 Wiki 仅当作附件，它包含更适合全文检索和人读的上下文。
- 若 `rag_excluded=true`，跳过该卡但在 `build_report.json` 记录跳过数量与 card id。

### 输出文档设计

每条 RAG document 将稳定 metadata、检索文本和下游证据字段合并。常见检索字段包括：

- `dense_text`：更适合 embedding 的拼接文本；
- `sparse_text`：更适合 BM25 的词面文本；
- 平台、引擎、指标、card type、priority、review status、warning；
- `required_signals`、`supporting_signals`、`anti_signals`；
- 来源主张、验证步骤、优化动作、风险与适用性。

同时写 `build_report.json`，记录实际 cards/wiki 路径、rag docs 路径、索引路径等。这是避免 CLI 在多套输出目录中误读默认旧文件的关键锚点。

## 2.3 文本与查询理解

### `text_utils.py`

- 做文本清洗、平台/引擎别名归一。
- 处理中英文混合切词，构造可用于 BM25、词频计数的 token。
- 使用 `Counter` 计算简化 token cosine。它不是语义 embedding，只是 local baseline 的可解释 dense-like 特征。

### `query_understanding.py`

将自由文本或结构化输入归一为 `EvidenceFrame` / `QuerySpec`：

- 提取平台、引擎、指标、症状、反信号；
- 识别性能现象和诊断意图；
- 对别名做统一，避免 Android/安卓、Unity/Unity3D、平均帧率/avgFps 断裂；
- 组织检索所需的 structured signals。

这层规则化理解的价值是：用户输入未必是高质量检索 query，但性能诊断有固定术语与证据维度；把它们转换成 fields 能让后续过滤、加权和报告结构稳定。

## 2.4 Local 检索：`retriever.py`

### 为什么保留 local

它是一个完全本地、可解释、低依赖的基线与 fallback，不依赖模型下载/向量 DB。更重要的是可用于校准和解释 Qdrant 的结果。

### 算法

1. 按 query token 和每个 `sparse_text` 计算 BM25；
2. 对 `dense_text` 使用 token-counter cosine；
3. 叠加领域规则：
   - 平台/引擎软匹配；
   - required/supporting signal 命中加分；
   - P0 / `index_core` 加分；
   - warning、冲突信号等惩罚；
   - review status、card type、优先级过滤或偏好；
4. 按总分排序，并按类型返回现象、根因、验证、优化等卡片集合。

这里要准确表达：**它不是“BM25 + 真实 dense embedding”**，而是 BM25 加 token-counter cosine 的规则化 baseline；真实 embedding 在 Qdrant 分支。

## 2.5 Qdrant 建库：`vector_store.py`

### 默认配置

- dense model：`thenlper/gte-large`；
- sparse model：`Qdrant/bm25`；
- collection：`knowledge_rag_cards`；
- 默认索引目录：`data/qdrant`；
- manifest：`data/qdrant_manifest.json`。

### 建库过程

1. 校验显式传入的 rag docs 路径；显式路径错了就 fail-fast，不静默回退。
2. 读取 docs，生成 dense 和 sparse vectors。
3. 将 metadata/payload 与向量写入 Qdrant collection。
4. 本地目录建库使用临时目录/原子切换思路，尽量避免半成品索引替换正式索引。
5. 产出 manifest，保存模型名、collection、来源路径等；查询优先读取 manifest，防止“建库模型”和“查询模型”不一致。
6. 远端 Qdrant 场景可使用 collection alias 切换，实现较安全的版本替换。

## 2.6 Qdrant 查询与二阶段精排：`qdrant_retriever.py` / `reranker.py`

### 一阶段召回

- 支持 dense；也支持 dense + sparse hybrid/fusion。
- 将 platform、engine、card type、priority、review status 等条件映射为 Qdrant filter，缩小候选集。
- 一阶段目标是高召回，不直接等价于最终诊断排序。

### 二阶段：`local_hybrid`

默认 `rerank_mode=local_hybrid`：拿 Qdrant 的候选集合，复用 `reranker.py` 与 local 的证据/质量规则重新排序。目的：语义模型擅长“相关”，领域规则擅长“哪个证据更适合当前性能诊断”。传 `rerank_mode=none` 才关闭。

**面试亮点**：这是经典 retrieve-then-rerank，但 reranker 不是另一个黑盒大模型，而是可解释的领域 hybrid 规则，成本更低、结果更可审计。

## 2.7 最终报告：`final_reporter.py`

检索结果只包含排序信息不足以让下游 Agent 可靠回答，因此 final reporter：

1. 根据 `build_report.json` 回表到上游 cards；
2. 将命中分成现象模式、根因、验证方法、优化动作等；
3. 补齐 source claims、verification steps、optimization actions、risks、applicability；
4. 构造机器消费的 `agent_context`；
5. 可输出 Markdown 人读报告和 JSON package。

这层把“召回卡片”升级成“有来源、验证路径和风险的证据包”，降低 LLM 自行拼接结构时遗漏反证或把建议当事实的风险。

## 2.8 中性服务门面：`service_facade.py`

- 归一化 request payload，构造 `QuerySpec`。
- 提供 health、检索 payload、agent context 和 MCP tool schema。
- 对本地 Qdrant 访问加进程内互斥，减轻目录 lock 冲突；它不能替代多进程/多副本分布式锁。
- 统一异常到可调用服务的错误响应。

## 2.9 CLI 与评测

| 脚本 | 作用 |
| --- | --- |
| `build_rag_docs.py` | 只构建语料与 build report |
| `build_qdrant_index.py` | 根据语料/manifest 建索引 |
| `build_full_rag.py` | 顺序执行完整构建 |
| `retrieve_cards.py` | local/Qdrant 检索 CLI |
| `generate_final_report.py` | 输出结构化 agent context / 报告 |
| `smoke_test_retrieval.py` | 比较 local、Qdrant raw、Qdrant rerank，检查非空/一致性 |
| `generate_nl_test_report.py` | 固定自然语言 case 的检索回归与 Markdown 报告 |

`smoke_test_retrieval.py` 是对检索质量的回归保护，但它不是端到端业务正确性的充分证明；自然语言测试也依赖启发式关键词评价，应结合人工抽检。

## 2.10 风险、限制、可改进点

- 本地 Qdrant 目录要求串行访问；不要并发建库、检索、报告生成。
- 需评估 embedding 模型对中文/性能术语的适配，可增加领域向量模型或 query expansion。
- 当前 local rules 和权重应有离线标注集、MRR/Recall/NDCG 及版本化实验，而不只靠 smoke test。
- 可将二阶段从规则扩展为 cross-encoder 或 LLM rerank，但需控制延迟、成本和可解释性。
- 报告应标注“检索证据”和“模型推断”，不能把卡片证据直接当实时性能结论。
