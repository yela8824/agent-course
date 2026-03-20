# 98 - Agent 插件化架构（Plugin Architecture）

> 核心问题：Agent 的工具集是静态写死的，还是可以像 VS Code 插件一样动态扩展？

---

## 为什么需要插件架构？

写死工具的问题很明显：

- **不可扩展**：加个新工具要改核心代码
- **不可复用**：同一套工具逻辑在多个 Agent 里各写一份
- **不可卸载**：某个工具出 bug，只能停机修
- **权限混乱**：所有工具都在同一权限层，无法沙箱隔离

插件架构解决这些问题：**Agent 核心只提供能力框架，具体工具/行为以插件形式注入。**

---

## 真实案例：OpenClaw 的 Skills 系统

OpenClaw 就是这个模式的绝佳例子。每个 Skill 是一个独立目录：

```
~/.openclaw/
  skills/
    weather/
      SKILL.md        ← 插件声明：何时使用、如何使用
      scripts/
        get-weather.sh
    github/
      SKILL.md
      references/
        api-patterns.md
    mysterybox/        ← 用户自定义插件！
      SKILL.md
```

Agent 启动时扫描所有 Skill 目录，**动态构建工具列表**。用户问天气 → 加载 weather skill → 按 SKILL.md 指引执行。

这就是插件架构：**主程序不知道具体插件，插件不知道彼此。**

---

## pi-mono 的实现：TypeScript 工具注册表

pi-mono 用显式注册表模式管理工具插件：

```typescript
// pi-mono/src/tools/registry.ts
type ToolPlugin = {
  name: string
  description: string
  schema: JSONSchema
  execute: (params: unknown, ctx: ToolContext) => Promise<ToolResult>
  // 插件元数据
  category?: string
  requiresAuth?: boolean
  rateLimit?: { rpm: number }
}

class ToolRegistry {
  private tools = new Map<string, ToolPlugin>()

  register(plugin: ToolPlugin) {
    if (this.tools.has(plugin.name)) {
      throw new Error(`Tool ${plugin.name} already registered`)
    }
    this.tools.set(plugin.name, plugin)
    console.log(`[Registry] Loaded plugin: ${plugin.name}`)
  }

  unregister(name: string) {
    this.tools.delete(name)
  }

  getAll(): ToolPlugin[] {
    return [...this.tools.values()]
  }

  get(name: string): ToolPlugin | undefined {
    return this.tools.get(name)
  }

  // 生成 LLM 的 tools 数组
  toAnthropicTools(): Anthropic.Tool[] {
    return this.getAll().map(p => ({
      name: p.name,
      description: p.description,
      input_schema: p.schema,
    }))
  }
}

export const registry = new ToolRegistry()
```

**插件文件只需 export 一个对象然后注册自己：**

```typescript
// pi-mono/src/tools/plugins/web-search.ts
import { registry } from '../registry'

registry.register({
  name: 'web_search',
  description: 'Search the web for current information',
  schema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Search query' },
      count: { type: 'number', default: 5 },
    },
    required: ['query'],
  },
  category: 'search',
  rateLimit: { rpm: 60 },

  async execute({ query, count = 5 }, ctx) {
    // 实现搜索逻辑
    const results = await brave.search(query, count)
    return { type: 'text', text: formatResults(results) }
  },
})
```

---

## 动态加载：文件系统扫描

不想手动 import 每个插件？用动态扫描：

```typescript
// pi-mono/src/tools/loader.ts
import { readdir } from 'fs/promises'
import { join } from 'path'

async function loadPlugins(pluginDir: string) {
  const files = await readdir(pluginDir)
  const results = { loaded: 0, failed: 0, errors: [] as string[] }

  for (const file of files) {
    if (!file.endsWith('.ts') && !file.endsWith('.js')) continue

    try {
      // 动态 import 触发文件内的 registry.register()
      await import(join(pluginDir, file))
      results.loaded++
    } catch (err) {
      results.failed++
      results.errors.push(`${file}: ${err.message}`)
      // 插件加载失败 → 不影响其他插件和主程序
      console.error(`[Loader] Failed to load ${file}:`, err)
    }
  }

  console.log(`[Loader] ${results.loaded} plugins loaded, ${results.failed} failed`)
  return results
}

// 启动时调用
await loadPlugins('./tools/plugins')
await loadPlugins('./tools/user-plugins')  // 用户自定义插件目录
```

**关键设计：一个插件崩溃不影响其他插件。**

---

## 插件生命周期管理

```typescript
// 插件可以声明 lifecycle hooks
type ToolPlugin = {
  // ... 基础字段 ...
  
  // 生命周期
  onLoad?: () => Promise<void>      // 插件初始化（建立连接、预热缓存）
  onUnload?: () => Promise<void>    // 插件卸载（释放资源）
  healthCheck?: () => Promise<boolean>  // 健康检查
}

class ToolRegistry {
  async register(plugin: ToolPlugin) {
    // 初始化
    if (plugin.onLoad) {
      await plugin.onLoad()
    }
    this.tools.set(plugin.name, plugin)
  }

  async unregister(name: string) {
    const plugin = this.tools.get(name)
    if (plugin?.onUnload) {
      await plugin.onUnload()  // 优雅关闭
    }
    this.tools.delete(name)
  }

  // 热更新：卸载旧版本，加载新版本
  async reload(name: string, newModule: string) {
    await this.unregister(name)
    await import(newModule + '?t=' + Date.now())  // 清除模块缓存
  }
}
```

---

## 权限沙箱：插件级别的访问控制

不同插件应该有不同权限：

```typescript
type PluginPermissions = {
  network?: boolean          // 能否访问网络
  filesystem?: string[]      // 允许访问的路径列表
  env?: string[]             // 允许读取的环境变量
  maxExecutionMs?: number    // 最大执行时间
}

class SecureToolRegistry extends ToolRegistry {
  async executeWithSandbox(
    name: string,
    params: unknown,
    ctx: ToolContext
  ): Promise<ToolResult> {
    const plugin = this.get(name)
    if (!plugin) throw new Error(`Tool ${name} not found`)

    const perms = plugin.permissions ?? {}

    // 注入受限 context
    const sandboxedCtx: ToolContext = {
      ...ctx,
      // 如果没有网络权限，替换 fetch
      fetch: perms.network
        ? ctx.fetch
        : () => Promise.reject(new Error(`Tool ${name} has no network permission`)),
      // 文件访问白名单
      readFile: (path) => {
        const allowed = perms.filesystem ?? []
        if (!allowed.some(dir => path.startsWith(dir))) {
          throw new Error(`Access denied: ${path}`)
        }
        return ctx.readFile(path)
      },
    }

    // 执行超时控制
    const timeout = perms.maxExecutionMs ?? 30_000
    return await Promise.race([
      plugin.execute(params, sandboxedCtx),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error(`Tool ${name} timeout`)), timeout)
      ),
    ])
  }
}
```

---

## learn-claude-code 视角：Skills 作为一等公民

在 learn-claude-code（Python 实现）里，技能加载是这样的：

```python
# learn-claude-code/agent/skill_loader.py
import importlib
import pkgutil
from pathlib import Path

class SkillLoader:
    def __init__(self, skills_dir: str):
        self.skills_dir = Path(skills_dir)
        self.skills: dict[str, "Skill"] = {}
    
    def discover(self) -> list[str]:
        """扫描技能目录，返回找到的技能名列表"""
        found = []
        for path in self.skills_dir.glob("*/SKILL.md"):
            skill_name = path.parent.name
            found.append(skill_name)
        return found
    
    def load(self, name: str) -> "Skill":
        """按需加载技能（懒加载）"""
        if name in self.skills:
            return self.skills[name]
        
        skill_path = self.skills_dir / name / "SKILL.md"
        if not skill_path.exists():
            raise ValueError(f"Skill {name} not found")
        
        skill = Skill(name=name, path=skill_path)
        skill.load()
        self.skills[name] = skill
        return skill
    
    def match(self, user_intent: str) -> "Skill | None":
        """根据用户意图匹配最合适的技能"""
        candidates = []
        for name in self.discover():
            skill = self.load(name)
            score = skill.relevance_score(user_intent)
            if score > 0.7:  # 阈值
                candidates.append((score, skill))
        
        if not candidates:
            return None
        
        return max(candidates, key=lambda x: x[0])[1]
```

**这正是 OpenClaw 的实际工作原理**：系统提示里列出所有 Skill 的 description，让 LLM 自己判断是否需要加载某个 Skill 的 SKILL.md，然后按指引执行。

---

## 插件通信：插件间怎么互相调用？

插件不应该直接相互依赖（会造成耦合），而是通过事件总线：

```typescript
// 插件间通信：事件总线
class PluginEventBus {
  private handlers = new Map<string, Set<Function>>()

  on(event: string, handler: Function) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set())
    }
    this.handlers.get(event)!.add(handler)
  }

  emit(event: string, data: unknown) {
    const handlers = this.handlers.get(event) ?? new Set()
    for (const handler of handlers) {
      handler(data).catch(console.error)  // 单个 handler 失败不影响其他
    }
  }
}

const bus = new PluginEventBus()

// 搜索插件发布结果
registry.register({
  name: 'web_search',
  async execute(params) {
    const results = await doSearch(params.query)
    bus.emit('search:complete', { query: params.query, results })
    return { type: 'text', text: formatResults(results) }
  },
})

// 缓存插件监听并缓存搜索结果
bus.on('search:complete', async ({ query, results }) => {
  await cache.set(`search:${query}`, results, 3600)
})
```

---

## 实战：给 Agent 加一个自定义插件

假设你想给 pi-mono 加一个查股价的工具：

```typescript
// my-plugins/stock-price.ts
import { registry } from '../tools/registry'
import axios from 'axios'

registry.register({
  name: 'get_stock_price',
  description: 'Get real-time stock price for a given ticker symbol',
  schema: {
    type: 'object',
    properties: {
      ticker: { type: 'string', description: 'Stock ticker, e.g. AAPL, TSLA' },
    },
    required: ['ticker'],
  },
  permissions: {
    network: true,
    maxExecutionMs: 5000,
  },

  async onLoad() {
    console.log('[StockPlugin] Initialized')
    // 可以在这里预热 HTTP 连接
  },

  async execute({ ticker }) {
    const { data } = await axios.get(
      `https://query1.finance.yahoo.com/v8/finance/chart/${ticker}`
    )
    const price = data.chart.result[0].meta.regularMarketPrice
    return {
      type: 'text',
      text: `${ticker}: $${price}`,
    }
  },
})
```

然后在 loader 里加一行：
```typescript
await loadPlugins('./my-plugins')  // 就这一行，新工具生效
```

---

## 核心设计原则总结

| 原则 | 实现方式 |
|------|---------|
| **开闭原则** | 主程序对扩展开放，对修改关闭 |
| **单一职责** | 每个插件只做一件事 |
| **失败隔离** | 插件崩溃不影响主程序 |
| **懒加载** | 按需加载，降低启动时间 |
| **权限最小化** | 每个插件只有它需要的权限 |
| **生命周期** | onLoad/onUnload 管理资源 |

---

## 一句话总结

> **插件架构让 Agent 从"功能固定的工具"变成"可扩展的平台"——OpenClaw 的 Skills、pi-mono 的 ToolRegistry，都是这个思想的落地实现。写插件而不是改主程序，这才是可维护 Agent 的正确姿势。**
