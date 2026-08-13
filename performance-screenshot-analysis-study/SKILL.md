---
name: performance-screenshot-analysis-study
description: This skill should be used when explaining, reviewing, maintaining, extending, or preparing interviews for a privacy-sanitized performance-data and screenshot joint-analysis project. It provides a linear, interview-oriented learning path covering the project value, full main workflow, deterministic code versus LLM/VLM responsibilities, design choices, implementation details, and follow-up questions.
---

# 性能数据与截图联合分析：面试复习教材

将本技能当作一套按顺序阅读的项目复习教材，而不是源码审计档案。目标是让使用者从“知道代码做了什么”，走到“能自然讲清项目、解释设计、回答追问、讨论改进方案”。

## 固定阅读顺序

不要跳读。按以下顺序学习，后文默认已掌握前文概念：

1. 阅读 `references/00-学习路线与总览.md`：先建立项目的用途、学习目标、术语和阅读路线。
2. 阅读 `references/01-三分钟讲清项目.md`：练习一句话、30 秒、3 分钟讲项目的价值与边界。
3. 阅读 `references/02-主线流程从输入到报告.md`：沿一条真实主线理解输入、确定性分析、VLM、LLM、输出如何连续衔接。
4. 阅读 `references/03-关键设计与代码实现.md`：理解每个关键设计为什么存在、由哪些函数完成、硬逻辑与模型判断如何分工。
5. 阅读 `references/04-扩展能力与可靠性.md`：补充视觉扫描、云端 MCP、报告、缓存、并发、失败降级和视频工具。
6. 阅读 `references/05-面试表达与追问训练.md`：将前面内容转化成自然表达，并按问题练习展开。
7. 被追问函数、数据对象或模型边界时，查阅 `references/06-函数与数据对象追问索引.md`。

## 脱敏命名约定

为保证匿名化后仍能讨论实现细节，统一采用以下规则：

- **产品与服务**：统一称为“性能采集平台”；真实平台名、域名、报告编号和凭据均不保留。
- **模块与源码布局**：主入口使用 `pipeline.py`，业务模块统一位于 `core/`；云端连接为 `PlatformMcpClient`，云端数据对象为 `RemoteReportData`。
- **核心领域模型**：保留 `PerformanceDataBundle`、`PerformanceSignal`、`PerformanceEvent`、`SceneResult`、`EventAnalysis`、`VerificationPlan`、`VisualFinding` 等通用领域术语。
- **外部协议占位符**：统一使用 `reportId`、`PLATFORM_MCP_TOKEN`、`https://mcp.example.invalid/v1` 与 `<REDACTED_TOKEN>`；它们只说明接口角色，不代表真实协议。
- **指标与技术通词**：保留 `FPS`、`FrameTime`、`SevereJank`、MCP、LLM、VLM、JSON-RPC 等通用性能分析概念；`表格化时序文本载荷` 表示匿名化后的表格时序文本载荷。

## 回答或讲解规则

1. 始终先给结论，再按“输入 → 处理 → 输出 → 价值/边界”解释；不要先报函数名。
2. 出现英文类名、字段名或函数名时，同时说明它保存什么、由谁调用、对结果有什么影响。
3. 明确区分两类事实：
   - **代码硬逻辑**：解析、统计阈值、筛选、校验、缓存、并发、数据写入；结果可重复。
   - **模型判断**：截图场景、根因假设、下钻方向、语义聚类、复测文字、画面判读；结果需要结构约束和人工复核。
4. 不把模型的推测说成根因事实；没有采集到的数据必须说明“未采集，不能证实”。
5. 使用直接、通俗的技术语言，不使用比喻，不让读者自行拼接文档。

## 快速定位索引

- 一句话价值、项目边界：`01`。
- 整个主流程与数据如何变化：`02`。
- FrameTime、FPS、SevereJank、目标帧率、事件聚合：`03` 第 1～3 节。
- VLM/LLM 分别做什么、两阶段下钻：`02` 第 5～6 节、`03` 第 4 节。
- 缓存、Top 25、模型输出校验、降级：`03` 第 5～7 节。
- 视觉识别、批次、并发、整段核查：`04` 第 1 节。
- 云端 MCP、表格化时序文本载荷、内存截图：`04` 第 2 节。
- HTML、Token、全量存档、图片视频：`04` 第 3～5 节。
- 面试回答、追问、局限和可演进设计：`05`。
- `PerformanceDataBundle`、`detect_performance_signals`、`EventAnalyzer`、`VerificationPlanner` 等对象/函数的输入、输出和章节位置：`06`。
