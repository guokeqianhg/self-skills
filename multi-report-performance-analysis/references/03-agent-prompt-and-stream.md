# 03. Agent 装配、Prompt SOP 与流式执行

## 1. `buildAgent()` 的组成

对应文件：`src/agent/common/multi-report-analysis-agent.ts`。

MultiReportAnalysisAgent 使用 LangChain 的 `createAgent()` 创建 ReAct Agent。它的输入是模型、MCP 工具和中间件，不传 checkpointer。

```ts
return createAgent({
  model,
  tools,
  middleware: [
    buildSystemPromptMiddleware(...),
    createTokenUsageMiddleware(...),
    toolDedupeMiddleware(),
    toolRetryMiddleware({ maxRetries: 3 }),
    reasoningFallbackMiddleware(),
    structuredOutputRetryMiddleware(...),
  ],
});
```

对比 Agent 的工具来源只有 MCP。没有 memory、session search、skill 或 HITL 工具，原因是一次性任务无需对话能力。

## 2. 中间件逐层作用

### `buildSystemPromptMiddleware`

将 compare 人设、完整 SOP、当前时间、业务约束注入 system message。使用名称 `compare-report`，fallback identity 是 `COMPARE_REPORT_FALLBACK_IDENTITY`。

Anthropic provider 才启用 `cache_control`，其他 OpenAI 兼容网关传该字段可能 400，或出现空 SSE。

### `createTokenUsageMiddleware`

从每次 AIMessage 的 `usage_metadata` 累加：input、output、total、cache read、cache creation、reasoning token 和模型调用次数。MultiReportAnalysisAgent 最终只将 input/output token 交给 callback writer。

### `toolDedupeMiddleware`

将 `(toolName, args)` 生成去重 key。args 用稳定序列化：对象 key 按字典序排列，数组保留顺序，因此 `{a:1,b:2}` 与 `{b:2,a:1}` 会被视为相同调用。

命中重复时不调真实 MCP，返回一个伪 ToolMessage，提醒模型上文已有结果，应该直接输出或改变参数。memory/skill 等工具在默认白名单；MultiReportAnalysisAgent 只有 MCP 工具，因此其主要价值是阻断模型对同一 case、同一指标、同一时间段的反复查询。

### `toolRetryMiddleware`

工具调用失败最多重试 3 次。它放在 toolDedupe 后面，重复调用属于模型逻辑问题，不应该被当作“工具失败”重试。

### `reasoningFallbackMiddleware`

部分代理会把最终回答写入 `additional_kwargs.reasoning_content`，但 content 为空。该中间件在无 tool_calls 且 content 为空、reasoning 非空时，将 reasoning 回填 content。它也会兜底某些网关返回非 AIMessage/Command 的异常对象。

### `structuredOutputRetryMiddleware`

仅非 Anthropic 启用。负责最终输出 JSON 的抽取、Zod 校验、分类重试和后处理。详细实现见 `04-structured-output-and-schema.md`。

## 3. 为什么不用 todoListMiddleware

代码注释记录了实际观察：在 GLM-5.2 等长 reasoning 模型上，todoListMiddleware 会让模型把“更新待办”视作一个独立工具调用。模型在每个 SOP 阶段前先写一次 todos，之后才调真实工具，导致空 turn、reasoning token 和整体时长明显增加。

多报告任务的步骤已经写在 prompt 中，任务又不是交互式协作，所以去掉 todoListMiddleware 是减少无效推理的取舍。

## 4. HumanMessage 的构建

`compare()` 使用 `buildAnalysisInputText()` 生成输入，包含：

- 每份报告的 seq、caseId、项目、包名、版本、平台、机型、系统、报告名、时间和 labelIds；
- comparisonType 与原始 compareType；
- businessContext 或 userPrompt。

Legacy 路径只传 `caseIds` 时，代码会按数组顺序补 `{seq: i+1, caseId}`。这是兼容旧调用方式。

## 5. Prompt 的分析阶段

Prompt 文件：`src/agent/prompt/multi-report.ts`。

### A：基础事实采集

每个 case 要调用：

```text
metrics_get_report_data(case_id)
metrics_list_labels(case_id, include_statistics=true)
```

Prompt 要求所有 case 同一轮并行发出。目标是先建立可比事实：目标 FPS、FPS/帧时间、Jank、内存、算力、热功耗、平台元信息。

### B：异常识别

模型根据 A 阶段识别 FPS 达标率、BigJank、Stutter、内存 slope、峰值 CPU/GPU、温度等异常。

### B′：知识库方向指引

可选调用 `knowledge_search`。它不代替本次数据，只帮助决定阶段 C 应验证什么。

跳过条件：所有报告达标，或异常均发生在启动前 3 秒。重复测试、MIXED 和 UNRELATED 固定跳过。跨版本、跨设备只查询最严重 case 一次；跨平台、跨游戏按报告并行 N 次。返回卡片要筛平台、指标锚点和观测模式，匹配差时宁可全部丢弃。

### C：异常下钻

主要工具：

| 工具 | 适合场景 | 返回重点 |
|---|---|---|
| `metrics_analyze_root_cause` | FPS/帧时间显著下降 | drop 段、相关性、候选根因、综合评分 |
| `metrics_get_drop_context` | BigJank/Stutter 等下降事件 | 时间区间、相关指标统计、相关系数 |
| `metrics_get_time_range_detail` | 需要逐帧细节 | 指定时段更细数据 |
| `metrics_get_time_series` | 需要趋势 | 完整或片段时序 |

`metrics_analyze_root_cause` 已经返回 `in_drop`、`root_cause_candidates`、`related_metrics` 和 `correlations`，因此 prompt 明确要求不要为了“再确认”无意义地重复拉时序。

### D：最终输出

异常都有归因后进入单向门，要求一次性填写模块 1~9。除“某报告从未下钻”或“某异常未归因”外，不允许回到工具调用阶段。

## 6. `streamOnce()` 的执行方式

```ts
const stream = await agent.stream(
  { messages: inputMessages },
  {
    streamMode: ['messages', 'values'],
    signal: abortController.signal,
    configurable: { thread_id },
    recursionLimit: 30,
  },
)
```

- messages mode：使用 `handleStreamEvent()` 提取正文和 reasoning，转发到 writer proxy。
- values mode：每个图节点结束后取得完整 state.messages，借助 `getCurrentTurnMessages()` 和 `emitToolCallsFromMessages()` 识别并发出的工具调用与结果。
- 最终不把 raw content 作为权威结果；完成后从最后一条 AIMessage 的私有字段读取结构化对象。

## 7. 上游错误重试

`streamOnce()` 的上游重试不同于 JSON 重试。它针对空响应、5xx、连接重置、超时等瞬态故障，最多 2 次。退避为约 1 秒、2 秒并加入 0~500ms 抖动。

每次重试都重新创建：

- `AbortController`
- `writerProxy`
- `emittedToolCalls`
- content/reasoning 累积变量

这是为了防止上一轮中断时的半截流或已发出的工具状态影响下一次完整执行。

## 8. 预算的真实含义

Prompt 内写有：目标 8 分钟、最长 15 分钟、A≤2 轮、C≤2~3 轮、总≤6 轮、工具调用达到 10 次后停止下钻。它们用于引导模型节奏。

代码层最明确的限制是 `recursionLimit=30`、工具去重、工具重试 3 次、上游重试 2 次。复习时不要将 prompt 数字误说成服务端完整硬超时实现。