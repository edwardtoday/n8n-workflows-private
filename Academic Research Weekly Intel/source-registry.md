# Academic Research Weekly Intel — Source Registry

> 所有接口在 2025-10-23 通过公开文档或手工测试样例确认仍然可用；请在上线前用 n8n 运行一次健康检查（HTTP 200）并缓存响应。默认抓取窗口为 `today-7d … today`。

## 1. 全站热度信源

- **arXiv API**  
  `https://export.arxiv.org/api/query?search_query=all:__QUERY__&start=0&max_results=100&sortBy=submittedDate&sortOrder=descending`  
  - 热度指标：按提交时间衰减的曝光分 + 引用/收藏代理（见评分公式）。  
  - 支持无鉴权访问；速率限制 ~3 req/s。

- **OpenAlex Works API**  
  `https://api.openalex.org/works?filter=from_publication_date:{from},to_publication_date:{to}&sort=cited_by_count:desc&per-page=50`  
  - 提供引用量、作者、机构等元数据。  
  - 热度取 `log1p(cited_by_count)` 与社交提及（`counts_by_year.cited_by_count`）作为权重。

- **Semantic Scholar Graph Search**  
  `https://api.semanticscholar.org/graph/v1/paper/search?query=__QUERY__&limit=50&fields=title,abstract,url,authors,citationCount,publicationDate,externalIds`  
  - 可返回 altmetric 风格的引用与同行引用趋势。  
  - 需 100 req/day free quota，足以支撑周报任务。

- **Hugging Face Papers Leaderboard (HOTS)**  
  `https://huggingface.co/api/papers?sort=trending&limit=50`  
  - 覆盖机器学习/AI 热点；提供 `metrics.tweets`, `metrics.bookmarks` 等社交指标。

- **Google Research Pubs（HTML 列表页）**  
  `https://research.google/pubs/`  
  - 抓取方式：解析列表页 `row-card` 卡片（标题/作者/摘要预览/详情链接）。  
  - 时间字段：页面通常只展示年份，为避免误判“未来年份”且让其参与近 7 天评分，workflow 会用“列表顺序 + 本次抓取时间”生成 `publishedAt`（并在 `raw.publicationYear` 保留原始年份）。  

- **Google AI Research（精选链接聚合）**  
  `https://ai.google/research/`  
  - 抓取方式：从页面中提取 `blog.google/technology/*` 与 `research.google/blog/*` 的精选链接，再逐条抓取文章页 `og:title / og:description / article:published_time`（并尝试 JSON-LD `datePublished`/`author` 兜底）。  

- **OpenReview（Archive Direct Upload）**  
  `https://openreview.net/`（数据接口：`https://api2.openreview.net/notes?invitation=OpenReview.net/Archive/-/Direct_Upload`）  
  - 抓取方式：按 `tcdate` 倒序分页抓取最近条目，并用 `mintcdate`（=抓取窗口起始时间）做近 7 天过滤。  
  - 去噪策略：仅保留标题/摘要/venue 命中 AI/ML/代码相关关键词的条目，避免“非技术领域论文”刷屏（命中与否会记录在 diagnostics）。  
  - 字段映射：`title/authors/abstract/venue/pdf/html` → 统一字段；详情链接使用 `https://openreview.net/forum?id=<note_id>`。  

- **RSS（补充：只保留标题/摘要命中 AI 关键词的条目）**
  - medRxiv：`https://www.medrxiv.org/rss.xml`（健康/医疗领域预印本，用于捕捉“AI+医疗/健康/养老”相关论文与综述）
  - Nature：`https://www.nature.com/nature.rss`（综合期刊新闻/论文条目，作为“高影响力候选”的补充信号）
  - Nature / Machine learning：`https://www.nature.com/subjects/machine-learning.rss`（机器学习主题聚合）
  - Frontiers in AI：`https://www.frontiersin.org/journals/artificial-intelligence/rss`（AI 期刊条目）
  - EdSurge：`https://www.edsurge.com/rss`（教育科技新闻/研究解读，用于捕捉“AI+教育”方向）

## 2. 重点领域信源（去重后合计取 10 篇）

| 领域 | 数据源与示例查询 | 说明 |
| --- | --- | --- |
| 软件开发 | OpenAlex：`search=software+engineering`, arXiv CS.SE | 关注软件工程、DevOps、软件架构。 |
| 智能硬件 | Semantic Scholar：`query=embedded systems OR edge computing`, arXiv CS.AR | 覆盖芯片、嵌入式、机器人硬件。 |
| LED 照明/显示 | IEEE Xplore RSS：`https://ieeexplore.ieee.org/rss/TOC5100168.xml`；CNKI 工程照明筛选（HTML 抓取） | 注重 LED 设备、照明系统。 |
| 图像处理 | arXiv CS.CV、OpenAlex `search=image processing` | |
| 图像识别 | arXiv CS.CV、Semantic Scholar `query=image recognition` | |
| AI 总览 | Hugging Face Trending、arXiv AI 全类目 | |
| 自然语言处理 | arXiv CS.CL、ACL Anthology RSS：`https://aclanthology.org/anthology.rss` | |

> **中文信源**：  
> - **中国知网（CNKI）工程照明期刊聚合**：通过 n8n HTML 抓取 `https://navi.cnki.net/knavi/journals/CFLR/detail` 最新文章。  
> - **万方数据 LED 专题**：`https://s.wanfangdata.com.cn/paper?q=LED%20%E7%85%A7%E6%98%8E&f=top`（需 HTML 提取）。  
> - **深圳大学-LED 研究中心 RSS**：`https://ldc.szu.edu.cn/rss`（校内公开源）。

## 3. 评分指标设计

热度分 `score`（0-100）=  
```
score = 40 * recency_weight +
        35 * popularity_weight +
        15 * engagement_weight +
        10 * domain_priority
```

- **recency_weight**：`exp(-Δdays / 3)`，确保 3 天内论文更高分。  
- **popularity_weight**：`log1p(citations + downloads)` 正规化至 0-1；若无引用则回退到 arXiv `comment` / HF `metrics`.  
- **engagement_weight**：Twitter/Reddit/email mentions 等社交指标，经 `log1p` 正规化。  
- **domain_priority**：若命中重点领域标签，则赋值 1；全站榜单该值为 0。

去重顺序：DOI → arXiv ID → Semantic Scholar Paper ID → 标题模糊匹配（Jaro-Winkler ≥ 0.92）。  
输出顺序：总榜取前 10；领域榜各领域 0-3 条，总计 10 条，若不足则按评分补足其它热门领域。

## 4. 数据列字段

统一字段：`title`, `authors`, `abstract`, `publishedAt`, `source`, `sourceType`, `doi`, `arxivId`, `link`, `pdf`, `score`, `domainTags`, `metrics`（包含 `citations`, `bookmarks`, `downloads`, `socialMentions`）。

## 5. 限流策略

- arXiv：顺序调用，每 interval 3 req/s；n8n `httpRequest` 节点加入 500 ms 延迟。  
- OpenAlex：免费用户限 100 req/min → `batchSize=5` 并行。  
- Semantic Scholar：限 100 req/day → 聚合查询（每领域 1 请求）并缓存结果到数据存储。  
- Hugging Face：公开接口，无严苛限流，仍设置 2 req/s。

## 6. 失败兜底

- 如某源返回 0 条，记录 diagnostics 并标记 `sourceStatus=empty`，不强行创建占位章节。  
- 若全站榜不足 5 条，回退从上一周缓存（Workflow Data Store）补齐。  
- AI/Langchain 节点解析 JSON 失败时，启动二次修复链（沿用现有 Sansi 工作流的 repair 流程）。
