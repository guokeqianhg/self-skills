# 1. `knowledge_asset_pipeline`：知识资产构建与治理

## 1.1 实际定位

这是**上游 ETL/知识治理工程**。它把外部网页、视频和专家资料转换为可审核的知识卡和 Markdown Wiki。它没有 embedding、Qdrant client 或在线召回算法；`rag_excluded` 只是给下游 RAG 构建提供可见性标记。

主链路按脚本编号组织，说明它是脚本型流水线，而非单体 Web 服务：

```text
01 来源登记 → 02 抓取/字幕 → 03 正文校验 → 04 LLM 抽卡
→ 05 卡片校验 → 06 Wiki 渲染
```

## 1.2 来源登记与 MySQL

### `01_build_sources.py`

- 读取根目录 Markdown 清单，提取链接、标题、章节上下文。
- 规范化 URL，降低同一资料因 query/trailing slash 等重复入库概率。
- 用 URL/稳定字段构建 source id，基于章节与文本做平台、引擎、主题、指标等启发式推断。
- upsert 到 MySQL `sources`；只更新自动推断字段，避免覆盖人工字段和抓取/审核状态。

### `01b_add_single_source.py`

- 单 URL 快速入口，通过 `importlib` 动态加载数字开头的 `01_build_sources.py` 并复用推断逻辑。
- 只负责登记，不自动执行 02–06；完整执行由 gateway 的 ingest pipeline 提供。

### `registry_db.py`

这是来源注册表 DAO 和兼容层：

- 把 MySQL row 与历史 JSONL 的嵌套对象互转。
- 分离自动列、人工维护列、抓取列，防止批量同步破坏人工审阅结果。
- 来源编辑采用版本/乐观锁语义：写入时校验期望版本，冲突时提示重新读取而不是静默覆盖。
- 负责 schema 兼容、查询、upsert、批量状态回写。

### `00_migrate_registry_to_mysql.py` 与 `sync_sources_table.py`

- 前者是一次性迁移：优先 JSONL、Excel 做补充；默认保护目标库，`--force` 才覆盖。
- 后者是 MySQL→MySQL 同步，读取两端列结构后做 upsert；`--delete-missing` 是显式破坏性同步选项。

**面试说法**：MySQL 是来源流程的状态机和协作真源；JSONL/Excel 是兼容快照。把二者分开避免了“每次跑脚本把人工改动覆盖”的常见 ETL 问题。

## 1.3 抓取、清洗与视频 ASR

### `02_scrape_sources.py` + `scrape_cleaning.py`

- 选择符合条件的来源，调用 网页抓取服务 抓取网页正文。
- 对返回内容做清洗并写三类可追溯资产：原始响应 `raw/`、抓取元信息 `meta/`、标准 Markdown `markdown/`。
- 将 fetch status / 时间 / 策略 / 错误等回写 `sources`，使失败可重试、可审计，而不是把失败隐藏在临时日志。

### `02b_fetch_video_transcripts.py`

- 面向 视频平台 等视频：优先获取字幕；必要时使用页面/API 备用路径。
- 没有可用字幕时，可下载音频，经 ffmpeg 转 WAV、按片段处理，再调用 Whisper 或 通用模型兼容接口 ASR。
- 处理 cookie、字幕解析、音频分块、失败信息和同样的 raw/meta/markdown 落盘。

### `02c_retry_failed_fetches.py`

根据已记录的 `fetch.strategy` 对失败项分流回 02 或 02b，不用人工猜“这个来源到底该走网页还是视频路径”。

## 1.4 正文校验与抽卡

### `03_validate_scrapes.py`

对 Markdown/元数据做确定性质量检查，如空正文、长度、有效内容等，回写抓取审核状态。它把“能请求成功”与“内容适合让 LLM 抽取”分开，减少垃圾输入对后续成本和质量的污染。

### `04_extract_cards.py`

- 使用 通用模型兼容接口 Chat Completions；模型、base URL、key 由环境变量控制。
- 以每篇清洗正文为输入，要求 LLM 产出结构化性能知识卡。
- 使用 Pydantic 模型约束卡片结构，再做本地后处理与 JSONL 写入。
- 同时写 usage 记录，便于核算模型消耗和故障排查。

知识卡面向“性能现象 → 根因假设 → 验证 → 优化”建模，常见字段包括 card id/source id、类型、优先级、适用平台/引擎、指标、信号、证据声明、验证步骤、优化动作、风险/反证、review 状态及 `rag_excluded`。

### `models.py` 与 `metric_aliases.py`

- `models.py` 承担 Pydantic schema 与约束，减少 LLM 自由文本直接进入正式资产。
- `metric_aliases.py` 将指标别名对齐，降低“FPS/avgFps/平均帧率”等不同写法导致的碎片化。

## 1.5 卡片校验：规则优先，批量语义兜底

### `05_validate_cards.py`

该阶段不只是 JSON schema 校验，而是分层治理：

1. 本地结构规则验证字段完整性、优先级、引用等。
2. 对明显类型（例如工具引用、特定 claim tone/content role）用本地启发式短路。
3. 只把边界样本批量送入 LLM 语义打标，默认批量 12 张，减少逐卡 LLM latency/cost。
4. 用语义缓存避免重跑重复判断；当前代码中的语义 Prompt 版本为 `2026-06-23-v2`，缓存 key 也纳入该版本、模型和待审 payload。

5. 输出 validated JSONL、验证 manifest，并按 P0/P1 等做分层。

这体现的工程思想：**确定性问题用确定性规则，模糊语义问题才付费给模型；把 N 次网络调用压缩为批量调用。**

## 1.6 Wiki 与管理工具

### `06_render_wiki.py`

将经过验证的卡片渲染为有组织的 Markdown Wiki、索引和主题页。Wiki 是给人阅读/审阅的知识展示层；JSONL 是机器处理和下游构建的数据层。

### `cards_admin.py`

支持：

- 按 `card_id` / `source_id` 删除卡片；
- 不删除卡片，仅切换 `rag_excluded`；
- 允许保留 Wiki 证据，但临时不进入下游检索。

### `webapp/app.py`

人机协作管理端，负责来源人工编目、卡片浏览和来源状态检查。核心价值不是替代 pipeline，而是把 `enabled`、`ingestion_priority`、`include_in_wiki`、标题/指标 override、notes 等人工判断安全写回 MySQL。

## 1.7 存储、失败处理与可恢复性

| 层 | 资产 | 失败后如何处理 |
| --- | --- | --- |
| 注册表 | MySQL `sources` | 查状态、修字段、基于策略重跑 |
| 原始资料 | `data/raw` | 保留原始响应排障 |
| 元数据 | `data/meta` | 查抓取策略、时间、错误 |
| 清洗文本 | `data/markdown` | 重新校验/抽取 |
| 草稿卡 | `cards_draft.jsonl` | 修 prompt/规则后重跑验证 |
| 正式卡 | `cards_validated.jsonl` | 审阅、删除或切 RAG 可见性 |
| 展示层 | `wiki/` | 根据卡片重渲染 |

## 1.8 面试时可追问的改进

- 使用数据库/对象存储记录每阶段 idempotency key，进一步提升重跑幂等性。
- 把 JSONL 正式资产改为带版本的对象存储/数据库表，支持历史版本回滚。
- 为 HTML 抓取、字幕/ASR、卡片 schema、渲染建立 fixture 测试；当前仓库未见完整自动化测试套件。
- 对 LLM 抽取增加 JSON schema mode、重试/backoff、prompt version 和质量抽检闭环。
