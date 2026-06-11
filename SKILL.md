# AI 产品运营研究 Skill

> 输入一个 AI 产品名，自动完成竞争态势、用户增长、商业化三维诊断，产出 90 天运营策略。

---

## 触发方式

用户说「研究 {产品名}」或「分析 {产品名} 的运营」时启动。

测试时对 Orchestrator 说：
> 「用测试模式研究 {产品名}」
> 测试模式按 TEST-PROTOCOL.md 定义的检查点逐步执行。

---

## Orchestrator 职责

1. **解析意图**：识别产品名 + 侧重点（全量/单维度）
2. **快速摸底（Product Discovery）**：用品牌词搜索确认产品定位、赛道、核心场景，消除认知盲区
3. **生成初始关键词矩阵**：品牌词/场景词/功能词/情感词/事件词（不含竞品词，竞品由 Agent 1 确认）
4. **Phase 1 · 产品定位与竞争态势**：派发 Agent 1，获取竞品列表和市场全景
5. **补全关键词矩阵**：基于 Agent 1 确认的竞品，补充竞品词
6. **Phase 2 · 用户分析 + 商业化诊断（并行）**：携完整关键词矩阵派发 Agent 2、Agent 3
7. **Phase 3 · 质量审核**：派发 Agent Q 审核各报告，未达标则打回（最多 2 轮）
8. **Phase 4 · 策略综合（分两步）**：
   - Step 1: 基于诊断事实提炼核心命题（"这个产品面临的根本问题是什么"），暂停与用户讨论
   - Step 2: 基于讨论共识产出策略方向和执行方案

---

## 分析逻辑（用户视角）

```
维度 1 · 产品定位与竞争态势（Agent 1）
  → 这个产品在哪、对手在哪、趋势往哪走
  → 竞品格局 + 关键事件 + 投放声量 + SEO/GEO

维度 2 · 用户分析与增长诊断（Agent 2）
  → 用户是谁、体验如何、增长卡在哪
  → 口碑舆情 + 用户旅程 + 服务质量 + PLG 成熟度

维度 3 · 商业化诊断（Agent 3）
  → 到底要挣谁的钱、结合生态优势还能挣什么钱、路径通不通、断在哪
  → 定价策略 + 渠道生态 + 转化漏斗 + 动作-转化关联

质量审核（Agent Q）
  → 独立审核数据有源、分析有框架、结论有洞察

策略综合（Orchestrator + 用户讨论）
  → Step 1: 基于诊断，提炼产品面临的核心命题（不是直接给方案）
  → Step 2: 与用户讨论命题，挑战假设，形成共识
  → Step 3: 基于共识产出策略方向和执行方案
```

---

## 工作流

```
Phase 0 · 摸底
  解析意图 → Product Discovery（搜索+抓取自身及核心竞品官网主页含导航栏）→ 生成初始关键词矩阵

Phase 1 · 产品定位与竞争态势
  派发 Agent 1 → 获取竞品列表 + 市场全景
  → Orchestrator 补全关键词矩阵（加入竞品词）

Phase 2 · 用户分析 + 商业化诊断（并行）
  派发 Agent 2（用户分析）+ Agent 3（商业化）
  → 两者共享完整关键词矩阵和 Agent 1 竞品列表

Phase 3 · 质量审核
  派发 Agent Q 逐份审核 → 未达标打回修订（≤2 轮）

Phase 4 · 策略综合（分两步，⚠️ 正式模式下也必须暂停讨论）
  Step 1: 提出核心命题 → 基于诊断事实+`references/06-strategic-thinking.md`追问清单，提炼核心命题
         → **必须暂停**，向用户展示命题和初步判断，发起讨论
         → 不跳过。没有讨论就没有犀利的策略——这是报告质量的分水岭
  Step 2: 基于讨论共识产出策略方向 → 策略按市场/用户/商业分类
```

---

## 调度逻辑

```
意图路由：
  IF 用户只说「研究 X」→ 全量执行（Phase 0-4）
  IF 用户指定侧重 → 只调对应 Agent，但 Agent 1 始终先行

采集模式（启动时自动判断）：
  IF MCP 工具可用（Exa/Firecrawl/Tavily）→ 模式 A
  ELSE → 模式 B

模式 A：Sub-agent 直接调用 MCP 工具搜索+分析
模式 B：Orchestrator 用 WebSearch 预搜索 → 结果注入 Sub-agent prompt → Sub-agent 纯分析

测试模式：
  用户说「测试模式」时，按 TEST-PROTOCOL.md 检查点逐步执行
```

---

## Agent 路由表

| 用户意图 | 调用 Agent | 输出模板 |
|---|---|---|
| 竞品/格局/事件/投放 | product-positioning | product-positioning |
| 用户/口碑/增长/PLG | user-growth-diagnosis | user-growth-diagnosis |
| 商业化/定价/渠道/转化 | commercialization-diagnosis | commercialization-diagnosis |
| 策略（需全部输入） | Orchestrator 综合 | ops-strategy-90day |
| 全部（默认） | 全部 Agent | 全部模板 |

---

## 数据采集工具（分层架构）

### Level 1 · MCP Search API（优先）

| MCP | 能力 | 主要服务 | 限流约束 |
| --- | --- | --- | --- |
| **Exa MCP** | 语义搜索，公司/新闻/研究专项索引 | Agent 1 (竞品情报、事件追踪) | 官方未公开，默认安全 |
| **Firecrawl MCP** | 确定 URL 深度爬取 + 结构化提取 | Agent 1 (官网)、Agent 3 (定价页) | 官方未公开，默认安全 |
| **Tavily MCP** | 高频实时搜索，180ms 延迟 | Agent 2 (舆情口碑)、全局通用搜索 | ⚠️ 限流：100 RPM（Dev）/1000 RPM（Prod） |

配置见项目根目录 `.mcp.json`。Sub-agent 自动继承。

**Tavily 限流策略**：

- 详见 `references/tavily-rate-limits.md`
- 单 Agent 调用次数限制：Agent 1 ≤8 次，Agent 2 ≤10 次，Agent 3 ≤6 次
- 所有调用串行执行，间隔 ≥2 秒
- 遇到限流自动降级到 Orchestrator 预搜索模式

### Level 2 · 内置工具（Fallback）

- **WebSearch**：Orchestrator 预搜索，结果作为上下文传入 Sub-agent
- **WebFetch**：抓取已知 URL 页面内容

### Level 3 · 辅助工具

- **gstack**（本地 skill）：Headless 浏览器截屏
- **dispatching-parallel-agents**（本地 skill）：并行调度

### 降级策略

MCP 不可用时，Orchestrator 切换到"预搜索+传上下文"模式：
1. Orchestrator 用 WebSearch 执行关键搜索
2. 将搜索结果作为上下文注入 Sub-agent prompt
3. Sub-agent 基于已有信息做结构化分析，不再自行搜索
4. 信息不足时标注"未获取（原因：XX）"

---

## 原则

- **先验证再行动**：对不确定的产品定位，必须先搜索验证，不可凭产品名猜测赛道和竞品
- **数据有源**：每个数字必须有信源，不可编造，无法确认时标注"未公开"
- **"搜不到"≠"不存在"**：搜索未返回时标注"未获取"，不做否定判断
- **竞品多源交叉验证**：至少2-3种搜索方式确认格局，不可只靠一次搜索下结论
- **Benchmark 批判性看待**：排名是技术验证而非用户选择理由，不作为竞争优势核心论据
- **派系壁垒**：渠道建议只推荐可行的路径，不可行的渠道（如百度产品进钉钉/飞书/抖音）直接不提，不需要特意标注"不行"
- **官网主页是必要信源**：抓取自身和竞品官网（含导航栏），但获取后按核心问题归类使用，不单独成段
- **境内外都看**：AI 竞争是全球的，境内外平台都要扫
- **可信度优先**：官方 > 权威媒体 > 社区观点，标注来源
- **合规**：不绕过登录/付费墙，尊重 robots.txt

---

## 参考文件索引

- `references/00-data-sources.md` — 境内外信息源地图、MCP 工具配置、合规
- `references/tavily-rate-limits.md` — Tavily API 限流策略与降级方案
- `references/01-product-positioning.md` — AI 赛道地图、竞品框架、事件追踪、投放方法论
- `references/02-user-growth.md` — 舆情方法论、口碑采集、AARRR、PLG 诊断、关键词矩阵指南
- `references/03-commercialization.md` — 渠道策略、商业化模式、定价框架、转化漏斗
- `references/04-ops-strategy.md` — 运营框架、阶段×重心映射、90 天计划
- `references/05-report-writing-guide.md` — 报告撰写规范（结构逻辑、分析深度、表达风格）
- `references/06-strategic-thinking.md` — 策略思考指引（专家追问清单、命题提炼方法、对抗性自检）
- `agents/` — 3+1 个 Agent 的 prompt 定义（产品定位、用户增长、商业化、质量审计）
- `templates/` — 4 份交付物模板

---
