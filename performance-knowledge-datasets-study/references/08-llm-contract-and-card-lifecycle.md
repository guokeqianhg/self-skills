# 8. LLM 合同与知识卡生命周期：模型到底做什么，代码如何约束它

> 本篇用于回答“Prompt 怎么设计”“模型输出怎么校验”“一张卡如何从原文变成可检索证据”“为什么不能把模型输出当事实”。

## 8.1 模型调用在全链路中的位置

```text
清洗 Markdown
  → 03 规则校验正文是否可处理
  → 04 LLM 抽取 `ExtractionResult(cards[])`
  → 本地规范化/过滤/指标收紧/来源绑定
  → cards_draft.jsonl
  → 05 规则校验、优先级、warning
  → 仅边界样本调用 LLM 做轻量语义标签
  → cards_validated.jsonl
  → Wiki / rag_docs / Qdrant / Agent context
```

关键原则：**04 的模型负责从正文中提取候选知识；05 的模型只负责固定枚举的局部语义标签；最终检索和 Agent context 的组装不是 LLM 调用。**

## 8.2 04 抽卡：Prompt、Schema、解析与修复

源码：`knowledge_asset_pipeline/scripts/04_extract_cards.py`。

### Prompt 设计的真实目标

`SYSTEM_PROMPT`（116 行起）明确要求模型不是总结文章，而是把来源拆成能支持“指标解释、异常判断、瓶颈归因、优化建议”的原子知识卡。核心约束：

1. 只提取正文明确支持的事实/机制/验证/优化；无证据不生成。
2. 分开现象、根因、验证、优化、案例、平台差异、指标定义，禁止为了凑完整闭环编造另一半。
3. 单个汇总指标不能直接断言根因；必须保留 `contradicting_signals` 和 `cannot_conclude`。
4. 禁止编造函数、调用栈、阈值、收益百分比。
5. `metrics` 必须是允许指标名；泛概念无法映射时放 `unmapped_signals`。
6. `evidence_excerpt` 必须极短；内容以中文转述为主。
7. 纯接入教程、无指标/现象/验证/优化关系的 UI 实现细节不产卡。
8. 一段正文可以产生多卡，也可以产生 0 卡。

这是“少而硬”而不是“尽可能多抽卡”的设计，目标是降低未来 RAG 中的噪声与过度归因。

### 输出合同

`OUTPUT_SCHEMA_PROMPT`（149 行起）要求顶层严格为：

```json
{"cards": ["KnowledgeCard", "..."]}
```

`models.py` 定义了真正的 Pydantic 合同：

- `ExtractionResult.cards: list[KnowledgeCard]`；
- `KnowledgeCard.card_type` 仅允许七类：`metric_definition`、`phenomenon_pattern`、`root_cause`、`verification_method`、`optimization_action`、`case_evidence`、`platform_difference`；
- `root_cause_categories` 限定为 rendering/cpu/memory/thermal/io/platform/config/unknown 等枚举；
- `generalization_level` 限定为 source/platform/engine/general principle；
- `SourceClaim` 必须带转述 claim、证据位置、短 excerpt、claim type；
- 信号、步骤、动作、风险、限制等必须是数组。

### 输出解析和修复控制

- `extract_json_object`（342 行）先剥除 Markdown code fence，再取首尾 JSON object；没有合法 `{...}` 就抛错，避免把聊天解释当作结构化数据。
- `extract_cards_with_chat_completion`（761 行）调用 Chat Completions 后执行 `ExtractionResult.model_validate_json`。
- 首次 Pydantic 解析失败时，代码有 repair 路径，要求模型按 schema 修复 JSON，再重新 `model_validate_json`；这只是**格式修复**，不是事实校验。
- `split_markdown`（235 行）优先按标题边界分块；小节超长才硬切，降低把概念或证据截断在 chunk 边界的风险。

### 半完成恢复

`load_usage_completion_state`（272 行）从 `extract_usage.jsonl` 重建每个来源已处理 chunk 状态；发现某来源上次只处理了部分 chunk 时，会通过 `purge_source_records` 清理该来源半成品后重抽。`audit_extraction_outputs`（309 行）还会报告：有 usage 但未完成、没有 usage、完整处理但 0 卡三类异常。

因此准确表述是：**抽卡具备 chunk 级进度审计和半成品清理重跑，不是对每个模型调用做无条件自动重试。**

## 8.3 05 语义校验：模型只输出固定标签

源码：`knowledge_asset_pipeline/scripts/05_validate_cards.py`。

### 为什么还要第二次调用模型

04 负责从资料中产生候选卡，05 的核心仍是本地规则。但“英文混杂是否影响中文可读性”“语气是否过强”“工具引用是否构成依赖”“内容主体是否只是采集方法”有模糊边界，规则难以稳定覆盖。因此 05 对**边界卡**调用 LLM 进行轻量标签，而不是重新做业务判断。

### 固定标签 Schema

`SemanticTagResult`（206 行）和 `SemanticBatchResultItem`（219 行）只允许：

| 字段 | 枚举 | 意图 |
| --- | --- | --- |
| `language_mix` | clean / mixed_but_acceptable / mixed_problematic | 判断中文输出是否被过长英文污染 |
| `claim_tone` | neutral / slightly_strong / overclaim | 标出无条件、绝对化归因风险 |
| `tool_reference_role` | none / supplementary_reference / dependency | 区分工具仅补充还是结论依赖外部工具 |
| `content_role` | general_reasoning / tool_capture_methodology / measurement_or_capture_constraint / mixed_or_unclear | 区分诊断知识与采集/测量约束 |

`SEMANTIC_SYSTEM_PROMPT`（166 行起）明确写了“只负责局部文本标签，不负责业务评审、不判断知识价值、不推断项目背景”；信息不足必须取最保守标签。这个限制非常适合面试时强调“模型只在它擅长的模糊分类处介入”。

### 规则、候选、缓存、批处理的顺序

```text
结构规则/指标别名/来源约束
→ `semantic_task_flags` 判断哪些标签需要模型
→ 无候选：直接 skipped
→ 已有 cache key：cache_hit
→ 模式非 required：heuristic_fallback 或本地默认
→ 需要模型：构造单卡或批量 payload → Chat Completions → Pydantic validate
→ 解析异常：repair JSON → 再验证
→ 写 validation 结果和 semantic cache
```

相关函数：

- `load_semantic_cache`（532 行）：读缓存记录；
- `semantic_cache_key`（544 行）：key 纳入 `SEMANTIC_PROMPT_VERSION`、模型名和 payload；
- `semantic_task_flags`（1068 行）：决定 `language_mix`、`claim_tone`、`tool_reference_role`、`content_role` 哪些值得进一步判断；
- `heuristic_semantic_tags`（1128 行）：本地启发式基线与模型调用候选筛选；
- `build_semantic_review_payload`（1146 行）：只发送需要检查的字段；
- `semantic_review_with_chat_completion`（1222 行）：单卡调用及 JSON repair；
- `resolve_semantic_tags`（1240 行）：缓存、skipped、heuristic fallback、required 模式的分支；
- `build_batched_card_stats`（1347 行）与批量解析函数：支撑批量语义调用。

当前 `SEMANTIC_PROMPT_VERSION` 是 `2026-06-23-v2`。默认批次大小为 12（CLI/主流程配置）。

## 8.4 卡片生命周期与不可夸大的结论

```text
原文事实
  → [LLM 候选提取]
草稿卡
  → [Pydantic + 本地归一/规则 + 局部语义标签]
已验证卡
  → [可见性 `rag_excluded` 决定是否入 RAG]
RAG 文档
  → [检索和规则精排]
证据包
  → 下游 Agent/LLM 可据此推理
```

每个状态的含义：

| 状态/产物 | 代表什么 | 不代表什么 |
| --- | --- | --- |
| `cards_draft.jsonl` | 模型提取后并完成基础后处理的候选 | 已被人工确认或绝对正确 |
| `cards_validated.jsonl` | 通过当前校验策略的正式上游输入 | 真实世界根因已经被证明 |
| `human_reviewed` | 有人工审核状态 | 所有字段均具备严格实验因果证明 |
| `rag_excluded=true` | 不进入新语料和召回 | 已删除或 Wiki 不可见 |
| top-K 命中 | 与输入症状/证据较相关的知识 | 当前待测性能问题的最终裁决 |
| Agent context | 有字段结构和来源的证据底座 | 已完成最终自然语言报告 |

## 8.5 面试可直接使用的回答

**问：如何避免 LLM 幻觉？**

先限制任务而非事后祈祷：04 Prompt 明确只抽正文支持内容、要求反证和不可判断项、禁止补阈值和收益；输出必须过 Pydantic schema，且指标只能落允许集合，未映射信号单独保存。05 再以规则优先、模型只做四类固定标签的方式审查边界内容。最后检索结果仍保留 source claims、验证步骤与风险，Agent 不能把命中直接包装成测量事实。

**问：为什么 05 不用 LLM 直接判断一张卡是否有价值？**

“有没有价值”会混入业务目标与模型主观性，难以稳定回归。代码把确定性约束交给规则，把模型限制为语言混杂、语气强度、工具依赖、内容角色四个枚举标签；这样输出可验证、可缓存、可批量、可追溯。

**问：Pydantic 够不够保证质量？**

不够。Pydantic 只保证结构和枚举，不保证事实正确或因果成立。因此还有正文预校验、来源证据、指标收紧、warning、语义标签、review status 和最终的人工审核边界。
