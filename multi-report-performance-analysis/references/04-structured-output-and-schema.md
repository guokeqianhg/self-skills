# 04. 结构化报告、Zod Schema 与定向重试

## 1. 这一层解决什么问题

多报告结果要被前端按模块、按报告列展示，不是仅供人阅读的一段 Markdown。因此模型输出必须满足固定的数据结构：前端需要知道每份报告是谁、某个指标属于哪份报告、异常项应该挂在哪一列、根因和优化建议如何展示。

MultiReportAnalysisAgent 使用 `multiReportSchema` 约束最终 JSON。这个 Schema 基于 Zod 定义；Zod 是 TypeScript 中的运行时数据校验库，作用类似 Python 的 Pydantic：不仅提供类型提示，还能在程序运行时检查模型实际返回的数据是否符合字段结构。

模型输出会经历四步：先从模型消息中提取 JSON，再修复少量可以机械修复的层级错误，然后用 Zod 校验，最后执行数值回填和结论复审。这样可以区分两类问题：格式问题由程序或重试处理；性能结论问题由工具事实和后续模型复审处理。

## 2. 顶层报告结构

`multiReportSchema` 的顶层包含：

```text
generatedAt                 生成时间，ISO 字符串
title                       报告标题
summaryConclusion           报告摘要
correlation                 本次报告是否适合横向对比，以及原因
reports                     2~5 份报告的元信息
module1 ~ module9           九个分析模块
```

`correlation` 包含：

- `level`：`HIGH` 或 `LOW`。它表示本次报告之间的可比性，不表示性能好坏。
- `reason`：说明为什么可比或不可比，例如同游戏、同平台、同版本的重复测试通常具有高可比性；混合设备和混合版本的报告需要说明限制。

`reports` 是前端表格的基础。每项包括：

| 字段 | 含义 | 为什么允许为空或强制存在 |
|---|---|---|
| `reportId` | 性能分析平台 的 caseId | 用于把各模块指标与某份报告关联，必须存在。 |
| `seq` | 请求中 `cases[].seq` 的顺序 | 决定前端 r1、r2 等列顺序，必须存在，但最终由程序覆盖。 |
| `label` | 展示名称 | 前端标题，必须存在。 |
| `gameName/platform/deviceModel/version` | 报告元信息 | 工具或请求缺少时可为 null。 |
| `targetFps` | 此报告的目标帧率 | 评级和达标率的基准，必须存在。 |
| `targetFpsSource` | `user_specified` 或 `inferred` | 说明目标帧率是业务方指定，还是模型从上下文推断。 |
| `targetFpsRationale` | 目标帧率理由 | 记录用户要求或推断依据，方便解释评分口径。 |

### `seq` 和 `ref` 为什么由程序处理

模型可能漏写 `seq`，也可能把报告顺序写错。`enrichWithSeqAndRef()` 会根据 HTTP 请求中的 `cases` 建立 `caseId -> seq` 映射，再覆盖 `reports[].seq`。

随后它为和报告有关的对象附加 `ref: 'r{seq}'`，例如：

- 模块 1 的 bestReport、worstReport、watchReport；
- 模块 2~6 的 `perReport` 指标卡；
- 模块 7 的异常项；
- 模块 9 的优化建议项。

`ref` 不在 Zod Schema 中，因为它是前端展示字段，不是模型分析字段。程序在 Schema 校验和数据归一化后补上它，保证前端可以稳定按 r1/r2 对齐。若模型输出了请求中不存在的 reportId，代码只记录 warn，不会为它注入 seq/ref。

## 3. 九个模块的字段与业务含义

### 模块 1：总体结论

`module1_overallConclusion` 包含：

- `comparisonType`：7 类对比场景之一；
- `summary`：总体结论；
- `bestReport`、`worstReport`：报告 ID 和 1~5 条理由；
- `watchReport`：不是最差、但有独立风险的报告；
- `keyFindings`：3~5 条关键发现；
- `notes`：最多 3 条限制说明；
- `comparabilityNote`：混合或无关报告时说明为什么不能做强结论。

best/worst/watch 允许为 null。并不意味着模型漏答：完全不相关的报告没有可靠的全局最佳或最差；没有额外风险时也不需要 watchReport。`normalizeWatchReport()` 会清除不存在的 reportId，也会清除与 worstReport 重复的 watchReport，避免“最差报告”被重复贴成“关注报告”。

### 模块 2：FPS

每个 `perReport` 卡片有：平均 FPS、Low 1%、Low 0.1%、P95/P99 帧时间、帧时间方差、目标 FPS、达标率和评级。

原始指标多数允许 null，因为不同平台或不同报告不一定提供完整指标。`rating` 不允许为 null，无法评级时用 `"N/A"`；这样前端能区分“某个数据缺失”和“整体无法评级”。`sparkline` 被 Schema 固定为 null，避免模型把大段时序数据塞进最终 JSON。

### 模块 3：流畅度

每个报告卡片包含小卡顿、大卡顿、总卡顿、Stutter、冻结次数和每小时 FPS 下降次数。模块还有 `platformCaliber`，专门说明 Android Jank 和 iOS Stutter 等字段不能简单等价。

### 模块 4：内存

每个报告卡片包含平均内存、峰值、测试开始到结束的变化量、每分钟增长速率、设备总内存、占用率、趋势和评级。

`trend` 是 `STABLE | GROWING | LEAKING | UNKNOWN`，不允许为 null；没有足够数据判断时用 UNKNOWN。数值字段允许 null，避免在缺少趋势或设备内存数据时补造数值。

### 模块 5：CPU/GPU

每个报告卡片包含 CPU/GPU 平均值和峰值、CPU 或 GPU 是否被识别为瓶颈、评级。具体数值由后处理根据平台口径回填，不完全信任模型生成。

### 模块 6：温度与功耗

每个报告卡片包含平均/峰值温度、平均/峰值功耗、是否检测到降频、每小时耗电率和评级。

这里的 null 很重要：当前 MCP 工具没有统一、可靠的字段可直接判断降频和耗电率，代码会将 `throttleDetected` 与 `batteryDrainPctPerHour` 设为 null，而不是根据温度峰值猜测“发生了降频”。

### 模块 7：异常项

包含异常规则和异常列表。异常项通常有 reportId、时间区间、指标、严重程度、偏离程度、相关原因和证据描述。若有 `metrics_get_drop_context` 的工具证据，程序会重建异常项；没有对应证据时保留模型原结果。完整逻辑见 `05-fact-normalization.md`。

### 模块 8：根因

`module8_rootCause` 必须是对象，核心字段是 `hypotheses` 数组。每条假设包含根因描述、证据、置信度以及可能适用的报告。它不是“确定故障结论”，而是基于当前数据形成的可验证归因。

### 模块 9：优化建议

`module9_optimization` 必须是对象，核心字段是 `items` 数组。建议项通常包含优先级、适用报告、问题、建议动作、预期收益和风险说明。模块 9 应与模块 8 的根因保持一致，因此数值回填后会进入定向复审。

## 4. 两种结构化输出路径

### Anthropic 路径

Claude 支持原生 JSON Schema。MultiReportAnalysisAgent 会传入 `multiReportJsonSchemaForAnthropic`：

```ts
outputConfig: {
  format: {
    type: 'json_schema',
    schema: multiReportJsonSchemaForAnthropic,
  },
}
```

这会让模型服务端尽量按 Schema 生成结果。此时应用层 `structuredOutputRetryMiddleware` 被关闭，避免重复处理。

### 非 Anthropic 路径

Kimi、Qwen、Gemini、GLM 或 OpenAI 兼容网关的原生结构化能力不完全一致，因此由 `structuredOutputRetryMiddleware` 在应用层完成 JSON 提取、Zod 校验和重试。

MultiReportAnalysisAgent 最后会从 AIMessage 的 `additional_kwargs.__structured_output_retry_response` 读取中间件挂载的结果。对 Anthropic 的 native structured response，当前代码依赖 LangChain 的消息形态；这是需要持续回归测试的兼容点，不能宣称所有 provider 的读取行为完全一致。

## 5. structuredOutputRetryMiddleware 的完整流程

中间件只在最终 AIMessage 没有 `tool_calls` 时处理。中间轮模型正在请求 MCP 工具时，content 为空是正常现象，不能误判为输出失败。

```text
最终 AIMessage
  ├─ 读取 content
  ├─ content 无法提取 JSON 时读取 reasoning
  ├─ parseJsonMarkdown / parsePartialJson 解析
  ├─ 从第一个 "{" 重新截取尝试解析
  ├─ preValidate：修复可确定的形状错误
  ├─ multiReportSchema.safeParse：运行时结构校验
  ├─ finalizeCompareReport：事实回填与结论复审
  └─ 将最终对象写入 additional_kwargs
```

### JSON 提取

`extractJsonFromText()` 先使用 `parseJsonMarkdown()`，可以兼容模型把 JSON 包在 Markdown 代码块里的情况。若整体文本失败，但存在 `{`，会从第一个 `{` 开始再次尝试，处理模型在 JSON 前输出短解释的情况。

### 三类失败

| 失败类型 | 判断条件 | 处理方式 |
|---|---|---|
| `no_data_collected` | messages 中没有 `metrics_*` ToolMessage | 提示模型回到阶段 A 采集报告数据和标签统计。 |
| `schema_failed` | JSON 能解析，但 Zod 校验失败 | 将最多 20 个字段路径和错误原因写进提示，要求只修 JSON。 |
| `no_json` | content 和 reasoning 都提取不到对象 | 提示模型基于已拿到的工具结果直接输出纯 JSON，不要 Markdown 或解释。 |

最多重试 2 次。这里不会重新建立 MCP 或重跑整段工具调用链，而是在已有对话和工具结果基础上修正最终输出。

## 6. 机械形状修复

`repairCompareReportShape()` 只修复明确且无语义歧义的层级错误：

```text
module8_rootCause = [ ... ]
→ module8_rootCause = { hypotheses: [ ... ] }

module9_optimization = [ ... ]
→ module9_optimization = { items: [ ... ] }
```

这类错误不需要重试模型。程序不会替模型补根因、补建议或猜缺失字段；它只修复数组和对象容器的误用。

## 7. finalizeCompareReport 的责任边界

`finalizeCompareReport(parsed, messages)` 是结构化层与事实层的连接点：

1. 执行 shape repair；
2. 重新运行 `multiReportSchema.safeParse()`；
3. 调用 `normalizeCompareReport()`，从 ToolMessage 回填模块 2~6 核心事实，校正异常项等；
4. 调用 `reviseModule8And9IfNeeded()`，检查根因和优化是否仍匹配校正后的数据；
5. 再次运行 Schema 校验。

如果初步 Schema 校验失败，当前实现会记录 warn 并返回修复后的 raw 对象作为降级结果。这体现的是“尽量有结果”的策略，但也意味着不能说每次回调都绝对满足完整顶层 Schema。