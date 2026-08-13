# 06. 结论复审、重试与故障恢复

## 1. 数值校正后的语义问题

MultiReportAnalysisAgent 先让模型写完整报告，再用工具事实回填 modules 2~6 的数值。这样会产生一个自然问题：module8 根因和 module9 优化建议可能是基于回填前的错误草稿写的。

例如模型写“r2 平均 FPS 75，明显不达标，因此建议优先降低 GPU 压力”；事实回填后 r2 实际平均 FPS 是 88，达标率正常，那么原根因和建议可能不再成立。

代码没有简单地每次都重写根因建议，因为成本高、也可能引入新的变化。它使用“事实 diff + 轻量判定 + 必要时定向重写”的两级流程。

## 2. 影响事实的构建

函数：`buildModule8And9ImpactFacts(report)`。

它从报告中抽取会影响根因/建议的高价值字段，拍平成字典。包括：

- `correlation.level`
- module1 的 comparisonType、best/worst/watch 报告
- 每份报告的 target FPS 和来源
- module2 FPS、Low%、帧时间、达标率、评级
- module3 Jank、Stutter、Freeze、FPS drop、评级
- module4 内存均值、峰值、增长率、趋势、评级
- module5 CPU/GPU、瓶颈、评级
- module6 温度、功耗、降频、耗电、评级
- module7 异常规则、异常数量和异常项签名

数值会转换为有限数并保留一位小数，空字符串转 null。这样可以避免 `88` 和 `88.0000001` 这类浮点噪声触发无意义重写。

## 3. 事实 diff 与阈值

函数：`buildModule8And9ReviewContext(original, normalized)`。

- 如果 module8 hypotheses 和 module9 items 都为空，不复审。
- 构建校正前与校正后的事实字典。
- 用 `diffModule8And9ImpactFacts()` 比较每个 key。
- 无变化：直接使用 normalized 报告。
- 有变化：构建 `changedFacts` 和 `materialChangeCount`。

阈值是 `MAX_MODULE8_AND9_GATE_CHANGE_LIMIT = 24`。高影响变化超过 24 项时，认为草稿与事实差距较大，跳过轻量判定，直接进入完整修订。

## 4. 轻量判定：KEEP 或 REVISE

函数：`requestModule8And9Decision()`。

它使用 `LLMFactory.resolveAuxiliaryConfig()` 获取辅助模型，只做小任务：判断现有 module8、module9 是否需要实质修改。输出 Schema 约束为：

```json
{
  "module8Decision": "KEEP | REVISE",
  "module8Reason": "...",
  "module9Decision": "KEEP | REVISE",
  "module9Reason": "..."
}
```

输入包括：

- 变化后的事实清单，最多 24 条；
- 当前 module8_rootCause；
- 当前 module9_optimization。

Prompt 明确约束：字段变得更保守、数字被清空，但结论方向仍一致时，应返回 KEEP；不要因为轻微数值调整就重写。

若轻量模型输出格式不合格，会额外进行 1 次只针对 JSON 格式的重试。轻量判定失败时，为安全起见回退完整修订。

## 5. 完整定向修订

函数：`requestModule8And9Revision()`。

只有在以下情况调用主模型：

- 高影响变化超过 24；
- 轻量模型返回 module8 或 module9 为 REVISE；
- 轻量判定请求或解析失败。

修订 prompt 的关键限制：

- 只修改 module8_rootCause 和 module9_optimization；
- 不能改 modules 1~7；
- 证据必须来自校正后的事实；
- module8 根因与 module9 建议必须一致；
- 输出严格 JSON。

修订结果会分别经过 `rootCauseModuleSchema` 和 `optimizationModuleSchema` 校验。某模块缺失或不合法，不采用该模块的修订，保留原值。这样避免“为了修一个模块，把原本正常的另一模块也覆盖坏”。

## 6. `applyModule8And9Revision()`

若 revision 接受：

- `module8Changed=true` 时覆盖 module8。
- `module9Changed=true` 时覆盖 module9。
- 将 adjustmentNotes 打日志，便于追踪“哪部分是回填后的定向修订”。

若 revision 为 null，返回原 normalized 报告，不让复审失败阻断整个报告交付。

## 7. 两套重试不能混淆

### 结构化输出重试

发生在最终 AI 输出不合法时：没有 JSON、JSON 不符合 Zod、没有拉工具数据。最多 2 次。它的目标是让模型产出可用 JSON。

### 工具重试

由 `toolRetryMiddleware({maxRetries:3})` 处理工具调用失败。它的目标是处理 MCP 工具执行失败。

### 上游瞬态错误重试

发生在 `agent.stream()` 请求模型或网关时：空响应、5xx、连接重置、超时。最多 2 次，指数退避加抖动。它的目标是处理网络/网关临时失败。

这三套机制分别处理“格式错误”“工具错误”“模型链路错误”。面试中应明确区分，不能笼统说“失败就重试”。

## 8. 失败链路日志

`compare()` 的 catch 会沿 `error.cause` 最多展开 8 层，记录 name、message、code、pregelTaskId 等信息。原因是 LangGraph、MCP 适配层和模型 SDK 会包装错误，默认日志通常只展示最外层，无法定位空响应、网络错误或底层工具错误。

## 9. 当前韧性设计的边界

- 回调失败本身是否重试取决于 callback service 的实现，本模块重点记录结果并调用 writer，不应夸大为端到端 Exactly Once。
- 结构化中间件最后可能返回 repair 后 raw 降级对象，不能说所有回调结果都绝对通过顶层 Schema。
- prompt 中 15 分钟预算不是独立 wall-clock watchdog；极端模型响应慢时仍需网关、服务层或未来任务调度层补真正超时控制。

## 10. 一次失败时实际会走哪条路径

需要先区分失败发生的位置，代码不会用同一种“重试”处理所有问题。

如果模型已经完成工具调用，但最终返回的是 Markdown 说明或半截 JSON，属于结构化输出问题。`structuredOutputRetryMiddleware` 会保留已有 ToolMessage，不重新拉报告数据，只补一条明确的纠错消息，让模型按现有数据重新组织 JSON，最多执行 2 次。

如果 MCP 工具本身报错，例如查询报告数据失败，属于工具执行问题，由 `toolRetryMiddleware` 在当前工具调用层最多重试 3 次。它不会把整个 Agent 会话从头开始执行。

如果 `agent.stream()` 直接出现空 SSE、5xx、连接超时或连接重置，则属于模型网关或网络瞬态问题。`streamOnce()` 会销毁本轮 AbortController、内容累积和工具事件状态，等待指数退避时间后重新发起整轮 stream，最多 2 次。这里重新执行是必要的，因为原来的模型流没有可靠完成。

如果以上重试仍然失败，`compare()` 会展开 cause 链记录根因，再调用 writer 的 `onError()`。AnalysisCallbackWriter 发送一次 failed 回调，并通过 `settled` 阻止后续重复回调。