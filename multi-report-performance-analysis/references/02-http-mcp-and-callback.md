# 02. HTTP、MCP 与回调交付

## 1. `POST /analyzeComparison` 的输入校验

对应文件：`src/api/modules/multi-report.ts`。

### 请求体

```ts
interface MultiReportRequestBody {
  taskId?: string;
  uid?: number | string;
  compareType?: number;
  businessContext?: string;
  cases?: MultiReportCaseRequest[];
}

interface MultiReportCaseRequest {
  seq: number;
  caseId: number;
  labelIds?: number[];
  projectName?: string;
  pkgName?: string;
  appVersion?: string;
  platform?: string;
  deviceModel?: string;
  osVersion?: string;
  caseName?: string;
  reportTime?: string | number;
}
```

### 校验逻辑

- `taskId` 必须为非空字符串。
- uid 优先读取 `X-Platform-User-Id`；header 不存在才使用 body.uid。
- compareType 必须是 1~7 且能在 `COMPARISON_TYPE_MAP` 中找到映射。
- cases 必须为数组，长度 2~5。
- 每个 seq 必须是 1~5 的整数；caseId 必须是正整数。
- cases 必须按 seq 严格升序，不允许重复。
- labelIds 只保留整数。`0` 表示所有 label 的业务语义在代码注释中说明“本期暂不消费”。
- businessContext 会去除首尾空白；空白字符串视为未提供。

校验失败仍返回 HTTP 200，但 body 为 `{ret:1,msg}`。这是对接协议选择，不是常规 REST 风格。

## 2. Writer 装配

路由总是创建 `LogStreamWriter`。如配置项中存在 `internal.analysisCallback`，再创建 `AnalysisCallbackWriter`；如果 analysisCallback 缺失但 callback 存在，会回退使用普通 callback 地址并打印 warn。若两个地址都没有，Agent 仍可运行，但只记录日志，不向业务系统交付结果。

MultiReportAnalysisAgent 构造函数接收 `StreamWriter[]` 而非单一 writer，因此写入能力可以同时广播到日志、回调、控制台等不同通道。

## 3. MCP 客户端生命周期

对应函数：`MultiReportAnalysisAgent.createMCPClient()`。

```ts
new MultiServerMCPClient({
  'performance-mcp-tools': {
    url: config.internal.mcp,
    headers,
    reconnect: { enabled: true, maxAttempts: 3, delayMs: 1000 }
  }
})
```

认证选择：

- 有 uid：请求头 `X-Platform-User-Id`。
- 无 uid、有 token：请求头 `X-MCP-Token`。

MultiReportAnalysisAgent 在 `compare()` 中创建 MCP 客户端，在 finally 中执行 `mcp.close().catch(() => {})`。这保证任务异常、Schema 失败或回调失败后，MCP 连接仍会释放。

`buildAgent()` 内运行 `mcp.getTools()`，工具定义由 MCP 服务动态返回，不在 MultiReportAnalysisAgent 内静态写死。Agent 因此可以跟随 MCP 服务新增工具，但也意味着工具 Schema 的兼容性必须处理。

## 4. MCP 工具 Schema 清洗

MCP 工具 Schema 可能包含 `$schema`、`format`、`anyOf` nullable union、数组上下界等字段。不同模型的工具调用 API 对这些字段支持不一致：

- Anthropic/代理可能拒绝过多 union 类型；
- Gemini 等 OpenAI 兼容后端可能拒绝 `$schema` 或将 Zod 对象错误序列化。

MultiReportAnalysisAgent 获取工具后调用 `sanitizeToolsSchema(tools)` 做原地清洗。LLMFactory 对 ChatOpenAI 还在 `bindTools()` 入口和内部 tool conversion 出口做二次规范化，以避免跨依赖版本导致 Zod v4 检测失效。

这部分的价值是让“Agent 能调用工具”成为跨模型能力，而不是只在某个 provider 上刚好可用。

## 5. AnalysisCallbackWriter 的状态机

对应文件：`src/stream/analysis-callback-writer.ts`。

Writer 内部核心状态：

```text
pending result ──onResult──> cached result
cached result ──onComplete──> done callback + settled=true
any state      ──onError────> failed callback + settled=true
settled        ──any event──> ignore
```

### 成功回调

`onResult(data)` 只保存结果。`onComplete(tokenUsage)` 负责发送：

```json
{
  "taskId": "...",
  "status": "done",
  "title": "...",
  "summaryConclusion": "...",
  "fullContent": { "...": "完整 CompareReport" },
  "inputTokens": 0,
  "outputTokens": 0
}
```

### 失败回调

`onError(error)` 发送：

```json
{
  "taskId": "...",
  "status": "failed",
  "errorMsg": "..."
}
```

`settled` 防止 done 和 failed 双发。title 被截断为 255 字符，summary 为 512，errorMsg 为 512，避免回调体被异常长文本撑大。

### 回调请求上下文

回调请求会透传：

- `X-Request-Id`
- `X-Session-Id`
- `X-Platform-User-Id`
- `Content-Type: application/json; charset=UTF-8`

这样业务方可通过相同 requestId 关联入口请求日志和异步回调日志。

## 6. 面试可讲的设计点

- 快速受理：HTTP 接口不阻塞等待 LLM，后台执行，防止网关超时。
- Writer 抽象：Agent 不关心结果写到哪里，回调、日志、控制台可组合。
- 单终态回调：通过 `settled` 避免重复交付，适合结构化任务而不是聊天流。
- MCP 生命周期：创建、动态取工具、finally 释放连接，避免后台任务泄漏。
- 协议兼容：先清洗工具 Schema，再进入模型工具调用链。