# 01. 总体架构与调用链

## 1. 模块解决的问题

多报告模块的输入不是一句自然语言，而是一组结构化报告：2~5 个 性能分析平台 `caseId`，外加业务方给定的对比类型和业务上下文。输出也不是聊天回复，而是一份可由前端直接渲染的 9 模块 JSON 对比报告。

它解决的核心问题可以拆成三段：

1. 从多份报告中采集相同口径的性能事实；
2. 找到真正的异常区间，再用时段内关联指标解释性能回归；
3. 让最终 JSON 的结构、数值和根因结论尽量稳定，而不是只依赖 LLM 一次生成。

代码的总体架构如下：

```text
Blueeye / 调用方
    │ POST /analyzeComparison
    ▼
Fastify multiReportAnalysisRoutes
    ├─ 请求校验、compareType 映射
    ├─ 创建 LogStreamWriter + AnalysisCallbackWriter
    ├─ runWithLogger 绑定请求日志上下文
    └─ 立即返回 accepted
             │ 后台异步
             ▼
       MultiReportAnalysisAgent.compare()
    ├─ 建立 MultiServerMCPClient
    ├─ buildAgent() 装配 LLM、MCP tools、中间件
    ├─ 组织 HumanMessage（报告集合 + 类型 + 业务上下文）
    ├─ agent.stream() 执行 ReAct 工具循环
    ├─ 从最终 AIMessage 读取结构化报告
    ├─ 事实回填 / 数值归一化
    ├─ module8/9 一致性复审
    └─ 注入 seq/ref
             │ writer.onResult + onComplete
             ▼
       AnalysisCallbackWriter
    └─ 仅一次 HTTP done / failed 回调
```

## 2. 为什么 MultiReportAnalysisAgent 是独立实现

单报告 `ReportAgent` 支持多轮对话、memory、skill、checkpointer 和可能的 HITL；多报告分析是一次性后台任务，输入是固定报告集合，输出是固定 JSON，执行完即结束。

`MultiReportAnalysisAgent` 因此不继承单报告的 BaseAgent，也不挂 checkpointer、memory、skill、format-error tool。这样做有三个直接影响：

- 不保存跨请求执行状态，不需要通过 threadId 恢复；
- 不让历史记忆影响本次报告，避免任务之间结论串扰；
- 可用工具只有 MCP 数据工具，模型的行动空间更小，分析路径更稳定。

这不是功能缺失，而是按任务形态做的隔离。后续如果需要“基于对比报告继续追问”，合理方式是将已完成 JSON 和工具调用历史交给聊天 Agent，而不是把一次性 MultiReportAnalysisAgent 改成带状态对话 Agent。

## 3. 从请求到回调的时序

### 3.1 受理

调用方发送 `POST /analyzeComparison`。路由校验成功后不等待模型完成，直接返回：

```json
{ "ret": 0, "msg": "accepted" }
```

原因是模型分析可能运行数分钟。将任务与 HTTP 请求解绑，可以避免上游超时或连接断开导致任务中断。

### 3.2 后台执行

路由用 `void runWithLogger(contextLogger, async () => ...)` 启动 `agent.compare()`。这里的 `runWithLogger` 利用 AsyncLocalStorage 延续请求日志上下文，使后台日志仍带有 requestId、sessionId、uid、taskId。

### 3.3 交付

Agent 成功拿到结果后调用 writer 的 `onResult()`，再调用 `onComplete()`；异常时调用 `onError()`。AnalysisCallbackWriter 内部先缓存结果，直到结束信号才向业务方发一次最终回调。

这种“缓存最终结果 + 结束时一次回调”与聊天 SSE 场景不同。多报告结果是完整 JSON，业务方通常不需要消费中间 token，而是需要一个可落库、可渲染的终态对象。

## 4. 7 类对比场景

接口将数字 compareType 映射为内部 comparisonType：

| 数字 | 枚举 | 业务含义 | 分析重点 |
|---:|---|---|---|
| 1 | `SAME_GAME_REPEAT_RUN` | 同游戏、同设备、同版本重复测 | 稳定性与复现性 |
| 2 | `SAME_GAME_CROSS_VERSION` | 同游戏跨版本 | 回归或收益 |
| 3 | `SAME_GAME_CROSS_DEVICE` | 同游戏跨机型 | 高中低端设备瓶颈 |
| 4 | `SAME_GAME_CROSS_PLATFORM` | 同游戏跨平台 | 平台差异与口径说明 |
| 5 | `SAME_DEVICE_CROSS_GAME` | 同设备跨游戏 | 横向竞品或内容差异 |
| 6 | `MIXED` | 多个维度同时变化 | 可比性有限，必须说明 |
| 7 | `UNRELATED` | 没有公共维度 | 不做强行横向结论 |

重要事实：调用方传入 `compareType`，后端将其映射为领域字段 `comparisonType`，而非由模型自动分类。模型会读取报告元信息辅助解释 `correlation.reason`，但不应覆盖该映射结果。

## 5. 关键数据结构在链路中的位置

```text
MultiReportRequestBody
  └─ cases: MultiReportCaseRequest[]
       └─ MultiReportCaseInput[]
            └─ buildAnalysisInputText()
                 └─ HumanMessage
                      └─ Agent messages / ToolMessage
                           └─ MultiReportToolContext
                                └─ CompareReport（Zod schema）
                                     └─ enrichWithSeqAndRef()
                                          └─ callback fullContent
```

- `MultiReportCaseRequest`：HTTP 输入协议。
- `MultiReportCaseInput`：Agent 内部使用的规整 case。
- `MultiReportToolContext`：从 ToolMessage 解析出的工具事实表。
- `CompareReport`：`multiReportSchema.parse()` 推导出的 9 模块报告类型。
- `seq/ref`：由程序而不是模型生成。`seq` 决定前端列顺序，`ref='r{seq}'` 是前端引用标记。

## 6. 代码与 Prompt 的职责边界

代码真正强制执行的内容包括：参数合法性、MCP 生命周期、Schema 清洗、重试次数、数值回填、回调互斥和 `seq/ref` 注入。

Prompt 主要控制模型工作流：阶段 A 并行拉数、阶段 B 异常识别、阶段 B′ 知识库、阶段 C 下钻、阶段 D 输出；还包括目标 8 分钟、最长 15 分钟等预算。

复习或面试时必须把两者区分开。把 prompt 写成“系统强制并行”或“代码硬超时 15 分钟”都是不准确的。