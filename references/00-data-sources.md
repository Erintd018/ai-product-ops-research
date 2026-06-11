# 信息源地图 · 采集工具 · 合规边界

> 「要什么去哪找、用什么取」的境内外大表。所有 Agent 共享。
> 原则：官方/公开优先，尊重 robots.txt 与服务条款，不绕过登录态与付费墙，不抓个人隐私。
> 分析需要什么数据，就用合适的方式去取。

---

## 1. 境内外主流平台地图

| 类别 | 境内平台 | 境外平台 |
|---|---|---|
| 应用分发 | App Store（中区）、华为/小米/OPPO/vivo 应用市场、豌豆荚 | App Store（全球）、Google Play、APKPure |
| 社交/短视频 | 抖音、快手、小红书、微博、视频号、B站 | TikTok、Instagram、YouTube、X(Twitter) |
| 内容社区 | 知乎、即刻、V2EX、酷安、少数派、CSDN | Reddit、Hacker News、Product Hunt、Medium、Dev.to |
| 即时通讯/社群 | 微信（公众号/社群/小程序）、QQ 群 | Discord、Telegram、Slack |
| 专业评测/口碑 | 36氪、极客公园、爱范儿、IT之家 | G2、Capterra、TrustRadius、AlternativeTo |
| 开发者 | GitHub（中文项目/Issues）、稀土掘金、SegmentFault | GitHub、Stack Overflow、HuggingFace |
| 视频/直播 | B站、抖音直播、视频号直播 | YouTube、Twitch |
| 职场/行业 | 脉脉、Boss直聘社区 | LinkedIn、Glassdoor |

---

## 2. 行业新闻与数据源

| 类别 | 境内 | 境外 |
|---|---|---|
| 科技新闻 | 36氪、极客公园、量子位、机器之心、虎嗅 | TechCrunch、The Verge、The Information、Ars Technica、Wired |
| AI 专业 | 机器之心、新智元、AI 科技评论 | AI News、MIT Technology Review、arxiv/HuggingFace Papers |
| 投融资 | IT桔子、36氪融资频道、天眼查 | Crunchbase、PitchBook、CB Insights |
| 数据/排名 | 七麦数据、蝉大师、百度指数、微信指数 | Sensor Tower、data.ai、SimilarWeb、Google Trends |
| 政策/监管 | 网信办、工信部公告、国家标准委 | EU AI Act 动态、FTC、NIST AI Framework |

---

## 3. 数据采集工具体系

### 分层架构

| 层级 | 定位 | 工具 | 何时使用 |
| --- | --- | --- | --- |
| **Level 1** | MCP Search API | Exa / Firecrawl / Tavily | MCP 已配置且可用时，Sub-agent 直接调用 |
| **Level 2** | 内置工具 | WebSearch / WebFetch | MCP 不可用时，Orchestrator 预搜索 |
| **Level 3** | 辅助工具 | gstack / Playwright | 需要截屏或 JS 渲染时 |

### MCP Server 配置

| 名称 | 安装方式 | 核心能力 | 主要服务 | 降级方案 | 限流约束 |
| --- | --- | --- | --- | --- | --- |
| **Exa MCP** | `npx -y exa-mcp-server` | 语义搜索，公司/新闻/研究索引 | Agent 1 (竞品+事件) | Orchestrator WebSearch | 官方未公开 |
| **Firecrawl MCP** | `npx -y firecrawl-mcp` | URL 深度爬取 + 结构化提取 | Agent 1 (官网), Agent 3 (定价页) | WebFetch | 官方未公开 |
| **Tavily MCP** | `npx -y tavily-mcp@latest` | 实时搜索，180ms，高频低成本 | Agent 2 (舆情口碑) | Orchestrator WebSearch | ⚠️ 100 RPM（Dev）/1000 RPM（Prod）|

**Tavily 限流详情**：详见 `tavily-rate-limits.md`

### 降级策略（模式 B）

当 MCP 不可用时：
1. Orchestrator 用 WebSearch 执行关键搜索
2. 将搜索结果作为上下文注入 Sub-agent prompt
3. Sub-agent 基于已有信息做结构化分析，不再自行搜索
4. 信息不足时标注"未获取（原因：XX）"

### 采集方式选型（按场景）

| 采集方式 | 适用场景 | 工具 |
|---|---|---|
| AI 语义搜索 | 公司融资、新闻、行研、竞品发现 | Exa MCP |
| 实时通用搜索 | 社媒发现、舆情、品牌提及 | Tavily MCP |
| 确定 URL 爬取 | 官网、定价页、changelog、博客 | Firecrawl MCP |
| 页面内容获取 | 已知 URL、简单页面 | WebFetch |
| GUI 截屏 | 反爬页面、需要视觉证据 | gstack |
| 情感分析 | 评论主题编码与情感打分 | Claude 原生能力 |

---

## 4. 信息源按用途索引

| 要找的东西 | 去哪找 |
|---|---|
| 产品定位、功能、定价 | 官网 Pricing 页、产品文档、FAQ（Firecrawl 爬取） |
| 版本迭代节奏 | Changelog / Release notes / Blog、GitHub releases（Firecrawl 监控） |
| 新品与功能发布 | 官方博客、X/微博官号、Product Hunt（Exa 搜索） |
| 公司背景、融资 | Crunchbase、PitchBook、IT桔子、天眼查（Exa 搜索） |
| 用户规模/流量 | SimilarWeb、应用商店榜单、Sensor Tower、data.ai |
| 应用商店评论 | App Store / Google Play / 各安卓市场（Apify 批量抓取） |
| B端口碑 | G2、Capterra、TrustRadius |
| 社区讨论 | Reddit、知乎、V2EX、即刻、Hacker News（Apify 抓取） |
| 社媒运营数据 | 各平台公开页面（Firecrawl）+ mcp-metricool |
| 行业政策 | 网信办、工信部、EU AI Act、FTC（Exa 新闻搜索） |

---

## 5. 合规与可信度

- **合规红线**：不绕过登录态/付费墙；不采集可识别个人隐私；大规模采集优先走官方 API；尊重 robots.txt
- **可信度分级**：官方一手 > 权威媒体/分析机构 > 第三方估算工具 > 社区个人观点
- **标注要求**：第三方流量/下载数据标注「估算」；定价/用户量/融资标注获取日期
- **样本偏差**：应用商店评论偏极端；社区活跃用户≠全体用户；分析时显式说明
