---
name: multi-report-performance-analysis
description: This skill should be used when reconstructing, studying, explaining, interviewing about, or extending the 性能分析平台 multi-report AI performance comparison module without access to its source code. It is a source-level technical archive of /analyzeComparison, MultiReportAnalysisAgent, MCP tools, prompt SOP, report schema, numerical normalization, retries, callbacks, and design trade-offs.
---

# 性能分析平台 多报告性能对比：源码级技术档案

## 本 Skill 的定位

将本 Skill 当作 `performance-analysis-agent` 多报告性能对比模块的离线源码笔记，而不是功能使用手册。目标是在无法访问仓库时，仍可复原以下内容：

- 请求从 `/analyzeComparison` 到一次性 HTTP 回调的完整调用链；
- `MultiReportAnalysisAgent` 如何构建 LangChain Agent、连接 MCP、消费流式状态；
- prompt 如何要求模型采集、判断、下钻和输出；
- Zod Schema、结构化重试、工具数据回填、平台指标口径和 module8/9 二次复审的实现逻辑；
- 每项设计的业务问题、代码路径、真实边界和面试表达。

## 阅读顺序

首次复习时，按以下顺序阅读：

1. `references/01-architecture-and-call-chain.md`：建立整体架构和数据流。
2. `references/02-http-mcp-and-callback.md`：理解接口、参数校验、MCP 生命周期和回调交付。
3. `references/03-agent-prompt-and-stream.md`：理解 Agent、中间件、SOP、工具调用和流式重试。
4. `references/04-structured-output-and-schema.md`：理解 9 模块 JSON、Zod 和结构化重试。
5. `references/05-fact-normalization.md`：理解数值回填、平台口径、异常项重建。
6. `references/06-conclusion-revision-and-resilience.md`：理解 module8/9 复审和失败恢复。
7. `references/07-interview-and-resume.md`：把源码细节转成面试与简历表达。
8. `references/08-function-and-data-index.md`：按函数、变量和数据流快速定位实现职责。

## 命名约定

为保证脱敏后的档案可回查且术语一致，统一采用以下规则：

- **模块与基础设施**：使用 `MultiReportAnalysis` / `multiReportAnalysis`，文件路径使用 `multi-report`；路由函数为 `multiReportAnalysisRoutes()`，回调 Writer 为 `AnalysisCallbackWriter`，回调配置为 `internal.analysisCallback`。
- **领域模型**：保留 `CompareReport` 及其领域操作，如 `repairCompareReportShape()`、`finalizeCompareReport()`、`normalizeCompareReport()`；`comparisonType` 表示报告间的比较语义。
- **执行入口与请求字段**：保留 `MultiReportAnalysisAgent.compare()` 和请求字段 `compareType`，前者表示执行一次报告比较任务，后者是调用方传入的比较场景；二者映射到领域字段 `comparisonType`。
- **身份透传**：入口、MCP 和回调统一使用 `X-Platform-User-Id`；无 uid 的 MCP 鉴权使用 `X-MCP-Token`。
- **HTTP 路由**：统一为 `POST /analyzeComparison`。

上述名称均为脱敏后的规范名；文档中不再混用旧名称或旧路径。

## 事实边界

区分三类内容：

- **代码强保证**：路由参数校验、`seq/ref` 注入、Zod 结构校验、工具事实回填、有限重试、回调互斥。
- **模型/PROMPT 约束**：阶段 A 并行采集、总轮次、8/15 分钟预算、知识库筛选、阶段 D 不再调工具。它们是模型执行指令，不是全部由程序强制执行。
- **当前主链路未实现**：截图按时间戳关联、多模态视觉识别、视觉 CoT、视觉 Function Call。不得把这些写成 MultiReportAnalysisAgent 已实现能力。

## 原始代码定位

| 模块 | 源文件 |
|---|---|
| HTTP 接口 | `src/api/modules/multi-report.ts` |
| Agent 主体 | `src/agent/common/multi-report-analysis-agent.ts` |
| 系统 Prompt / SOP | `src/agent/prompt/multi-report.ts` |
| 报告 Schema | `src/agent/schema/multi-report/index.ts` |
| 结构化输出重试 | `src/agent/middlwares/structured-output-retry-middleware.ts` |
| 模型工厂 | `src/agent/llm/factory.ts` |
| 回调 Writer | `src/stream/analysis-callback-writer.ts` |

## 使用原则

- 解释实现时先说明输入、处理过程、输出和失败分支，再解释所用技术。
- 面试表达时从业务问题切入，穿插 `Fastify`、`LangChain`、`ReAct`、`MCP`、`Zod`、`ToolMessage`、`指数退避` 等技术名词，但避免抽象黑话。
- 不虚构准确率、时延、线上调用量或成本收益；仅使用代码可验证的数字和真实实测数据。
