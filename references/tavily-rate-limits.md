# Tavily API 限流策略

> 本文档定义 Tavily MCP 工具的使用限流规则，确保在开发环境中稳定运行。

---

## 限流规则（官方）

| 环境 | 搜索端点 RPM | Crawl 端点 RPM | Research 端点 RPM |
|------|-------------|--------------|-----------------|
| **Development** | 100 | 100 | 20 |
| **Production** | 1,000 | 100 | 20 |

> RPM = Requests Per Minute（每分钟请求数）

---

## 当前问题诊断

### 问题表现
- 错误码：`400 [1210] API 调用参数有误`
- 实际原因：触发 Tavily API 限流

### 问题场景
- **并发调用**：同时发起 2+ 次 tavily_search 可能触发限流
- **高频调用**：1 分钟内多次调用累积超限
- **历史案例**：54 分钟内 16 次调用，前 14 次成功，后 2 次并发调用失败

---

## 工程方案修改

### 原则

1. **串行化调用**：Agent 内所有 tavily_search 按顺序执行，不并发
2. **控制密度**：单 Agent 内 tavily_search 调用间隔 ≥ 2 秒
3. **全局限流**：全流程 tavily_search 总次数控制在限流范围内
4. **优雅降级**：触发限流时自动切换到降级方案

---

## 各 Agent 限流策略

### Agent 1 · 产品定位与竞争态势

**Tavily 使用场景**：品牌提及、投放痕迹搜索

| 阶段 | 调用次数 | 间隔 | 合计耗时 |
|------|---------|------|---------|
| 竞品投放追踪 | 3-5 次 | 2 秒 | 6-10 秒 |
| 品牌声量扫描 | 2-3 次 | 2 秒 | 4-6 秒 |
| **单 Agent 总计** | **≤8 次** | - | **≤16 秒** |

**优化措施**：
- 合并相似搜索（如 `"品牌名" site:x.com` + `"品牌名" 投放` 合并为一次）
- 优先用 Exa 搜索融资/公司信息，减少 Tavily 调用

---

### Agent 2 · 用户分析与增长诊断

**Tavily 使用场景**：舆情口碑、用户评价搜索（主力）

| 阶段 | 调用次数 | 间隔 | 合计耗时 |
|------|---------|------|---------|
| 境内社媒搜索（小红书/知乎/V2EX） | 3-4 次 | 2 秒 | 6-8 秒 |
| 境外社媒搜索（Reddit/Discord） | 3-4 次 | 2 秒 | 6-8 秒 |
| 应用商店评论搜索 | 1-2 次 | 2 秒 | 2-4 秒 |
| **单 Agent 总计** | **≤10 次** | - | **≤20 秒** |

**优化措施**：
- 按平台分组搜索，合并同类查询
- 优先用 Firecrawl 爬取已知 URL 的评论页，减少搜索调用

---

### Agent 3 · 商业化诊断

**Tavily 使用场景**：商业化讨论、定价对比

| 阶段 | 调用次数 | 间隔 | 合计耗时 |
|------|---------|------|---------|
| 商业化模式讨论 | 2-3 次 | 2 秒 | 4-6 秒 |
| 定价对比讨论 | 2-3 次 | 2 秒 | 4-6 秒 |
| **单 Agent 总计** | **≤6 次** | - | **≤12 秒** |

**优化措施**：
- 优先用 Firecrawl 直接爬取 /pricing 页面
- 仅当无法直接获取时才用 Tavily 搜索讨论

---

## 全流程限流预算

### 并发执行场景（Phase 2）

```
Phase 2: Agent 2 + Agent 3 并行
├─ Agent 2: ≤10 次调用，耗时 ≤20 秒
└─ Agent 3: ≤6 次调用，耗时 ≤12 秒
    → 并发峰值：16 次/分钟（在安全范围内）
```

### 串行执行场景（Phase 1）

```
Phase 1: Agent 1 顺序执行
└─ Agent 1: ≤8 次调用，耗时 ≤16 秒
    → 无并发风险
```

### 全流程总预算

| Phase | Agent | Tavily 调用 | 累积耗时 |
|-------|-------|------------|---------|
| Phase 0 | Orchestrator | 2-3 次 | ≤6 秒 |
| Phase 1 | Agent 1 | ≤8 次 | ≤16 秒 |
| Phase 2 | Agent 2 | ≤10 次 | ≤20 秒 |
| Phase 2 | Agent 3 | ≤6 次 | ≤12 秒 |
| **全流程总计** | - | **≤27 次** | **≤54 秒** |

> 在 100 RPM 限流下，全流程耗时 < 1 分钟，完全安全。

---

## 降级策略

### 触发条件

当 Tavily 返回以下任一错误时：
- `429 Too Many Requests`
- `400 [1210] API 调用参数有误`（可能是限流导致）
- 连续 3 次超时

### 降级方案

```
1. Agent 检测到限流错误
2. 自动暂停 Tavily 调用 30 秒
3. 重试 1 次，仍失败则切换到降级模式
4. 降级模式：
   - 基于已获取的搜索结果进行分析
   - 明确标注"数据来源：降级搜索（受 Tavily 限流影响）"
   - 信息不足时标注"未获取（原因：Tavily 限流）"
```

---

## Agent 实现要求

### 代码层约束

```javascript
// 伪代码示例
const tavilySearchWithRateLimit = async (query, agentContext) => {
  const { lastTavilyCallTime } = agentContext;

  // 1. 检查限流：距上次调用需 ≥2 秒
  const now = Date.now();
  if (lastTavilyCallTime && now - lastTavilyCallTime < 2000) {
    await sleep(2000 - (now - lastTavilyCallTime));
  }

  // 2. 执行搜索
  try {
    const result = await tavilySearch(query);
    agentContext.lastTavilyCallTime = Date.now();
    return result;
  } catch (error) {
    // 3. 限流错误处理
    if (error.code === 429 || error.message.includes('1210')) {
      await sleep(30000); // 30 秒冷却
      return tavilySearch(query); // 重试一次
    }
    throw error;
  }
};
```

### Prompt 层约束

在每个 Agent 的 prompt 中添加：

```
## Tavily 限流要求（重要）

1. 所有 tavily_search 调用必须串行执行，不可并发
2. 每次调用间隔 ≥ 2 秒
3. 单 Agent 内 tavily_search 调用次数限制：
   - Agent 1: ≤8 次
   - Agent 2: ≤10 次
   - Agent 3: ≤6 次
4. 遇到 429 或 400[1210] 错误时：
   - 暂停 30 秒
   - 重试 1 次
   - 仍失败则切换到降级模式
```

---

## 监控与预警

### 运行时指标

- 每个触发 Tavily 调用的工具调用
- 调用间隔时间
- 是否触发限流
- 降级模式触发次数

### 日志格式

```
[Tavily] Call #1: query="小米MiMo 评价", interval=0ms
[Tavily] Call #2: query="MiMo API 定价", interval=2005ms ✓
[Tavily] Call #3: query="MiMo vs DeepSeek", interval=2010ms ✓
[Tavily] Rate limited: 400 [1210], waiting 30s...
[Tavily] Retry after 30s: success ✓
```

---

## 配置检查清单

- [ ] 确认 Tavily API Key 类型（Development vs Production）
- [ ] 开发环境预期：100 RPM，Production：1000 RPM
- [ ] Agent prompt 已添加限流约束
- [ ] Phase 2 并发调用总次数 ≤ 16 次/分钟
- [ ] 降级策略已实现
- [ ] 日志监控已开启

---

## 参考资料

- Tavily 官方文档：https://docs.tavily.com/documentation/rate-limits
- 本项目 SKILL.md：AI 产品运营研究工作流
- 本项目 references/00-data-sources.md：数据源地图