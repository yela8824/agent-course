# Dynamic Tool Loading：动态工具加载

> 第 27 课 - 让 Agent 在运行时灵活装卸工具

## 为什么需要动态工具加载？

大多数 Agent 框架在启动时就固定了工具集。但生产环境中，我们经常需要：

1. **按需加载** - 不同用户/场景需要不同工具
2. **权限控制** - 根据用户权限动态启用工具
3. **资源优化** - 减少 context 中的工具定义开销
4. **热更新** - 不重启 Agent 就能添加新工具
5. **插件系统** - 支持第三方工具扩展

## 核心概念

### 1. Tool Registry（工具注册表）

所有工具的中央管理器：

```typescript
// pi-mono 风格的工具注册表
interface ToolRegistry {
  // 注册工具
  register(tool: Tool): void;
  
  // 注销工具
  unregister(toolName: string): void;
  
  // 获取当前可用工具
  getAvailableTools(context: ExecutionContext): Tool[];
  
  // 检查工具是否存在
  has(toolName: string): boolean;
}

class ToolRegistryImpl implements ToolRegistry {
  private tools = new Map<string, Tool>();
  private conditions = new Map<string, ToolCondition>();
  
  register(tool: Tool, condition?: ToolCondition) {
    this.tools.set(tool.name, tool);
    if (condition) {
      this.conditions.set(tool.name, condition);
    }
  }
  
  getAvailableTools(context: ExecutionContext): Tool[] {
    return Array.from(this.tools.values()).filter(tool => {
      const condition = this.conditions.get(tool.name);
      // 无条件 = 始终可用
      if (!condition) return true;
      // 检查条件
      return condition.check(context);
    });
  }
}
```

### 2. Tool Condition（工具条件）

决定工具何时可用：

```typescript
interface ToolCondition {
  check(context: ExecutionContext): boolean;
}

// 基于权限的条件
class PermissionCondition implements ToolCondition {
  constructor(private requiredPermission: string) {}
  
  check(context: ExecutionContext): boolean {
    return context.user.permissions.includes(this.requiredPermission);
  }
}

// 基于时间的条件
class TimeWindowCondition implements ToolCondition {
  constructor(
    private startHour: number,
    private endHour: number
  ) {}
  
  check(context: ExecutionContext): boolean {
    const hour = new Date().getHours();
    return hour >= this.startHour && hour < this.endHour;
  }
}

// 基于 channel 的条件
class ChannelCondition implements ToolCondition {
  constructor(private allowedChannels: string[]) {}
  
  check(context: ExecutionContext): boolean {
    return this.allowedChannels.includes(context.channel);
  }
}

// 组合条件
class CompositeCondition implements ToolCondition {
  constructor(
    private conditions: ToolCondition[],
    private operator: 'AND' | 'OR' = 'AND'
  ) {}
  
  check(context: ExecutionContext): boolean {
    if (this.operator === 'AND') {
      return this.conditions.every(c => c.check(context));
    }
    return this.conditions.some(c => c.check(context));
  }
}
```

### 3. 动态加载模式

#### 模式 A: Lazy Loading（延迟加载）

工具定义延迟到首次使用：

```typescript
class LazyToolLoader {
  private loadedTools = new Map<string, Tool>();
  private loaders = new Map<string, () => Promise<Tool>>();
  
  // 注册加载器，但不立即加载
  registerLoader(name: string, loader: () => Promise<Tool>) {
    this.loaders.set(name, loader);
  }
  
  async getTool(name: string): Promise<Tool | undefined> {
    // 已加载，直接返回
    if (this.loadedTools.has(name)) {
      return this.loadedTools.get(name);
    }
    
    // 有加载器，执行加载
    const loader = this.loaders.get(name);
    if (loader) {
      const tool = await loader();
      this.loadedTools.set(name, tool);
      return tool;
    }
    
    return undefined;
  }
}

// 使用示例
const loader = new LazyToolLoader();

// 注册加载器（不执行实际加载）
loader.registerLoader('database_query', async () => {
  // 只在首次调用时加载
  const { DatabaseTool } = await import('./tools/database');
  return new DatabaseTool(await getDbConnection());
});
```

#### 模式 B: Context-Based Loading（上下文加载）

根据对话上下文动态调整工具集：

```typescript
class ContextAwareToolLoader {
  private allTools: Tool[] = [];
  
  // 根据当前对话内容推断需要的工具
  getToolsForContext(messages: Message[]): Tool[] {
    const context = this.analyzeContext(messages);
    
    return this.allTools.filter(tool => {
      // 代码相关对话 -> 加载代码工具
      if (context.isCodeRelated && tool.category === 'code') {
        return true;
      }
      
      // 数据分析对话 -> 加载数据工具
      if (context.isDataRelated && tool.category === 'data') {
        return true;
      }
      
      // 基础工具始终可用
      if (tool.category === 'core') {
        return true;
      }
      
      return false;
    });
  }
  
  private analyzeContext(messages: Message[]): ContextAnalysis {
    const recentText = messages.slice(-5).map(m => m.content).join(' ');
    
    return {
      isCodeRelated: /code|function|bug|error|implement/i.test(recentText),
      isDataRelated: /data|query|database|analyze|report/i.test(recentText),
      // ... 更多分析
    };
  }
}
```

#### 模式 C: Plugin System（插件系统）

支持第三方工具扩展：

```typescript
interface ToolPlugin {
  name: string;
  version: string;
  tools: Tool[];
  
  // 生命周期钩子
  onLoad?(registry: ToolRegistry): Promise<void>;
  onUnload?(): Promise<void>;
}

class PluginManager {
  private plugins = new Map<string, ToolPlugin>();
  private registry: ToolRegistry;
  
  constructor(registry: ToolRegistry) {
    this.registry = registry;
  }
  
  async loadPlugin(plugin: ToolPlugin) {
    // 版本检查
    if (this.plugins.has(plugin.name)) {
      const existing = this.plugins.get(plugin.name)!;
      if (existing.version >= plugin.version) {
        console.log(`Plugin ${plugin.name} already loaded with same or newer version`);
        return;
      }
      // 卸载旧版本
      await this.unloadPlugin(plugin.name);
    }
    
    // 注册工具
    for (const tool of plugin.tools) {
      this.registry.register(tool);
    }
    
    // 调用生命周期钩子
    await plugin.onLoad?.(this.registry);
    
    this.plugins.set(plugin.name, plugin);
    console.log(`Loaded plugin: ${plugin.name}@${plugin.version}`);
  }
  
  async unloadPlugin(name: string) {
    const plugin = this.plugins.get(name);
    if (!plugin) return;
    
    // 卸载工具
    for (const tool of plugin.tools) {
      this.registry.unregister(tool.name);
    }
    
    // 调用生命周期钩子
    await plugin.onUnload?.();
    
    this.plugins.delete(name);
  }
  
  // 从文件系统加载插件
  async loadFromDirectory(dir: string) {
    const files = await fs.readdir(dir);
    
    for (const file of files) {
      if (file.endsWith('.plugin.js')) {
        const pluginModule = await import(path.join(dir, file));
        await this.loadPlugin(pluginModule.default);
      }
    }
  }
}
```

## OpenClaw 中的实现

OpenClaw 的工具加载由 Policy Pipeline 和 Skills 系统共同驱动：

```typescript
// 简化的 OpenClaw 工具加载流程
class OpenClawToolLoader {
  async getToolsForSession(session: Session): Promise<Tool[]> {
    const tools: Tool[] = [];
    
    // 1. 加载核心工具
    tools.push(...this.coreTools);
    
    // 2. 应用 policy 过滤
    const filteredTools = await this.policyPipeline.filter(
      tools,
      session.context
    );
    
    // 3. 加载适用的 skills
    const skills = await this.skillLoader.loadForSession(session);
    for (const skill of skills) {
      if (skill.tools) {
        tools.push(...skill.tools);
      }
    }
    
    // 4. 根据 channel 过滤
    return filteredTools.filter(tool => 
      this.isToolAvailableForChannel(tool, session.channel)
    );
  }
}
```

### Skills 作为动态工具

Skills 本质上是动态加载的工具集：

```yaml
# skills/github/SKILL.md
name: github
description: GitHub operations via `gh` CLI
tools:
  - gh_issue_create
  - gh_pr_list
  - gh_workflow_run
condition:
  require_cli: gh
  require_auth: github
```

```typescript
class SkillLoader {
  async loadSkill(skillPath: string): Promise<Skill | null> {
    const skillMd = await fs.readFile(
      path.join(skillPath, 'SKILL.md'),
      'utf-8'
    );
    
    const skill = this.parseSkillMd(skillMd);
    
    // 检查前置条件
    if (skill.condition?.require_cli) {
      const hasCliTool = await this.checkCliExists(skill.condition.require_cli);
      if (!hasCliTool) {
        console.log(`Skill ${skill.name} disabled: missing CLI ${skill.condition.require_cli}`);
        return null;
      }
    }
    
    if (skill.condition?.require_auth) {
      const hasAuth = await this.checkAuth(skill.condition.require_auth);
      if (!hasAuth) {
        console.log(`Skill ${skill.name} disabled: missing auth ${skill.condition.require_auth}`);
        return null;
      }
    }
    
    return skill;
  }
}
```

## 工具 Schema 动态生成

有时工具的 schema 本身也需要动态生成：

```typescript
// 根据数据库 schema 动态生成查询工具
async function generateDatabaseTool(connection: DbConnection): Promise<Tool> {
  // 获取表结构
  const tables = await connection.query(`
    SELECT table_name, column_name, data_type 
    FROM information_schema.columns 
    WHERE table_schema = 'public'
  `);
  
  // 构建 enum 约束
  const tableNames = [...new Set(tables.map(t => t.table_name))];
  
  return {
    name: 'database_query',
    description: `Query the database. Available tables: ${tableNames.join(', ')}`,
    parameters: {
      type: 'object',
      properties: {
        table: {
          type: 'string',
          enum: tableNames,  // 动态生成的枚举
          description: 'Table to query'
        },
        columns: {
          type: 'array',
          items: { type: 'string' },
          description: 'Columns to select'
        },
        where: {
          type: 'string',
          description: 'WHERE clause'
        }
      },
      required: ['table']
    },
    execute: async (params) => {
      // 执行查询...
    }
  };
}
```

## 实战技巧

### 1. Tool Discovery（工具发现）

让 Agent 知道有哪些工具可以请求加载：

```typescript
// 提供工具目录
const toolCatalog = {
  name: 'tool_catalog',
  description: 'List available tools that can be loaded',
  parameters: {},
  execute: async () => {
    return {
      available_tools: [
        { name: 'github', description: 'GitHub operations', status: 'not_loaded' },
        { name: 'database', description: 'Database queries', status: 'loaded' },
        { name: 'email', description: 'Send emails', status: 'not_loaded' },
      ]
    };
  }
};

// 请求加载工具
const requestTool = {
  name: 'request_tool',
  description: 'Request to load an additional tool',
  parameters: {
    type: 'object',
    properties: {
      tool_name: { type: 'string' }
    },
    required: ['tool_name']
  },
  execute: async ({ tool_name }, context) => {
    const result = await context.toolLoader.loadTool(tool_name);
    return { 
      success: result.success,
      message: result.success 
        ? `Tool ${tool_name} is now available`
        : `Failed to load ${tool_name}: ${result.error}`
    };
  }
};
```

### 2. Graceful Degradation（优雅降级）

工具不可用时的处理：

```typescript
class ResilientToolExecutor {
  async execute(toolName: string, params: any): Promise<ToolResult> {
    const tool = this.registry.getTool(toolName);
    
    if (!tool) {
      // 尝试动态加载
      const loaded = await this.tryLoadTool(toolName);
      if (!loaded) {
        return {
          success: false,
          error: `Tool ${toolName} is not available`,
          suggestion: this.suggestAlternative(toolName)
        };
      }
    }
    
    try {
      return await tool.execute(params);
    } catch (error) {
      // 工具执行失败，尝试降级
      const fallback = this.getFallbackTool(toolName);
      if (fallback) {
        return await fallback.execute(params);
      }
      throw error;
    }
  }
  
  private suggestAlternative(toolName: string): string {
    const alternatives: Record<string, string> = {
      'database_query': 'Use read tool to read CSV exports instead',
      'github_api': 'Use exec tool with gh CLI',
      'send_email': 'Use exec tool with mail command',
    };
    return alternatives[toolName] || 'No alternative available';
  }
}
```

### 3. Tool Versioning（工具版本）

处理工具升级：

```typescript
interface VersionedTool extends Tool {
  version: string;
  deprecated?: boolean;
  deprecationMessage?: string;
  migrateParams?: (oldParams: any) => any;
}

class VersionedToolRegistry {
  private tools = new Map<string, VersionedTool[]>();
  
  register(tool: VersionedTool) {
    const versions = this.tools.get(tool.name) || [];
    versions.push(tool);
    // 按版本排序
    versions.sort((a, b) => this.compareVersions(b.version, a.version));
    this.tools.set(tool.name, versions);
  }
  
  getTool(name: string, requestedVersion?: string): VersionedTool | undefined {
    const versions = this.tools.get(name);
    if (!versions?.length) return undefined;
    
    if (requestedVersion) {
      return versions.find(t => t.version === requestedVersion);
    }
    
    // 返回最新的非 deprecated 版本
    const active = versions.find(t => !t.deprecated);
    if (active) return active;
    
    // 如果都 deprecated 了，返回最新的并警告
    const latest = versions[0];
    console.warn(`Tool ${name} is deprecated: ${latest.deprecationMessage}`);
    return latest;
  }
}
```

## 性能考虑

### Token 预算

动态工具加载的一个重要考虑是 token 成本：

```typescript
class TokenAwareToolLoader {
  private readonly maxToolTokens = 4000; // 工具定义的 token 上限
  
  getToolsWithinBudget(
    availableTools: Tool[],
    context: ExecutionContext
  ): Tool[] {
    let currentTokens = 0;
    const selectedTools: Tool[] = [];
    
    // 按优先级排序
    const sortedTools = this.prioritizeTools(availableTools, context);
    
    for (const tool of sortedTools) {
      const toolTokens = this.estimateTokens(tool);
      
      if (currentTokens + toolTokens <= this.maxToolTokens) {
        selectedTools.push(tool);
        currentTokens += toolTokens;
      }
    }
    
    return selectedTools;
  }
  
  private prioritizeTools(tools: Tool[], context: ExecutionContext): Tool[] {
    return tools.sort((a, b) => {
      // 核心工具优先
      if (a.category === 'core' && b.category !== 'core') return -1;
      // 最近使用的工具优先
      const aLastUsed = context.toolUsage.get(a.name) || 0;
      const bLastUsed = context.toolUsage.get(b.name) || 0;
      return bLastUsed - aLastUsed;
    });
  }
}
```

## 总结

动态工具加载让 Agent 更加灵活和高效：

| 模式 | 优点 | 适用场景 |
|------|------|----------|
| Lazy Loading | 减少启动时间 | 工具初始化开销大 |
| Context-Based | 智能推断 | 多领域 Agent |
| Plugin System | 可扩展性强 | 开放平台 |
| Conditional | 精细控制 | 多租户/权限系统 |

核心原则：
1. **最小权限** - 只加载需要的工具
2. **优雅降级** - 工具不可用时有备选方案
3. **Token 预算** - 控制工具定义的开销
4. **版本管理** - 支持工具平滑升级

下节课我们讲 **Agent Reflection（反思机制）** - 让 Agent 学会自我审视和改进。

---

*思考题：如果一个工具在执行过程中突然不可用（比如 API key 过期），Agent 应该如何处理？*
