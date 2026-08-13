---
name: performance-knowledge-datasets-study
description: This skill should be used when reviewing, explaining, maintaining, debugging, demonstrating, or preparing interviews for a privacy-sanitized knowledge-asset pipeline, hybrid retrieval service, and workspace orchestration gateway.
---

# 性能知识资产工程：代码复习与面试技能

这是脱离源码保存的离线代码复习手册，而不是功能使用说明。所有结论均来自整理时审阅的源码快照，描述实际实现边界，不把规划能力当作已实现能力。

## 审阅基线与脱敏边界

- **审阅基线**：`performance_knowledge_workspace` 的离线源码快照；原分支、commit 和原始绝对路径未随本 Skill 保留。
- **最后核验日期**：2026-08-13。函数行号、字段和默认值均对应该快照；后续若获得新源码，应重新核验并更新本日期。
- **脱敏规则**：产品、仓库、域名、凭据和人员/环境标识已匿名化；目录、类、函数、数据库、协议和算法术语保留为可讨论实现的通用标识。
- **状态口径**：`[当前代码已实现]`、`[当前有限实现]`、`[推荐生产化演进]` 的定义见 `references/10-glossary-and-state-labels.md`，所有现状表述以此为准。

## 先建立全局模型

严格按以下顺序阅读，避免先记碎片再自行拼图：

1. 阅读 `references/00-main-story-and-study-plan.md`：先掌握一句话、30 秒、2 分钟讲法，以及一条 URL 和一次检索请求如何走完整链路。
2. 阅读 `references/00-workspace-architecture.md`：确认三层职责、数据真源、派生产物与完整调用链。
3. 阅读 `references/01-knowledge-asset-pipeline.md`：复习来源治理、网页/视频抓取、抽卡、验证、Wiki 渲染。
4. 阅读 `references/02-hybrid-retrieval.md`：复习语料对齐、查询理解、本地与 Qdrant 检索、二阶段精排、报告打包。
5. 阅读 `references/03-workspace-orchestration-gateway.md`：复习 HTTP/MCP、受控子进程、异步 Job、Worker、发布。
6. 碰到“规则还是模型、缓存/锁/重试/降级在哪里”的追问，先查 `references/07-decision-reliability-map.md`。
7. 碰到函数、调用方、算法权重或具体文件位置的追问，查 `references/06-code-level-function-map.md`。
8. 需要 REST/MCP 字段、Job 状态机、lease、失败或幂等边界时，查 `references/09-api-mcp-job-contract.md`。
9. 碰到术语、命名、当前实现与生产演进的边界时，查 `references/10-glossary-and-state-labels.md`。
10. 需要 Schema、依赖或部署资产时，查 `references/04-contracts-deployment-inventory.md`；面试前最后读 `references/05-interview-review.md` 与 `references/08-llm-contract-and-card-lifecycle.md`。


## 回答代码问题的工作方式

- 先按问题所属层选择相应 reference；跨层问题始终回到 `00` 的端到端数据流核对。
- 明确区分**事实实现**、**设计取舍**、**现有限制**和**可演进方案**。例如上游工程是知识资产构建层，不是在线向量检索服务；Qdrant 本地目录模式不是多副本共享方案。
- 解释函数时固定覆盖：输入/输出、核心算法或控制流、依赖的持久化对象、错误处理和调用方。
- 解释一次“新增资料后为什么不能马上检索”时，指出 `ingest_source` 默认不重建 RAG；只有上游卡片/Wiki 已更新，需再执行语料与索引构建。
- 解释并发时，区分 MySQL 任务与来源的并发控制、Qdrant 本地目录串行限制、HTTP/MCP 进程内会话限制，避免笼统表述“系统是线程安全的”。

## 面试表达约束

- 用“知识资产构建与治理 + 可解释混合检索 + 服务编排”概括项目，避免夸大为已完整上线的智能诊断闭环。
- 强调双轨检索的动机：`local` 是可解释回退与对照基线；Qdrant 负责语义候选召回；`local_hybrid` 做二阶段领域精排。
- 强调渐进式服务化：对包化的 `hybrid_retrieval` 直接函数调用；对脚本化的 `asset_pipeline` 用白名单子进程适配，避免 HTTP 输入变成任意命令执行。
- 必须主动说出目前边界：无鉴权、长任务超时风险、本地 Qdrant 不适合多副本、MCP session 在进程内、上游脚本主流程缺少一个全自动总编排器。

## 资料索引

| 文档 | 重点 |
| --- | --- |
| `00-main-story-and-study-plan.md` | **先读**：统一项目故事、三种讲法、45 分钟复习路径 |
| `00-workspace-architecture.md` | 三层架构、真源、端到端调用链、阅读路径 |
| `01-knowledge-asset-pipeline.md` | 01–06 ETL、MySQL、抓取/LLM/校验细节 |
| `02-hybrid-retrieval.md` | 语料、查询理解、BM25、向量、精排、报告 |
| `03-workspace-orchestration-gateway.md` | REST、MCP、Job、Worker、adapter 安全边界 |
| `04-contracts-deployment-inventory.md` | Schema、接口、配置、Docker、全源码清单 |
| `05-interview-review.md` | 2 分钟陈述、高频问答、亮点和改进项 |
| `06-code-level-function-map.md` | 函数签名、调用关系、算法权重、追问索引 |
| `07-decision-reliability-map.md` | 硬逻辑/模型边界、缓存、并发、重试、降级 |
| `08-llm-contract-and-card-lifecycle.md` | 抽卡 Prompt/Schema、校验 Prompt、卡片生命周期 |
| `09-api-mcp-job-contract.md` | REST/MCP 字段、Job 状态机、lease、失败与幂等边界 |
| `10-glossary-and-state-labels.md` | 统一术语、当前实现/有限实现/生产演进标记 |



