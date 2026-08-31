# `perfdog_dual_rag` 的完整使用流程与代码细节

先把整个过程看成两条链：

```text
A. 离线构建链：把知识卡变成可检索资产

Wiki Starter 正式知识卡 + Wiki
→ rag_docs.jsonl
→ 可选：Qdrant 向量库
→ build_report / manifest


B. 在线查询链：把一个性能问题变成证据包

查询参数
→ QuerySpec / EvidenceFrame
→ local 或 Qdrant 检索
→ rerank
→ final_reporter
→ retrieve package / agent_context
```

---

# 一、使用前：Dual RAG 依赖哪些上游产物？

Dual RAG 自己不抓网页、不调 LLM 抽卡。

它启动构建前，默认会去同级的：

```text
perfdog_llm_wiki_starter/
```

找这些资产：

```text
data/cards/cards_validated.jsonl
wiki/
wiki/index.md
wiki/index_core.md
```

其中最关键的是：

```text
cards_validated.jsonl
```

因为它是经过上游 `05_validate_cards.py` 校验后的正式知识卡。

一张卡大致有：

```text
card_id
source_id
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
verification_steps
optimization_actions
review_status
card_priority
rag_excluded
source_claims
```

Dual RAG 使用这些字段构建检索文档。

---

# 二、第一条链：构建 `rag_docs.jsonl`

核心代码在：

```text
builder.py
```

核心函数是：

```text
build_rag_corpus(...)
```

它的输入是：

```text
正式知识卡 JSONL
+ Wiki 目录
+ 输出目录
+ 是否允许 needs_manual_review 卡进入 RAG
```

默认输出：

```text
perfdog_dual_rag/data/rag_docs.jsonl
```

---

## 2.1 第一步：定位卡片文件和 Wiki 目录

构建开始时，代码会先确认输入路径。

默认优先寻找：

```text
perfdog_llm_wiki_starter/data/cards/cards_validated.jsonl
```

如果该路径不可用，才可能使用草稿卡作为兼容回退。

之后定位：

```text
perfdog_llm_wiki_starter/wiki/
```

同时读取：

```text
wiki/index.md
wiki/index_core.md
```

这两个索引页的作用不是提供主要文本内容，而是帮助判断：

```text
某张卡是否已经被 Wiki 收录
某张卡是否属于 core 核心知识
```

---

## 2.2 第二步：读取正式卡并决定“哪些卡可以进入 RAG”

构建器逐行读取：

```text
cards_validated.jsonl
```

每行解析成一张知识卡。

随后先进行过滤。

### 会跳过的卡

#### 情况一：`rag_excluded=true`

```text
卡片保留在知识库
Wiki 仍可展示
但不进入 rag_docs.jsonl
```

这是人为控制“是否参与检索”的开关。

#### 情况二：`review_status=needs_manual_review`

默认情况下，这类卡不会进入 RAG。

原因是：

```text
它已通过基础结构处理
但仍存在需要人工确认的风险
```

如果明确传入：

```text
include_needs_manual_review=true
```

构建器才允许它进入 RAG。

所以默认策略是：

```text
正式卡
≠ 一定进入 RAG

正式卡
+ 非 rag_excluded
+ 不需要人工复核
→ 才进入默认 RAG
```

---

## 2.3 第三步：扫描 Wiki，并建立 `card_id → Wiki 页面` 映射

构建器扫描 Wiki 下的 Markdown 文件。

但会跳过：

```text
index.md
index_core.md
```

因为它们是目录页，不是某张卡对应的正文页。

每个 Wiki 页面会尝试识别它属于哪张卡：

```text
优先读 front matter 的 id
→ 如果没有，尝试从文件名中识别 CARD-...
→ 将 ID 统一成大写
```

最终形成一个映射：

```text
CARD-XXX
→ wiki/03_root_causes/xxx.md
```

这个映射让 Dual RAG 知道：

```text
某张知识卡在 Wiki 中的页面在哪里
```

---

## 2.4 第四步：把一张知识卡转换为一条 RAG 文档

这一阶段是核心。

一张知识卡不是原样复制进 `rag_docs.jsonl`，而是重新组织成适合检索的文档。

可以理解成：

```text
Knowledge Card
→ Rag Document
```

构建后的 RAG 文档保留两类信息。

### 第一类：用于过滤、排序和解释的结构化字段

例如：

```text
card_id
source_id
card_type
platforms
engines
metrics
card_priority
review_status
rag_excluded
source_url
source_title
wiki_path
```

这些字段后面会用于：

```text
平台匹配
引擎匹配
指标对齐
筛选卡片类型
P0/P1 加权
人工审核状态加权
回到 Wiki 或来源
```

### 第二类：用于 BM25 和 embedding 的检索文本

构建器会把卡片的不同语义字段组织为检索文本，例如：

```text
标题
摘要
现象
根因描述
必要信号
支持信号
反证信号
验证步骤
优化动作
风险
局限
来源证据
Wiki 正文
```

这样做的目的不是“把所有字段塞在一起”，而是让检索可以从多个角度命中。

例如用户问：

```text
Android Unity 掉帧，CPU 高，GPU 未满载
```

可能命中：

```text
observed_pattern
required_signals
contradicting_signals
verification_steps
```

而不只依赖卡片标题。

---

## 2.5 第五步：写入 `rag_docs.jsonl`

构建器最终将每条 RAG 文档写入：

```text
data/rag_docs.jsonl
```

它是一个 JSONL 文件：

```text
一行 = 一条可检索文档
```

这份文件有两个用途：

```text
1. Local 检索直接读取它；
2. Qdrant 建库时读取它并生成向量。
```

所以它是 Dual RAG 的关键中间层。

---

## 2.6 第六步：生成 `build_report.json`

构建结束后，还会生成：

```text
data/build_report.json
```

报告会记录本轮构建信息，例如：

```text
输入 cards 路径
输入 Wiki 路径
输出 rag_docs 路径
总卡片数
最终纳入数
rag_excluded 跳过数
needs_manual_review 跳过数
Wiki 成功对齐数
核心索引收录数
各 card_type 数量
P0 / P1 数量
review_status 统计
```

它的作用是排障。

如果一张卡没有被检索到，不应该一开始就怪 Qdrant，而应该沿着下面顺序查：

```text
它是否存在于 cards_validated.jsonl？
→ 是否被 rag_excluded？
→ 是否是 needs_manual_review？
→ build_report 是否显示被跳过？
→ 是否写入 rag_docs.jsonl？
→ Qdrant 是否在这之后重建？
```

---

# 三、第二步：可选建立 Qdrant 向量库

核心代码在：

```text
vector_store.py
```

核心函数是：

```text
build_qdrant_index(...)
```

它的输入是：

```text
rag_docs.jsonl
+ dense_model
+ sparse_model
+ collection_name
+ qdrant_path 或 qdrant_url
+ batch_size
```

默认配置：

```text
dense_model      = thenlper/gte-large
sparse_model     = Qdrant/bm25
collection_name  = perfdog_rag_cards
batch_size       = 16
dense dimension  = 1024
```

---

## 3.1 第一步：读取 `rag_docs.jsonl`

建库代码逐行读取：

```text
rag_docs.jsonl
```

每条文档要准备：

```text
文档 ID
检索文本
结构化 metadata
```

其中：

```text
文档 ID
```

通常基于稳定卡片标识，例如 `card_id`。

这样后续 Qdrant 返回候选时，系统知道：

```text
这条向量对应哪张知识卡
```

---

## 3.2 第二步：生成 dense 向量

Dense embedding 模型：

```text
thenlper/gte-large
```

会将每条检索文本转换为 `1024` 维向量。

概念上：

```text
“CPU 主线程过载导致掉帧”
→ [0.21, -0.03, ..., 0.15]
```

dense 向量主要解决：

```text
用户表达与知识卡用词不同
但语义相近
```

例如：

```text
“游戏逻辑耗时太高造成卡顿”
```

和：

```text
“主线程计算过重导致 FPS 下跌”
```

即使不完全相同，也能有较高相似度。

---

## 3.3 第三步：生成 sparse 向量

Sparse 模型：

```text
Qdrant/bm25
```

会为文本建立关键词、词权重等稀疏表示。

它更擅长匹配：

```text
avgFps
cpuAvgPct
p95
Jank
GC
Unity
Android
```

这类精确指标名和专业术语。

为什么不能只用 dense？

因为：

```text
dense 擅长语义
但不一定足够重视精确字段

性能检索需要既理解“掉帧”
也精确理解“avgFps”“cpuAvgPct”
```

所以项目建库时同时保存：

```text
dense vector
+ sparse vector
```

---

## 3.4 第四步：写入 Qdrant Point

Qdrant 中每条记录可以理解为一个 Point。

一个 Point 包含：

```text
point id
dense vector
sparse vector
payload
```

其中 `payload` 是普通结构化字段，例如：

```text
card_id
source_id
card_type
platforms
engines
metrics
card_priority
review_status
rag_excluded
title
summary
```

向量用于“相似度搜索”。

payload 用于“精确过滤”。

例如用户请求：

```text
platform = Android
engine = Unity
card_types = ["root_cause", "verification_method"]
```

系统可以尽量避免把：

```text
iOS 专属
Unreal 专属
优化类
```

等不匹配的文档混入候选。

---

## 3.5 第五步：本地 Qdrant 的安全建库方式

如果使用：

```text
qdrant_path
```

即本地目录模式，代码不会直接破坏当前索引。

而是大致按下面思路：

```text
创建临时 Qdrant 目录
→ 向临时目录写入完整新索引
→ 验证本次写入 point_count
→ 成功后替换原目录
```

这样可以避免：

```text
旧索引被删
→ 新索引构建到一半失败
→ 最终没有任何可用索引
```

也就是说：

```text
先完成新版本
→ 再切换
```

---

## 3.6 为什么本地 Qdrant 要加锁？

本地 Qdrant 使用文件目录保存数据。

如果两个任务同时：

```text
一个任务重建索引
另一个任务查询索引
```

可能出现并发问题。

因此项目有：

```text
QDRANT_OPERATION_LOCK
```

它在本进程中保护本地 Qdrant 操作。

注意边界：

```text
QDRANT_OPERATION_LOCK
= 进程内锁

不是
= 多机器、多容器之间的分布式锁
```

所以本地 Qdrant 更适合：

```text
单机
开发环境
单实例部署
```

---

## 3.7 远程 Qdrant 如何切换索引？

如果使用：

```text
qdrant_url
```

即远程 Qdrant Server，构建可以使用更安全的 collection / alias 思路。

概念上：

```text
构建新 collection
→ 写入全部新点
→ 校验完成
→ 将稳定 alias 指向新 collection
```

例如：

```text
当前 alias:
perfdog_rag_cards
→ 指向旧 collection

新构建完成后：
perfdog_rag_cards
→ 指向新 collection
```

这样查询方始终访问：

```text
perfdog_rag_cards
```

不需要关心真实版本 collection 名。

---

## 3.8 第六步：生成 `qdrant_manifest.json`

建库完成后，写入：

```text
data/qdrant_manifest.json
```

它记录：

```text
使用了哪个 rag_docs.jsonl
使用了哪个 dense 模型
使用了哪个 sparse 模型
dense 维度
collection 名称
写入点数
本地路径或远程 URL
构建时间
```

它的用途是检查：

```text
查询模型与建库模型是否一致
collection 是否一致
当前索引来自哪一份语料
点数是否合理
```

---

# 四、第三条链：一次查询请求如何被处理？

现在假设调用：

```json
{
  "backend": "qdrant",
  "platform": "Android",
  "engine": "Unity",
  "metrics": ["avgFps", "cpuAvgPct"],
  "symptoms": ["掉帧", "CPU 高"],
  "anti_signals": ["GPU 未满载"]
}
```

入口在：

```text
service_facade.py
```

主要对外函数：

```text
build_retrieval_service_payload()
build_agent_context_service_payload()
```

完整调用链：

```text
请求 JSON
→ 参数归一化
→ QuerySpec
→ EvidenceFrame
→ local 或 Qdrant 检索
→ reranker
→ final_reporter
→ 返回 package 或 agent_context
```

---

# 五、请求参数归一化：`service_facade.py`

服务入口首先做的不是检索，而是参数校验与默认值填充。

例如：

```text
backend
```

只能是：

```text
local
qdrant
```

例如：

```text
fusion
```

只能是：

```text
rrf
dbsf
```

例如：

```text
rerank_mode
```

只能是：

```text
none
local_hybrid
```

如果用户传错类型，例如：

```text
top_k = "abc"
```

会在真正检索前报参数错误，而不是进入底层后出现难排查异常。

---

## 5.1 主要参数是什么意思？

```text
backend
→ 选择 local 或 qdrant

platform
→ Android / iOS / Windows 等

engine
→ Unity / Unreal 等

metrics
→ 当前观测到的具体性能指标

symptoms
→ 当前现象，例如掉帧、卡顿、内存上涨

anti_signals
→ 与某类根因冲突的反向证据

top_k
→ 最终每个卡片类型保留多少结果

candidate_limit
→ Qdrant 初步召回多少候选，默认 128

strict_platform
→ 平台不匹配是否直接过滤

strict_engine
→ 引擎不匹配是否直接过滤

fusion
→ RRF 或 DBSF

dense_only
→ 是否关闭 sparse，只做 dense 检索

rerank_mode
→ 是否进行 local_hybrid 二阶段精排

include_debug
→ 是否返回完整评分、候选与调试信息
```

---

# 六、`QuerySpec` 和 `EvidenceFrame` 是怎么工作的？

## 6.1 `QuerySpec`

`QuerySpec` 是一次查询的结构化条件。

它把请求变成统一对象：

```text
平台是什么
引擎是什么
指标是什么
症状是什么
哪些是反证
想要哪些卡片类型
平台/引擎是否严格过滤
```

它解决的问题是：

> 后面 Local、Qdrant、reranker 都使用同一种查询表达，而不是每层自己解析 HTTP JSON。

---

## 6.2 `EvidenceFrame`

`EvidenceFrame` 是在 `QuerySpec` 基础上产生的“当前证据框架”。

它会把：

```text
metrics
symptoms
anti_signals
```

做规则化理解。

例如：

```text
avgFps
→ 帧率指标族

cpuAvgPct
→ CPU 指标族

掉帧
→ 帧率异常现象

GPU 未满载
→ GPU 根因的反向证据
```

它后续主要服务于：

```text
evidence alignment
```

即：

```text
这张卡的假设、支持信号、反证条件
是否和当前问题真的一致
```

---

# 七、如果 `backend=local`：本地检索怎么跑？

核心代码：

```text
retriever.py
text_utils.py
```

流程：

```text
读取 rag_docs.jsonl
→ 建立 BM25 索引
→ 对每条 doc 计算 token Counter
→ 遍历文档
→ 先做硬过滤
→ 再计算文本相关性
→ 再计算业务规则加减分
→ 按 card_type 分组取 Top K
```

---

## 7.1 第一步：硬过滤

如果设置：

```text
strict_platform=true
```

而某张卡明确不适用于 Android，则直接返回：

```text
None
```

不参与排序。

`strict_engine=true` 同理。

此外还会基于：

```text
card_types
card_priorities
review_statuses
exclude_warnings
```

进行过滤。

---

## 7.2 第二步：文本基础分

对查询和文档计算：

```text
BM25
+
Counter cosine
```

基础公式：

\[
baseScore=(0.58 \times BM25)+(0.42 \times CounterCosine)
\]

含义：

```text
BM25
→ 精确词、指标名、术语命中

Counter cosine
→ 查询和文档的词频分布是否相近
```

---

## 7.3 第三步：平台、引擎倍率

在基础分上乘以平台和引擎适配倍率。

平台：

```text
精确匹配：1.08
通用文档：0.92
普通不匹配：0.68
strict 不匹配：直接排除
```

引擎：

```text
精确匹配：1.05
通用文档：0.94
普通不匹配：0.72
strict 不匹配：直接排除
```

---

## 7.4 第四步：Evidence 对齐

系统会根据当前 `EvidenceFrame`，查看卡片是否匹配：

```text
指标族
required_signals
supporting_signals
contradicting_signals
症状
趋势
场景
验证方式
```

匹配会加分。

冲突会扣分。

例如：

```text
用户输入：
GPU 未满载

某卡：
“GPU 满载造成掉帧”
```

这张卡会被降权。

---

## 7.5 第五步：知识质量加减分

再加上：

```text
P0：+0.14
核心索引：+0.04
human_reviewed：+0.12
auto_checked：+0.04
warning：逐项扣分，总上限约 0.20
```

最终分数可理解为：

\[
score =
((0.58 \times BM25)+(0.42 \times CounterCosine))
\times platformMultiplier
\times engineMultiplier
+ evidenceBoost
+ priorityBoost
+ reviewBoost
- warningPenalty
\]

---

# 八、如果 `backend=qdrant`：向量检索怎么跑？

核心代码：

```text
qdrant_retriever.py
vector_store.py
```

流程：

```text
QuerySpec
→ 构建查询文本
→ 生成 dense 查询向量
→ 生成 sparse 查询向量
→ 按 metadata 过滤
→ Qdrant 获取候选
→ dense / sparse fusion
→ 候选集交给 reranker
```

---

## 8.1 查询文本如何构建？

系统不会只把一句症状传给向量模型。

而是把结构化字段拼成查询文本，例如：

```text
平台：Android
引擎：Unity
指标：avgFps、cpuAvgPct
现象：掉帧、CPU 高
反证：GPU 未满载
```

这样 dense 与 sparse 查询都能感知：

```text
问题场景
专业指标
具体症状
反向条件
```

---

## 8.2 Metadata 过滤怎么做？

Qdrant 中每条 Point 都有 payload。

查询时可以尽量按：

```text
card_type
platform
engine
review_status
card_priority
```

过滤或缩小范围。

这一步的目标是：

```text
先不让明显无关的卡进入候选集
→ 再计算向量相似度
```

---

## 8.3 Dense 与 Sparse 如何融合？

dense 和 sparse 分别得到候选。

再按 `fusion` 参数融合：

```text
rrf
或
dbsf
```

默认：

```text
dbsf
```

得到最终候选集。

注意，这时只是：

```text
候选排序
```

还不是最终业务排序。

---

# 九、`reranker.py`：候选如何被二阶段精排？

核心概念：

> 本项目的 reranker 不是模型，而是本地规则精排器。

默认：

```text
rerank_mode=local_hybrid
```

它的工作是：

```text
Qdrant 召回较大的 candidate 集
→ 对 candidate 应用 Local 的业务评分规则
→ 得到最终排序
```

例如：

```text
Qdrant 语义上认为 GPU 卡很相关
但 anti_signals 表示 GPU 未满载
→ reranker 将 GPU 卡降权

Qdrant 召回 P1 auto_checked 卡
同时有 P0 human_reviewed 卡
→ reranker 提升 P0 human_reviewed 卡
```

所以最终结构是：

```text
Qdrant
→ 负责广召回

reranker
→ 负责把候选按平台、引擎、证据、优先级、审核、warning 重新排序
```

这就是两阶段检索。

---

# 十、`final_reporter.py`：如何把结果变成 Agent 可用证据？

检索与精排结束后，不能直接把候选 JSON 返回。

`final_reporter.py` 会继续做：

```text
回表补充卡片原字段
→ 按卡片类型分组
→ 整理来源
→ 整理证据
→ 形成 final package
→ 形成 agent_context
```

---

## 10.1 为什么要“回表”？

Qdrant payload 和候选检索结果通常是检索所需的压缩字段。

但最终要让 Agent 使用时，需要补充：

```text
完整摘要
现象
必要信号
支持信号
反证
不能下的结论
验证步骤
优化动作
来源证据
风险与局限
```

因此最终报告器会根据：

```text
card_id
```

回到 `rag_docs` 或 cards 相关数据中补齐。

---

## 10.2 最终包按什么组织？

不是单纯：

```text
Top 1
Top 2
Top 3
```

而是按诊断角色组织：

```text
phenomenon_pattern
→ 现象解释

root_cause
→ 候选根因

verification_method
→ 验证方法

optimization_action
→ 优化建议

case_evidence
→ 历史案例或支撑材料

platform_difference
→ 平台差异
```

这样下游 Agent 可以按正确的诊断顺序工作：

```text
先理解现象
→ 提出候选根因
→ 设计验证步骤
→ 再讨论优化
```

---

## 10.3 `agent_context` 与普通检索结果有什么区别？

普通检索结果：

```text
更完整
包含更多 ranking、分数、候选与调试信息
适合查看检索行为
```

`agent_context`：

```text
更精简
更强调证据组织
适合直接注入下游 Agent Prompt
```

可以理解为：

```text
retrieve package
= 给系统调试或调用方看的完整检索包

agent_context
= 给下游智能体阅读的案卷材料
```

但仍然要强调：

> `agent_context` 是证据包，不是最终根因结论。

---

# 十一、Dual RAG 和 Gateway 怎么连接？

Gateway 不重写 Dual RAG 算法。

它通过 `adapters/dual_rag.py` 直接调用：

```text
build_retrieval_service_payload()
build_agent_context_service_payload()
build_rag_corpus()
build_qdrant_index()
```

对外暴露为：

```text
POST /api/retrieve
POST /api/agent-context
POST /api/dual-rag/build-assets
```

以及 MCP Tool：

```text
perfdog_retrieve_knowledge
perfdog_build_agent_context
perfdog_dual_rag_build_assets
```

调用关系：

```text
HTTP / MCP
→ Gateway runtime
→ Dual RAG adapter
→ service_facade
→ 检索或构建
```

读操作，例如检索：

```text
同步返回
```

写操作，例如构建 corpus / Qdrant：

```text
进入 MySQL Job
→ Worker 后台执行
```

---

# 十二、一个实际使用过程示例

假设你新增了一条文章，并通过 Wiki Starter 完成了：

```text
抓取
→ 抽卡
→ 校验
→ Wiki 渲染
```

此时新卡已经存在：

```text
cards_validated.jsonl
```

但还不能保证能被 RAG 搜到。

接下来需要：

```text
Dual RAG full build
```

实际过程：

```text
cards_validated.jsonl
+ wiki/
→ build_rag_corpus()
→ data/rag_docs.jsonl
→ build_qdrant_index()
→ data/qdrant/
→ qdrant_manifest.json
→ 可选发布 active release
```

之后发起查询：

```json
{
  "backend": "qdrant",
  "platform": "Android",
  "engine": "Unity",
  "metrics": ["avgFps", "cpuAvgPct"],
  "symptoms": ["掉帧"],
  "anti_signals": ["GPU 未满载"]
}
```

内部执行：

```text
JSON 参数归一化
→ QuerySpec
→ EvidenceFrame
→ dense / sparse 查询向量
→ Qdrant hybrid 候选召回
→ local_hybrid rerank
→ final_reporter
→ agent_context
```

最终 Agent 收到：

```text
与当前掉帧最相关的现象卡
+ CPU 方向的候选根因卡
+ 应如何验证的卡
+ 可尝试优化的卡
+ 每条卡的来源与反证边界
```

---

# 十三、这套代码最重要的边界

这部分是理解代码时必须记住的事实：

```text
Dual RAG 不生产知识
→ 上游 Wiki Starter 才生产卡片

Dual RAG 不直接分析 PerfDog 原始报告
→ 它只消费调用方传入的结构化症状和指标

Qdrant 不负责最终排序
→ 它主要做候选召回

reranker 不是模型
→ 它是本地业务规则精排

agent_context 不是最终诊断结论
→ 它是供下游 Agent 推理的证据材料

Local 不一定是 Qdrant 的自动故障降级
→ 当前由调用方显式选择 backend
```
