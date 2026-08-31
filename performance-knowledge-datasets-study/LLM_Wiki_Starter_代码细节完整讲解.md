先给结论：

> `perfdog_llm_wiki_starter` 不是一个自主规划、调用工具、循环反思的 Agent。  
> 它是一个**LLM 参与的知识 ETL / 知识生产流水线**：用确定性脚本管理来源、抓取、质量校验、状态和产物；让 LLM 只负责非结构化文本抽取与少量边界语义判断。

它解决的问题是：把分散的网页、官方文档、案例和视频资料，转换成**可追溯、可审核、可检索**的性能知识卡与 Wiki，供 Dual RAG 和下游诊断 Agent 使用。

---

# 1. 项目整体结构：它产出什么？

完整主线：

```text
专家资料链接清单 / 单条 URL
        ↓
01 来源登记、自动分类
        ↓
MySQL sources 表
        ↓
02 网页抓取 / 视频字幕与 ASR
        ↓
raw、meta、markdown 中间资产
        ↓
03 规则质量校验
        ↓
04 LLM 抽取知识卡草稿
        ↓
cards_draft.jsonl
        ↓
05 规则校验 + 受限 LLM 语义标签
        ↓
cards_validated.jsonl
        ↓
06 Markdown Wiki 渲染
        ↓
wiki/
        ↓
Dual RAG 构建 rag_docs.jsonl、Qdrant 向量库
        ↓
Gateway / Agent 检索、归因
```

它的职责止于：

> **把原始资料加工成高质量知识资产。**

它不直接负责：

```text
用户性能问题的最终根因裁决
实时性能数据采集
在线向量检索
自主规划和多轮工具调用
```

---

# 2. 数据到底存在哪里？哪些是“真源”？

这是面试容易讲错的地方。

| 数据 | 存放位置 | 作用 | 是否真源 |
|---|---|---|---|
| 来源与人工编目 | MySQL `sources` 表 | URL、标题、优先级、抓取状态、审核状态 | 是 |
| 原始网页/字幕 | `data/raw/` | 保存原始抓取结果，方便追溯 | 审计产物 |
| 抓取元信息 | `data/meta/` | HTTP、标题、抓取时间、错误等 | 审计产物 |
| 清洗正文 | `data/markdown/` | 后续 LLM 抽卡的输入 | 关键中间产物 |
| 草稿知识卡 | `data/cards/cards_draft.jsonl` | 04 阶段 LLM 的结构化抽取结果 | 草稿卡真源 |
| 正式知识卡 | `data/cards/cards_validated.jsonl` | 05 校验通过、供 Wiki/RAG 消费 | 正式卡文件真源 |
| 校验/缓存/用量记录 | `data/manifests/` | 抓取、抽取、校验审计与重跑辅助 | 辅助状态 |
| Wiki 页面 | `wiki/` | 给人读的 Markdown 输出 | 派生产物，可重建 |
| `cards_index` | MySQL | 给 Web 管理端做分页、筛选、普通文本搜索 | 派生索引，可重建 |

特别注意：

```text
MySQL sources
≠ cards_validated.jsonl
≠ Wiki
≠ Qdrant
```

它们分别管理不同层次：

```text
sources
→ “资料从哪里来、是否启用、抓取是否成功”

validated cards
→ “从资料中沉淀出了哪些正式知识”

Wiki
→ “如何给人阅读这些知识”

Qdrant
→ “如何做向量语义检索”
```

不能说：

> “所有知识都在 MySQL。”

更准确的说法是：

> 来源和运行态编目在 MySQL；知识卡以 JSONL 为主文件资产；Wiki、MySQL `cards_index` 和 Qdrant 都是由这些资产构建出的派生产物。

---

# 3. 第一阶段：来源登记与自动分类

相关脚本：

```text
01_build_sources.py
01b_add_single_source.py
registry_db.py
```

---

## 3.1 `01_build_sources.py` 做什么？

它读取根目录的：

```text
专家知识库资料链接清单.md
```

从 Markdown 中解析 URL、标题层级、案例信息和上下文，生成候选来源，并写入 MySQL `sources` 表。

```text
82:95:e:/perfdog_agent/perfdog_knowledge_datasets/perfdog_llm_wiki_starter/scripts/01_build_sources.py
// ... normalize_source_url 对 URL 做规范化；make_source_id 基于规范 URL 生成稳定来源 ID ...
```

### URL 为什么要先归一化？

同一篇资料可能以不同形式出现：

```text
https://example.com/a?b=1&c=2
https://EXAMPLE.com/a?c=2&b=1#section
```

如果直接把原 URL 当主键，可能重复收录。

因此代码会：

```text
host 转小写
→ query 参数排序
→ 去掉 fragment，例如 #section
→ 清理末尾 /
→ 得到 source_key
```

再基于 `source_key` 生成稳定的：

```text
source_id
```

例如：

```text
SRC-XXXXXXXXXX
```

所以：

- `source_key`：规范化 URL，用于去重；
- `source_id`：系统内部稳定来源 ID；
- 同一篇资料即使在清单中出现多次，也应尽量对应同一个来源。

---

## 3.2 自动推断了哪些信息？

这一阶段主要靠**关键词和规则**，不是 LLM。

它会基于 URL、标题、章节路径、案例上下文推断：

```text
platforms
engines
graphics_apis
topics
root_cause_categories
usage_scopes
fetch.strategy
mapped_metrics
```

例如：

```text
URL 或标题含 Unity
→ engines 可能推断为 Unity

标题含 Android
→ platforms 可能推断为 Android

Bilibili /video/ URL
→ fetch.strategy = video_transcript_first

普通网页
→ fetch.strategy = firecrawl
```

这些都是“初步编目提示”，不能当最终事实。

例如：

```text
mapped_metrics
```

代表“这篇来源可能与哪些 PerfDog 指标相关”，它用于帮助后续抽卡和检索，但不代表：

> 原文已经证明这些指标一定存在因果关系。

---

## 3.3 单条 URL 如何接入？

`01b_add_single_source.py` 用于新增一条来源。

它适合 Gateway 的：

```text
POST /api/wiki-starter/ingest-source
```

或 Web 管理端新增资料时使用。

输入大致是：

```text
url
title
section_hint
priority
include_in_wiki
title_override
mapped_metrics_override
notes
updated_by
```

其中：

- `title`：标题提示；
- `section_hint`：人为提供的上下文，例如“Unity 渲染”；
- `priority`：来源优先级，通常是 `P0` 或 `P1`；
- `include_in_wiki`：是否进入 Wiki 主链路；
- `mapped_metrics_override`：人工修正自动推断的指标；
- `notes`：人工收录理由；
- `updated_by`：记录操作者。

这里要区分：

```text
来源 priority
≠ 知识卡 card_priority
```

前者表示：

> 这条资料的处理或收录优先级。

后者表示：

> 从资料抽出的某张知识卡在知识库中的重要程度。

---

# 4. MySQL `sources`：来源治理的核心

相关模块：

```text
registry_db.py
workspace_store.py
```

`registry_db.py` 管理来源表和人工编辑逻辑。

来源记录通常包含几个层面：

```text
基础信息
→ source_id、url、source_key、title

自动推断
→ platform、engine、topic、metrics、fetch.strategy

人工治理
→ enabled、priority、include_in_wiki、title_override、notes

运行状态
→ fetch.status、fetch.last_error、review_status
```

可以这样理解：

```text
自动推断
= 系统初步猜测

人工字段
= 人类最终治理意见

运行状态
= 当前流水线跑到了哪里、是否失败
```

---

## 4.1 为什么来源要放 MySQL，而不是只用 JSONL？

因为来源不是一次性静态文件，而是有持续变化的运行态：

```text
是否抓取成功
最后一次错误是什么
是否需要人工复核
人工是否修改标题
是否启用
谁在什么时候修改过
```

这些更适合关系型数据库管理。

同时 Web 管理端会多人编辑来源，因此来源更新有乐观锁保护。

乐观锁可以理解为：

```text
用户 A 打开来源，版本号为 5
用户 B 同时修改并保存，版本变为 6
用户 A 再保存时发现版本不一致
→ 拒绝覆盖，提醒重新加载
```

这样防止“后保存的人把前一个人的修改静默覆盖掉”。

---

# 5. 第二阶段：网页抓取与视频文本获取

相关脚本：

```text
02_scrape_sources.py
02b_fetch_video_transcripts.py
02c_retry_failed_fetches.py
```

---

## 5.1 普通网页：Firecrawl 抓取

普通网页一般走：

```text
02_scrape_sources.py
```

流程：

```text
从 MySQL 读取 enabled 的 sources
→ 找 fetch.strategy=firecrawl 的来源
→ 调 Firecrawl API
→ 得到 Markdown / HTML / 元数据
→ 清洗正文
→ 写 raw、meta、markdown
→ 回写 MySQL 抓取状态
```

产物：

```text
data/raw/
→ 原始抓取响应

data/meta/
→ 标题、抓取时间、状态码等元数据

data/markdown/
→ 清洗后的 Markdown，供 04 LLM 抽卡使用
```

---

## 5.2 为什么要保留 raw、meta、markdown 三份？

因为它们用途不同：

```text
raw
→ 原始证据，方便复查“抓取服务到底返回了什么”

meta
→ 排查抓取时间、标题、错误、HTTP 等问题

markdown
→ 清洗后的可读正文，是 LLM 的正式输入
```

不能只保留 Markdown。

否则后续发现抽卡质量差时，无法判断问题来自：

```text
原网页本身
抓取服务
清洗规则
还是 LLM
```

---

## 5.3 抓取失败怎么办？

网页抓取会针对这些情况重试：

```text
网络异常
HTTP 429
5xx
正文过短
```

典型策略是有限次数重试加等待。

这里的重试属于：

> **抓取脚本针对外部服务失败的重试。**

它与 Gateway Job 的 lease 恢复不同：

```text
抓取重试
→ 外部网页/API 失败后，在脚本内部重试

Job lease 恢复
→ Worker 崩溃后，整个后台任务可重新入队
```

两者不要混为一谈。

---

## 5.4 视频：字幕优先，ASR 兜底

视频来源，比如 Bilibili，不走普通网页抓取，而走：

```text
02b_fetch_video_transcripts.py
```

流程：

```text
优先获取官方字幕
→ 走备用字幕/API 路径
→ 仍失败时下载音频
→ ffmpeg 转换音频
→ 必要时切分音频
→ Whisper 或 OpenAI-compatible ASR 转录
→ 生成 transcript Markdown
```

这里：

- **字幕优先**：成本更低、速度更快、内容通常更准确；
- **ASR**：Automatic Speech Recognition，自动语音识别；
- `ffmpeg`：音视频转码工具；
- `asr_fallback`：决定字幕失败时是否允许 ASR 兜底。

常见 `asr_fallback`：

```text
off
→ 不使用 ASR

auto
→ 字幕失败时尽量用 ASR

required
→ 必须完成 ASR，否则视为失败
```

要注意：

> 这不是 VLM，也不是视频画面理解。  
> 当前实现的核心是把视频转换为文本资料，后续仍按文本处理。

---

# 6. 第三阶段：正文质量校验

相关脚本：

```text
03_validate_scrapes.py
```

这一阶段**不调用 LLM**。

它做的是规则质检，目的是：

> 在花 LLM 成本前，先挡住错误页、空内容、重复内容和低质量转录。

主要检查：

```text
Markdown 是否存在
正文长度是否足够
是否像 404 页面
是否像访问拒绝 / 验证码 / 登录页
是否像购买页或营销页
是否缺少正常标题结构
正文是否与其他来源重复
视频字幕是否存在
有效语音内容是否过少
视频标题与字幕是否明显不匹配
```

结果会回写来源状态。

最关键的状态可以理解为：

```text
scraped
→ 已抓到内容，但尚未通过正文质检

validated
→ 正文校验通过，可以进入 LLM 抽卡

needs_manual_review
→ 不一定完全错误，但系统不够确定，需要人工判断
```

这一步的重要设计是：

```text
低质量输入
→ 不进入 LLM
→ 减少成本
→ 避免生成错误知识
```

也就是说，系统不是：

```text
抓到什么都扔给模型
```

而是：

```text
先用确定性规则过滤
→ 再让模型处理相对可靠的正文
```

---

# 7. 第四阶段：LLM 知识卡抽取

相关脚本：

```text
04_extract_cards.py
models.py
```

这是第一个真正使用 LLM 的阶段。

但它不是“让模型自由总结文章”，而是：

> 从已经通过正文校验的资料中，提取可检索、可验证、可追溯的结构化性能知识卡。

---

## 7.1 输入和输出

输入：

```text
data/markdown/<source_id>.md
+ 来源元数据
+ 平台/引擎提示
+ 可映射指标提示
+ 卡片数量预算
```

输出：

```text
data/cards/cards_draft.jsonl
```

每一行是一张 JSON 知识卡。

---

## 7.2 为什么叫“知识卡”，而不是文章摘要？

文章摘要通常是：

```text
“这篇文章主要介绍了……”
```

但下游性能诊断不需要泛泛总结，它需要能回答：

```text
出现什么现象？
可能是什么根因？
什么信号支持？
什么信号反驳？
应该怎么验证？
可以怎么优化？
这条结论来自哪里？
哪些结论不能直接下？
```

因此知识卡会包含类似字段：

```text
card_type
title
summary
platforms
engines
metrics
observed_pattern
required_signals
supporting_signals
contradicting_signals
cannot_conclude
verification_steps
optimization_actions
risks
limitations
source_claims
```

其中：

- `observed_pattern`：观察到的现象；
- `required_signals`：要支持该判断，最好必须看到的信号；
- `supporting_signals`：额外支持该判断的信号；
- `contradicting_signals`：与该判断冲突的信号；
- `cannot_conclude`：仅靠当前资料不能直接下的结论；
- `verification_steps`：如何验证该判断；
- `optimization_actions`：可以尝试的优化；
- `source_claims`：来源证据，包括位置和短摘录。

这使得知识库不是“结论堆积”，而是更适合诊断的证据结构。

---

## 7.3 卡片类型

主要有七类：

```text
metric_definition
→ 指标定义

phenomenon_pattern
→ 现象模式

root_cause
→ 根因知识

verification_method
→ 验证方法

optimization_action
→ 优化动作

case_evidence
→ 案例证据

platform_difference
→ 平台差异
```

这样下游 RAG 不是只召回“可能根因”，还可以召回：

```text
现象解释
验证方法
优化建议
风险与局限
平台差异
```

---

## 7.4 LLM 调用不是只靠一句 Prompt

一次抽卡请求由三部分组成：

```text
System Prompt
+ Output Schema Prompt
+ User Prompt
```

### System Prompt：限制模型角色和边界

模型被定义为：

```text
性能分析知识工程师
```

而不是普通总结助手。

核心约束包括：

```text
只能提取正文明确支持的内容
没有证据就不要生成卡
不能为了完整而编造根因或优化
不能仅凭一个汇总指标断言具体根因
必须填写反证与不能下的结论
不能编造函数名、调用栈、阈值、收益
泛泛的 CPU/GPU/帧率不能硬映射为精确 PerfDog 指标
纯接入教程或 UI 操作不应包装成性能根因或优化知识
```

### Output Schema Prompt：限制输出格式

模型必须只输出：

```json
{
  "cards": []
}
```

而不能输出解释性文本或 Markdown。

### User Prompt：给当前受限上下文

每一个正文块会携带：

```text
source_id
title
url
source_type
platforms_hint
engines_hint
mapped_metrics_hint
generalization_policy
允许指标白名单
本 chunk 最多生成几张卡
当前 Markdown 正文块
```

因此模型不是看“裸文章”，而是收到明确任务：

> 这是什么来源、适用什么场景、允许用哪些指标、禁止生成什么、最多抽多少张卡；请只基于下面正文抽取知识。

---

## 7.5 为什么正文要分块？

一篇长文可能超过模型上下文预算。

因此代码会优先按 Markdown 标题分块：

```text
按章节切
→ 尽量保证一个主题不被截断
→ 单个章节仍过长时才硬切
```

这样比按固定字数粗暴切分更好，因为：

```text
现象描述
根因说明
验证步骤
优化建议
```

往往在同一个章节内，切断后更容易导致模型误解上下文。

默认每块最多抽：

```text
3 张卡
```

每个来源通常最多保留：

```text
10 张卡
```

目的是防止模型把一篇文章过度拆碎，产生大量重复、弱相关卡。

---

## 7.6 Pydantic 在这里做什么？

Prompt 只能要求模型输出 JSON，不能保证模型一定遵守。

模型可能：

```text
返回 Markdown
JSON 格式错
漏字段
把数组写成字符串
card_type 写成不存在的值
```

所以输出会再经过 Pydantic 校验。

Pydantic 可以理解为：

> Python 中的结构化数据校验器。

它检查：

```text
字段是否存在
字段类型是否正确
枚举是否合法
数组结构是否符合要求
嵌套 SourceClaim 是否合规
```

抽卡结果会经过：

```text
模型返回文本
→ 去掉 Markdown code fence
→ 提取 JSON
→ Pydantic 校验
→ 若失败，最多做一次 JSON repair
→ 再次 Pydantic 校验
```

`repair` 的含义是：

> 带着原模型输出和 Pydantic 报错信息，让模型只修正 JSON 格式或字段结构。

注意：

```text
Pydantic
= 保证结构

不等于
= 保证事实正确
```

`repair` 也不等于事实核验。

它只解决：

```text
格式不合格
```

不能证明：

```text
模型说的内容一定真实
```

---

## 7.7 Pydantic 之后还有本地硬规则

模型输出通过 Pydantic 后，不会立刻写入正式卡。

还要经过本地规则处理，例如：

```text
canonical_key 转成 snake_case
清理空字段
数组去重
平台和引擎别名统一
指标白名单收紧
无法映射的内容放到 unmapped_signals
source_claims 去重和截断
非优化类卡清空优化字段
按来源限制泛化范围
```

随后执行：

```text
生成稳定 card_id
→ 同来源内按 card_type + canonical_key 去重
→ 本地评分排序
→ 应用每 chunk、每 source 卡片上限
→ 写入 cards_draft.jsonl
```

所以真实过程是：

```text
LLM 候选
→ Pydantic 结构校验
→ 本地字段归一
→ 指标收紧
→ 去重
→ 限额
→ 草稿卡
```

不是：

```text
文章
→ LLM
→ 直接成为正式知识
```

---

# 8. 第五阶段：正式卡校验与受限语义模型

相关脚本：

```text
05_validate_cards.py
```

这一阶段的设计原则是：

> 能确定的事情交给规则；只有规则难判断的语义边界才交给 LLM。

输入：

```text
cards_draft.jsonl
```

输出：

```text
cards_validated.jsonl
+ 校验报告
+ 语义缓存
+ 语义调用记录
```

---

## 8.1 本地规则做什么？

规则会检查例如：

```text
卡片结构是否合法
card_type 是否合法
指标是否落在允许范围
平台与引擎是否过度泛化
卡片是否只是工具操作说明
是否缺少足够的指标、现象、验证锚点
是否需要人工审核
优先级如何划分
```

规则可以稳定回答的问题不需要消耗 LLM。

例如：

```text
字段不存在
→ 规则直接判失败

指标名不在白名单
→ 规则直接处理

明显只是录屏操作说明
→ 规则可直接判定内容角色偏采集方法
```

---

## 8.2 LLM 在 05 阶段做什么？

它不会重新判断：

```text
这张卡是不是事实
这张卡是不是最终有价值
这张卡是否应该进入知识库
```

它只对规则难判断的内容打四类固定标签：

```text
language_mix
claim_tone
tool_reference_role
content_role
```

含义：

```text
language_mix
→ 中文表达中是否混入不必要的英文

claim_tone
→ 结论是否过度绝对化

tool_reference_role
→ 外部工具是辅助参考，还是结论必须依赖它

content_role
→ 是通用诊断知识、采集方法，还是测量约束
```

它只能在有限枚举中选择，例如：

```text
claim_tone:
neutral
slightly_strong
overclaim
```

这比“请自由评价这张卡”更可控。

---

## 8.3 为什么要最小化 LLM 输入？

05 阶段不会把整篇原文、整张大卡全部送给模型。

不同判断只发送必要字段。

例如：

```text
检查 claim_tone
→ title、summary、cannot_conclude、limitations

检查工具依赖
→ verification_steps、工具名称、少量 source_claims

检查 content_role
→ observed_pattern、verification_steps、limitations
```

目的：

```text
减少 token 成本
减少无关上下文干扰
降低模型自由发挥空间
让任务只聚焦一个局部判断
```

---

## 8.4 缓存、批处理和降级

05 阶段为降低成本做了四层优化：

```text
明显样本先走本地规则
→ 不确定但启发式足够的样本直接短路
→ 真正边界卡才送 LLM
→ 默认每批约 12 张卡合成一次请求
```

同时存在语义缓存。

缓存 Key 大致由：

```text
Prompt 版本
+ 模型名
+ 待审 payload
```

共同决定。

如果同一张卡、同一模型、同一 Prompt 版本再次校验：

```text
命中缓存
→ 不再调用模型
```

这使得重跑流程不会无意义重复花模型成本。

`semantic_mode` 有三个常见模式：

```text
auto
→ 尽可能调用模型；不满足条件时允许走启发式处理

required
→ 必须使用模型，模型不可用则任务失败

off
→ 不使用语义模型，只跑本地规则
```

这是真实的降级边界：

> 05 阶段可按模式降级；04 抽卡阶段没有“模型失败后自动用规则生成知识卡”的替代路径。

---

# 9. 第六阶段：渲染 Markdown Wiki

相关脚本：

```text
06_render_wiki.py
```

这一阶段不再调用 LLM。

输入：

```text
cards_validated.jsonl
```

输出：

```text
wiki/
```

目录通常按知识类型组织：

```text
01_metrics
02_patterns
03_root_causes
04_verification
05_optimizations
06_cases
07_platforms
```

每个 Wiki 页面通常包含：

```text
卡片标题
卡片类型
优先级
审核状态
平台/引擎
相关指标
现象
支持与反证信号
验证步骤
优化动作
风险与局限
来源链接和证据
```

还会生成：

```text
wiki/index.md
→ 全量目录

wiki/index_core.md
→ 核心知识目录
```

---

## 9.1 为什么 Wiki 是派生产物？

Wiki 可以从：

```text
cards_validated.jsonl
```

重新生成。

因此 Wiki 更像：

```text
人类阅读视图
```

而不是独立真源。

这样做的好处：

```text
卡片删除
→ 重新渲染 Wiki

卡片审核状态变化
→ 重新渲染 Wiki

卡片字段更新
→ 重新渲染 Wiki
```

不需要人工同步维护多份内容。

---

## 9.2 什么是增量渲染？

如果只删除或修改某个来源下的卡片，没必要重建全部 Wiki。

系统会拿到：

```text
affected_source_ids
```

即“哪些来源被影响”。

随后：

```text
清理这些来源旧页面
→ 用当前剩余正式卡重渲染这些来源
```

优点：

```text
更快
减少无关页面改动
避免全量渲染成本
```

---

# 10. `cards_admin.py`：删卡与 RAG 可见性治理

除了抽取和校验，还有两个重要治理动作：

```text
delete_cards
set_rag_visibility
```

---

## 10.1 删除卡片

删除可以按：

```text
card_id
```

删除单张或多张卡，也可以按：

```text
source_id
```

删除某个来源下所有卡。

流程：

```text
修改 cards_draft.jsonl
→ 修改 cards_validated.jsonl
→ 返回 affected_source_ids
→ Gateway 触发受影响来源的 Wiki 增量渲染
```

注意：

```text
删卡
≠ 自动删 RAG 向量
```

如果不重建 RAG，旧的：

```text
rag_docs.jsonl
Qdrant 索引
```

仍可能保留被删内容。

---

## 10.2 `rag_excluded`：保留卡片但退出检索

`rag_excluded=true` 的含义是：

```text
卡片仍存在
Wiki 可以仍然展示
但 RAG 构建时跳过它
```

适用于：

```text
知识卡暂时有风险
需要人工复核
不希望污染 Agent 检索
但不想丢失审计记录
```

它把：

```text
是否保存
```

和：

```text
是否参与线上检索
```

分开。

这是很典型的知识治理设计。

---

# 11. Web 管理端：为什么还需要 Webapp？

相关目录：

```text
webapp/
```

它的作用不是做 RAG 检索主服务，而是供人工管理知识资产。

典型能力包括：

```text
查看来源
编辑人工编目字段
启用或停用来源
查看抓取状态和错误
查看草稿卡与正式卡
按来源、平台、类型、审核状态筛选卡片
```

Web 管理端主要依赖：

```text
MySQL sources
MySQL cards_index
```

其中 `cards_index` 是从 JSONL 重建的管理索引。

它不是 Qdrant，也不负责语义检索。

可以这样区分：

```text
Webapp
→ 人工治理与查看

Qdrant
→ 语义检索

Gateway
→ 统一 API、MCP 和任务编排
```

---

# 12. 它如何被 Gateway 调用？

Wiki Starter 本身是脚本型工程，不是独立 HTTP 服务。

因此 Gateway 通过白名单任务把它服务化。

例如 Gateway 收到：

```text
POST /api/wiki-starter/ingest-source
```

会创建异步 Job，Worker 在后台调用 Wiki Starter 流水线：

```text
add_source
→ scrape_sources 或 fetch_video_transcripts
→ validate_scrapes
→ extract_cards
→ validate_cards
→ render_wiki
```

对每一步，Gateway 不只看：

```text
exit_code == 0
```

还会检查业务事实：

```text
source 抓取状态是否正确
是否真的生成 draft card
是否真的生成 validated card
```

这避免了批处理脚本“整体没异常退出，但当前来源实际失败”的假成功。

---

## 12.1 直接调用和通过 Gateway 调用的区别

直接 Python 调用：

```text
run_ingest_source_pipeline()
```

默认：

```text
rebuild_rag=false
```

也就是说：

```text
来源、卡片、Wiki 更新
但 RAG 检索不一定立即更新
```

而通过 Gateway：

```text
HTTP / MCP
→ Job
→ Worker
```

成功后可能触发后续：

```text
full RAG build
→ rag_docs
→ Qdrant
→ active release
```

所以不能笼统说：

> “新增一条资料一定自动进入向量库。”

必须先讲清楚：

```text
你是直接调用 Wiki Starter pipeline
还是走 Gateway Job 编排
```

---

# 13. 可靠性设计总结

这个项目的可靠性不是靠一个地方完成的，而是分层实现。

| 风险 | 对应机制 |
|---|---|
| 同 URL 重复收录 | URL 归一化 + `source_key` + 稳定 `source_id` |
| 人工编辑互相覆盖 | MySQL 乐观锁 |
| 网页 API 临时失败 | 抓取脚本有限重试 |
| 视频没有字幕 | 字幕优先 + ASR 兜底 |
| 错误页面进入 LLM | 03 规则正文质量校验 |
| LLM 输出不合法 JSON | Pydantic + 一次 repair |
| LLM 过度生成 | Prompt 约束 + 指标白名单 + 去重 + 限额 |
| 重跑重复花模型成本 | 05 语义缓存 |
| 卡片删了但 Wiki 未更新 | 受影响来源增量重渲染 |
| 高风险卡污染 RAG | `rag_excluded` |
| 脚本 exit code 成功但业务没成功 | Gateway 业务产物核验 |
| 多个写任务同时改资产 | Gateway Job + MySQL 单写锁 |

---

# 14. 当前限制：不能夸大的能力

面试时建议主动说明这些边界。

## 14.1 它不是自主 Agent

没有：

```text
Planner
多轮工具调用
自主决策任务路径
自我反思循环
自动选择外部工具
```

它是：

> Python 控制流程的 LLM 知识 ETL。

---

## 14.2 `validated` 不代表事实被绝对证明

`cards_validated.jsonl` 的真实含义是：

> 通过了当前版本的结构规则、质量规则和有限语义标签策略。

它不代表：

```text
这张卡的全部事实都经过人工专家逐句确认
```

因此仍然保留：

```text
source_claims
review_status
warnings
rag_excluded
needs_manual_review
```

用于控制下游使用方式。

---

## 14.3 LLM 不能替代性能测量事实

知识卡只能提供：

```text
现象模式
可能根因
验证方法
优化经验
反证条件
```

它不能替代当前真实报告中的：

```text
FPS
CPU
GPU
内存
温度
耗电
调用栈
时间序列
```

最终诊断必须结合当前实测数据。

---

## 14.4 不是所有更新都会自动进入 Qdrant

以下情况可能出现：

```text
卡片已经更新
Wiki 已经更新
但 rag_docs 或 Qdrant 还没有更新
```

因此要判断在线检索是否生效，不能只看 Wiki，而要看：

```text
RAG 是否重建
Qdrant 是否重建
active release 是否切换
```

---

# 15. 面试表达版本

## 30 秒版本

> `perfdog_llm_wiki_starter` 是一个 LLM 驱动的知识 ETL 流水线，不是自主 Agent。它先把专家资料 URL 注册到 MySQL，通过规则推断平台、引擎、指标和抓取策略；随后普通网页走 Firecrawl，视频优先获取字幕、失败时用 ASR 转文本。正文经过规则质检后，使用 OpenAI-compatible LLM 结合强约束 Prompt 和 Pydantic Schema 抽取结构化性能知识卡，包括现象、根因、验证、优化、反证和来源证据。之后再使用规则优先、受限 LLM 语义标签补充的方式生成正式卡片，并渲染成 Markdown Wiki，供下游 Dual RAG 和诊断 Agent 使用。

## 被问“为什么不让 LLM 端到端完成？”

> 因为来源去重、抓取、状态管理、重试、路径、数据落盘和质量门禁都属于确定性工程问题，让 LLM 决定会不稳定、不可审计。我们只让模型处理它更擅长的非结构化理解：从正文抽取候选知识，以及处理少量语言风格、结论强度和工具依赖等语义边界；最终字段归一、指标白名单、去重、限额和可见性仍由代码控制。

## 被问“如何控制幻觉？”

> 首先只让通过正文质检的资料进入模型；其次 Prompt 限制只能抽取原文明示内容，禁止编造阈值、函数、调用栈和收益；输出经过 Pydantic 结构校验，失败最多做一次格式 repair；之后本地规则收紧指标、归一化字段、清理不适用字段、去重和限额。05 阶段模型不重新判断知识真伪，只输出固定语义标签。最后每张卡保留 `source_claims`、反证和不能下的结论，下游只能把它作为证据，而不能代替当前实测数据。

## 被问“它和 RAG 的关系？”

> Wiki Starter 负责生产和治理知识资产；Dual RAG 负责把正式知识卡和 Wiki 转成 `rag_docs.jsonl` 与 Qdrant 索引；Gateway 再把检索、Agent Context 和长任务编排成 HTTP/MCP 服务。三者的分工是知识生产、知识检索、统一服务编排。
