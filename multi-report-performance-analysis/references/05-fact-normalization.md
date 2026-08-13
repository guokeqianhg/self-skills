# 05. ToolMessage 事实回填、平台口径与异常重建

## 1. 为什么要做本地事实回填

模型在最终输出里需要重复写入大量数值：平均 FPS、P95/P99、Jank、内存、CPU/GPU、温度、功耗等。即便模型已经调用了工具，也可能出现：

- 把 r1 的数字写到 r2；
- 将原始数字做不受控四舍五入；
- 错把不同指标或不同统计口径当成同一个；
- 工具没有返回时补一个看起来合理的数；
- 派生指标算错。

MultiReportAnalysisAgent 的解决方式是让模型负责分析和叙述，让程序负责可验证的数值事实。模型输出 JSON 后，程序回放本次 Agent messages 内的 ToolMessage，从 MCP 原始返回中重建事实表，再覆写关键数值字段。

## 2. `MultiReportToolContext`

核心内部结构：

```ts
type MultiReportToolContext = {
  reportDataByCase: Map<number, ParsedReportData>;
  dropContextByCase: Map<number, ParsedDropContext>;
  dropContextRule: {
    dropThresholdPercent: number;
    minDropDurationMs: number;
  } | null;
};
```

`extractToolContext(messages)` 遍历每条 ToolMessage：

- `metrics_get_report_data`：解析 metadata、statistics、statistics_extra、trends，放入 `reportDataByCase`。
- `metrics_get_drop_context`：解析 params、drop_events、related_metrics、correlations。
- `metrics_analyze_root_cause`：兼容读取其中的下降事件和相关上下文。

工具返回的统计行有基本一致性验证；min/avg/max 不合理的行不进入事实表，避免把无效原始数据回填给最终报告。

## 3. `normalizeCoreMetricsFromToolData()`

该函数按每份报告的 reportId 从 `reportDataByCase` 获取工具事实，并覆盖 modules 2~6。

### 模块 2：FPS

从工具数据读取：

- `fps.avg` -> `avgFps`
- `one_percent_low.avg` -> `low1pct`
- `frame_time.p95` -> `p95FrameTimeMs`
- `frame_time.p99` -> `p99FrameTimeMs`
- `frame_time.var` -> `frameTimeVariance`

达标率本地计算：

```text
achieveRatePct = avgFps / targetFps * 100
```

没有 report data 时，相关字段置 null。

### 模块 3：流畅度

读取或计算：

- `small_jank.per_10min`
- `big_jank.per_10min`
- 总 Jank：优先工具直接值；没有时 small + big
- `stutter.avg`
- `fps.drop_per_hour`
- 明确 freeze count

### 模块 4：内存

根据平台候选集挑到一个真实存在的内存指标：

- avg -> `avgMB`
- max -> `peakMB`
- trend slopePerMin -> `growthRateMBPerMin`
- endValue - startValue -> `deltaMB`

### 模块 5：CPU/GPU

根据平台候选集获取 CPU/GPU 指标，读取 avg 与 max，覆写 `cpuAvgPct`、`cpuPeakPct`、`gpuAvgPct`、`gpuPeakPct`。

### 模块 6：热功耗

按优先级找温度和功耗指标，回填 avg/max。`throttleDetected`、`batteryDrainPctPerHour` 没有可靠统一工具字段，当前直接设 null。

## 4. 平台口径路由

入口函数：`resolvePlatformMetricSet(source)`。

它根据工具数据 metadata 检测平台，而不是只相信调用方输入。不同平台候选如下：

| 平台 | 内存候选 | CPU 候选 | GPU 候选 |
|---|---|---|---|
| Android | `memory_pss`、`memory_real`、`memory_pss_swap` | `cpu_total`、`cpu_normalized_total` | `gpu_usage` |
| iOS | `memory_xcode` 等 | `cpu_total` | `gpu_device` |
| Windows | `memory_working_set` | `cpu_app`、`cpu_system` | `gpu0_sys_3D(0)` |
| macOS | `memory_working_set` | `cpu_total` | `gpu_render` |

重要点：不同平台“内存”或“CPU”名称相似，但统计含义可能不同。程序优先使用该平台可确认的字段；没有共同可靠口径时应在报告说明限制，不能硬比较。

## 5. 派生指标校验

### Low 0.1%

`normalizeLow01Pct()` 只接受同源工具指标，并校验：

```text
0 < low01pct <= low1pct <= avgFps
```

不满足时置 null。它防止模型或者异常数据源将 Low 0.1% 写得比 Low 1% 还高，或高于平均 FPS。

### 内存占用率

`normalizeMemoryOccupancy()` 从可信 metadata 读取设备总内存：

```text
occupancyPct = peakMB / totalDeviceMB * 100
```

要求 totalDeviceMB > 0，且不小于 peakMB；否则 totalDeviceMB 和 occupancyPct 均置 null。

## 6. 异常规则与异常项

### 异常规则

`normalizeAnomalyRule()` 会读取实际调用 `metrics_get_drop_context` 或 `metrics_analyze_root_cause` 时使用的 `drop_threshold_percent` 和 `min_drop_duration_ms`，写入 module7 rule。

如果没有工具证据，不强行写默认规则，因为原异常项可能来自不同分析路径。默认 15% / 1000ms 仅在有明确的规则参数解析时作为兜底。

### 异常项重建

`normalizeAnomalyItems()` 的原则是：有证据才校正，没有证据保留模型原结果。

- 对有 dropContext 的 reportId：从 dropEvents 筛选有效区间，按跌幅降序、持续时间降序、起点升序排序，重建异常项。
- 对没有 dropContext 的 reportId：保留模型输出中属于该 reportId 的异常项。
- 每 case 最多 3 项，全局最多 10 项。

历史 bug：早期实现从空数组重建，最后无条件覆盖 module7.items。当模型走 `analyze_root_cause` 而非 drop context 时，可能将原异常项全部清空。当前代码已明确修复为“有证据才覆盖”。

### 异常原因

`buildAnomalyItem()` 基于有限规则判断可能原因，使用磁盘、CPU、GPU、可用内存和帧时间等信号，无法判断时输出“关键指标异常波动，需进一步排查”。它不是通用因果推理器，复杂根因仍应由 module8 的 LLM 归因完成。

## 7. 数值回填的边界

- 强校正主要覆盖 modules 2~6 的核心数值。
- modules 1~7 的叙述文字、summary、key findings 不会全部由模板重写。
- 事实回填后可能让 module8/module9 的文字和数值矛盾，因此还需要二次复审，见 `06-conclusion-revision-and-resilience.md`。
- 当前工具重建异常项的 Temperature 展示单位需要回归测试；`formatMetricValue()` 的格式化策略可能对 Temperature 使用 `%`，不应在缺少测试时过度承诺。

## 8. 一份报告从工具结果到最终数值的实际过程

以 r2 的 FPS 与内存为例，模型在工具调用后可能先输出：平均 FPS 为 76、P95 帧时间为 31ms、内存峰值为 1800MB。最终交付前，程序不会直接采用这些值。

`extractToolContext()` 先扫描所有 ToolMessage，找到 `metrics_get_report_data(case_id=r2)` 的结果，再解析其中的 `statistics`、`statistics_extra` 和 `trends`。随后 `normalizeCoreMetricsFromToolData()` 读取 `fps.avg`、`frame_time.p95` 等字段，覆盖模型的 FPS 和帧时间；内存部分会按平台找到如 `memory_pss` 或 `memory_working_set` 的统计值，覆盖平均值和峰值。如果工具没有返回某字段，该字段会被设为 null。

之后 `normalizeMemoryOccupancy()` 再确认 metadata 中的设备总内存是否有效：总内存必须大于 0，且不能小于应用峰值内存；满足条件才计算 `peakMB / totalDeviceMB * 100`。这意味着最终报告中的“内存占用率”不是模型估计，而是由两个工具事实字段在本地计算得到。

这个过程的关键不是“程序把模型答案改得更好看”，而是明确区分数据来源：模型输出负责解释数据，工具结果负责确定数值。