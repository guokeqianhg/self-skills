# 08. 核心函数、变量与数据流索引

> 目的：无法访问源码时，用本文件快速回忆“这个函数在哪里、输入是什么、输出是什么、解决什么问题”。函数名后都附实际功能解释，避免只记住变量名。

## 1. HTTP 与任务启动

| 名称 | 文件 | 输入 | 输出/副作用 | 解决的问题 |
|---|---|---|---|---|
| `multiReportAnalysisRoutes()` | `src/api/modules/multi-report.ts` | Fastify app | 注册 `POST /analyzeComparison` | 对外暴露多报告分析接口。 |
| `COMPARISON_TYPE_MAP` | 同上 | 数字 1~7 | 内部 comparisonType | 将业务协议中的数字翻译成 Agent 可理解的场景枚举。 |
| `runWithLogger()` | logging | 当前请求 logger、异步函数 | 后台日志保留上下文 | HTTP 已返回后，后台 Agent 日志仍带 requestId/sessionId/uid。 |
| `AnalysisCallbackWriter` | `src/stream/analysis-callback-writer.ts` | taskId、回调地址 | done 或 failed 回调 | 将 Agent 最终结果转换成业务方可消费的一次性 HTTP 回调。 |

## 2. Agent 与 MCP

| 名称 | 输入 | 输出/副作用 | 实际功能 |
|---|---|---|---|
| `MultiReportAnalysisAgent.createMCPClient()` | uid 或 token、配置中的 MCP URL | `MultiServerMCPClient` | 创建连接 性能分析平台 MCP 的客户端；有 uid 用 `X-Platform-User-Id`，否则用 `X-MCP-Token`。 |
| `MultiReportAnalysisAgent.buildAgent()` | mcp、token 用量回调 | LangChain `ReactAgent` | 拉取 MCP 工具，清洗 Schema，选择模型，装配中间件。 |
| `MultiReportAnalysisAgent.compare()` | threadId、uid/token、cases、compareType、业务上下文 | writer 事件 | 多报告任务总入口：建 MCP、建 Agent、组织输入、运行流、取结果、后处理、关闭 MCP。 |
| `buildAnalysisInputText()` | 报告列表、comparisonType、业务上下文 | HumanMessage 文本 | 将请求参数整理成模型能理解的任务说明，避免模型缺少报告身份或比较目的。 |
| `sanitizeToolsSchema()` | MCP tools | 原地修改 tools Schema | 删除或折叠不同 provider 不支持的 JSON Schema 字段，减少工具调用前 400。 |

## 3. 中间件

| 名称 | 触发点 | 实际功能 | 注意边界 |
|---|---|---|---|
| `buildSystemPromptMiddleware()` | 每次模型调用前 | 注入 compare 系统提示、时间、身份和 Anthropic cache 控制。 | Prompt 是模型指令，不等于代码强制流程。 |
| `createTokenUsageMiddleware()` | 每次模型调用后/Agent 结束 | 汇总输入、输出、缓存、reasoning token。 | 回调只上报 input/output token。 |
| `toolDedupeMiddleware()` | 每次工具调用前 | 用稳定序列化的 tool+args 检测重复调用，命中后返回伪 ToolMessage。 | 只阻断完全相同参数；更换时间窗或指标仍允许调用。 |
| `toolRetryMiddleware()` | 工具执行失败 | 最多重试 3 次。 | 处理工具错误，不处理模型 JSON 错误。 |
| `reasoningFallbackMiddleware()` | 模型返回后 | content 为空、reasoning 有文本且无工具调用时，把 reasoning 回填为 content。 | 处理部分代理的字段放置异常。 |
| `structuredOutputRetryMiddleware()` | 最终无 tool_calls 的 AIMessage | 提取 JSON、Zod 校验、分类提示、最多 2 次重试。 | Anthropic 原生 JSON Schema 分支关闭该中间件。 |

## 4. 流式执行与失败恢复

| 名称 | 输入 | 输出/副作用 | 实际功能 |
|---|---|---|---|
| `streamOnce()` | ReactAgent、输入 messages、threadId | 最终 messages | 运行 `agent.stream()`，转发中间 reasoning/工具事件，处理上游瞬态错误重试。 |
| `isRetryableUpstreamError()` | 任意 error | boolean | 沿 cause 链判断空响应、5xx、超时、连接重置等是否值得重试。 |
| `makeWriterProxy()` | writers 数组 | 单个 StreamWriter | 将一次模型流的 reasoning、内容、工具调用、结果广播给所有 writer。 |
| `emitToolCallsFromMessages()` | 当前 turn messages | writer.onToolCall/onToolResult | 从 state 快照补发工具调用和工具结果，避免仅依赖 token 流丢失工具事件。 |
| `readStructuredResponse()` | 最后 AIMessage | 结构化对象或 null | 从中间件写入的 private key 读取最终 JSON。 |

## 5. 结构化输出

| 名称 | 输入 | 输出/副作用 | 实际功能 |
|---|---|---|---|
| `multiReportSchema` | 原始 JSON | Zod parse result | 定义顶层、报告元信息和 9 模块数据结构。 |
| `repairCompareReportShape()` | 原始对象 | 修复后的对象 | 将 module8/module9 常见的裸数组包装成必须的对象结构。 |
| `finalizeCompareReport()` | 已解析 JSON、完整 messages | 最终报告对象 | 将 Schema、事实回填、结论复审串起来。 |
| `enrichWithSeqAndRef()` | 报告对象、请求 cases | 补充 seq/ref 的对象 | 用请求层身份信息覆盖模型可能写错的报告顺序和前端引用。 |

## 6. 工具事实解析与数值回填

| 名称 | 输入 | 输出/副作用 | 实际功能 |
|---|---|---|---|
| `extractToolContext()` | 全部 messages | `MultiReportToolContext` | 遍历 ToolMessage，按 caseId 收集报告统计、drop events、相关指标和相关系数。 |
| `parseToolMessagePayload()` | 单个 ToolMessage | 解析后 payload/null | 将 MCP 返回字符串转换成对象，处理非 JSON 或异常返回。 |
| `normalizeCompareReport()` | 模型报告、messages | 已归一化报告 | 调用核心数值、Low0.1%、内存占用、异常规则、异常项、watchReport 等校正逻辑。 |
| `normalizeCoreMetricsFromToolData()` | report、tool context | 原地覆写模块 2~6 | 用工具统计值替换模型生成数值；缺工具事实时将关键字段置 null。 |
| `resolvePlatformMetricSet()` | 某报告的原始统计 | 指标候选集合 | 按 Android/iOS/Windows/macOS 选择内存、CPU、GPU、温度、功耗口径。 |
| `pickFirstAvailableMetric()` | 工具数据、候选指标列表 | 可用 metric/null | 在当前平台候选里选真正存在于数据源的指标。 |
| `normalizeLow01Pct()` | report、tool context | 原地更新 low01pct | 只保留满足 `0 < low01 <= low1 <= avg` 的可信值。 |
| `normalizeMemoryOccupancy()` | report、tool context | 原地更新 totalDeviceMB/occupancyPct | 从 metadata 读取可信整机内存，计算峰值占用率。 |
| `normalizeAnomalyRule()` | report、drop 参数 | 原地更新异常规则文本 | 用实际 drop threshold 与 duration 写规则，避免写出未使用的默认口径。 |
| `normalizeAnomalyItems()` | report、tool context | 原地更新 module7.items | 有 drop context 时重建异常；无证据时保留模型原异常。 |
| `buildAnomalyItem()` | 单条 drop event | 异常对象 | 将工具下降事件转换成前端可渲染异常项，并按有限规则给出候选原因。 |

## 7. 根因与建议复审

| 名称 | 输入 | 输出/副作用 | 实际功能 |
|---|---|---|---|
| `buildModule8And9ImpactFacts()` | report | 扁平事实字典 | 抽取会影响根因/建议的核心数值、评级、异常，便于校正前后比较。 |
| `buildModule8And9ReviewContext()` | 原始 report、归一化 report | changedFacts/null | 判断是否存在需要复审的事实变化。 |
| `requestModule8And9Decision()` | 报告、变化事实 | KEEP/REVISE/null | 用辅助模型低成本判断根因与建议是否仍需修改。 |
| `requestModule8And9Revision()` | 归一化 report | 修订对象/null | 用主模型只重写模块 8 和模块 9。 |
| `parseModule8And9Decision()` | 模型文本 | 判定/null | 校验轻量模型的 JSON 是否包含合法 KEEP/REVISE。 |
| `parseModule8And9Revision()` | 模型文本、当前报告 | 修订/null | 校验修订内容，拒绝缺字段或不合法模块。 |
| `applyModule8And9Revision()` | 基础报告、修订 | 最终报告 | 只覆盖被接受修订的模块，保留不合格或未改变模块。 |

## 8. 回调交付

| 名称 | 输入 | 输出/副作用 | 实际功能 |
|---|---|---|---|
| `onResult()` | 最终报告对象 | 缓存结果 | 不立即发 HTTP，等待任务正常结束。 |
| `onComplete()` | token 用量 | done 回调 | 将缓存报告与 token 用量发送给业务方。 |
| `onError()` | Error | failed 回调 | 发送失败原因，并将 writer 置 settled。 |
| `sendDone()/sendFailed()` | 回调 body | HTTP POST | 处理最终 payload、长度截断和请求头。 |
| `settled` | boolean 状态 | 防重复 | 确保同一任务不会重复 done，也不会 done 后再 failed。 |