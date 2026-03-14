# 47 - Agent 热更新与动态配置

> **核心问题**：生产环境的 Agent 能不能不重启就更新行为？

---

## 为什么需要热更新？

传统软件部署：改代码 → 编译 → 停服 → 重启 → 用户感知到中断。

Agent 的问题更严重：
- 重启会丢失所有**进行中的对话上下文**
- 用户正在等待 Agent 执行长任务，突然断了
- System Prompt 改一个字也要重启？代价太高

**热更新目标**：在不中断对话的前提下，动态更新 Agent 的：
1. System Prompt（行为指令）
2. 工具列表（新增/禁用工具）
3. 模型选择（临时切换到更强的模型）
4. 限流参数、Feature Flag

---

## 核心模式：配置驱动行为

把 Agent 行为从代码中分离，放到**可热更新的配置层**。

```
代码层（静态）          配置层（动态）
───────────────────    ─────────────────────
Agent Loop Engine   ←  system_prompt.txt
Tool Dispatcher     ←  tool_registry.json
Model Selector      ←  routing_rules.yaml
Rate Limiter        ←  limits.json
```

### 方案一：文件系统监听

最简单的实现 —— 用 `chokidar` 或 Python 的 `watchdog` 监听配置文件变化：

```typescript
// pi-mono 风格实现
import chokidar from 'chokidar';
import fs from 'fs';

class AgentConfig {
  private systemPrompt: string = '';
  private toolPolicy: Record<string, boolean> = {};

  constructor(private configPath: string) {
    this.load();
    this.watch();
  }

  private load() {
    const raw = JSON.parse(fs.readFileSync(this.configPath, 'utf8'));
    this.systemPrompt = raw.systemPrompt;
    this.toolPolicy = raw.toolPolicy ?? {};
    console.log('[config] 配置已加载');
  }

  private watch() {
    chokidar.watch(this.configPath).on('change', () => {
      console.log('[config] 检测到变化，热重载...');
      try {
        this.load();
      } catch (e) {
        console.error('[config] 加载失败，保持旧配置', e);
        // 关键：加载失败要回退，不能让 Agent 崩溃
      }
    });
  }

  getSystemPrompt() { return this.systemPrompt; }
  isToolAllowed(tool: string) { return this.toolPolicy[tool] !== false; }
}

// Agent Loop 每次对话读取最新配置
const config = new AgentConfig('./agent-config.json');

async function runAgent(userMessage: string) {
  const messages = [
    { role: 'system', content: config.getSystemPrompt() }, // 每次都读最新的
    { role: 'user', content: userMessage }
  ];
  // ...
}
```

---

## OpenClaw 的实现：SIGUSR1 信号

OpenClaw 使用操作系统信号量实现热更新，无需重启进程：

```bash
# 触发热重载（OpenClaw 内部）
kill -SIGUSR1 $(cat openclaw.pid)

# 或者通过 gateway 工具
gateway action=config.patch path="..." raw="..."
```

底层实现原理：

```javascript
// OpenClaw gateway 核心（简化版）
process.on('SIGUSR1', async () => {
  console.log('[gateway] 收到 SIGUSR1，重载配置...');
  
  try {
    const newConfig = await loadConfig();
    
    // 原子性替换配置对象
    Object.assign(currentConfig, newConfig);
    
    // 通知各模块配置已更新
    eventBus.emit('config:reloaded', newConfig);
    
    console.log('[gateway] 配置热重载完成');
  } catch (err) {
    console.error('[gateway] 热重载失败，保持原配置:', err);
    // NEVER crash the process on config error
  }
});
```

`config.patch` API：

```typescript
// 局部更新，不影响其他配置
await gateway.patch({
  path: 'agent.systemPrompt',
  value: '你是一个专门处理退款的客服 Agent...'
});

// 底层是 deep merge，不是全量替换
function deepPatch(target: any, path: string, value: any) {
  const keys = path.split('.');
  let obj = target;
  for (let i = 0; i < keys.length - 1; i++) {
    obj = obj[keys[i]] ??= {};
  }
  obj[keys[keys.length - 1]] = value;
}
```

---

## 方案二：数据库/KV 配置中心

生产环境推荐 —— 配置存数据库，多实例共享：

```python
# learn-claude-code 风格
import redis
import json
import threading
import time

class LiveConfig:
    def __init__(self, redis_url: str, agent_id: str):
        self.r = redis.from_url(redis_url)
        self.agent_id = agent_id
        self._cache = {}
        self._load()
        threading.Thread(target=self._poll, daemon=True).start()

    def _load(self):
        raw = self.r.get(f"agent:config:{self.agent_id}")
        if raw:
            self._cache = json.loads(raw)

    def _poll(self):
        """每 30 秒轮询一次（也可用 Redis pub/sub）"""
        while True:
            time.sleep(30)
            try:
                self._load()
            except Exception as e:
                print(f"[config] 轮询失败: {e}")

    def get(self, key: str, default=None):
        return self._cache.get(key, default)


# 使用
config = LiveConfig("redis://localhost:6379", "customer-service-agent")

def build_system_prompt():
    # 每次构建时读取最新值
    persona = config.get("persona", "你是一个通用助手")
    restrictions = config.get("restrictions", [])
    
    prompt = persona
    if restrictions:
        prompt += "\n\n限制：\n" + "\n".join(f"- {r}" for r in restrictions)
    return prompt
```

---

## Feature Flag：灰度更新 Agent 行为

不要一次性切换所有用户 —— 用 Feature Flag 做灰度：

```typescript
class AgentFeatureFlags {
  constructor(private config: LiveConfig) {}

  // 按用户 ID 灰度：10% 用户启用新行为
  isEnabled(flag: string, userId: string, rollout = 1.0): boolean {
    const flagConfig = this.config.get(`flags.${flag}`);
    if (!flagConfig) return false;
    if (flagConfig.enabled === false) return false;
    
    // 确定性哈希：同一用户始终得到相同结果
    const hash = this.stableHash(`${flag}:${userId}`);
    const pct = flagConfig.rollout ?? rollout;
    return (hash % 100) < (pct * 100);
  }

  private stableHash(str: string): number {
    let h = 0;
    for (const c of str) h = (h * 31 + c.charCodeAt(0)) & 0xffffffff;
    return Math.abs(h) % 100;
  }
}

// Agent Loop 中使用
async function selectModel(userId: string): Promise<string> {
  if (flags.isEnabled('use-claude-4', userId, 0.1)) {
    // 10% 用户用新模型
    return 'claude-opus-4-5';
  }
  return 'claude-sonnet-3-5';
}
```

---

## 动态工具注册

不仅配置可以热更新，工具也可以动态加载：

```typescript
class DynamicToolRegistry {
  private tools: Map<string, Tool> = new Map();

  register(name: string, tool: Tool) {
    this.tools.set(name, tool);
    console.log(`[tools] 已注册: ${name}`);
  }

  unregister(name: string) {
    this.tools.delete(name);
    console.log(`[tools] 已卸载: ${name}`);
  }

  // 给 LLM 的工具定义（每次调用都生成最新的）
  getSchemas(): ToolSchema[] {
    return [...this.tools.values()].map(t => t.schema);
  }

  async execute(name: string, params: any) {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`工具不存在: ${name}`);
    return tool.execute(params);
  }
}

const registry = new DynamicToolRegistry();

// 运行时添加工具（比如新部署了一个 MCP server）
registry.register('search_products', {
  schema: { name: 'search_products', description: '搜索商品', ... },
  execute: async ({ query }) => searchProductsAPI(query)
});

// 出问题了？立刻禁用，不用重启
registry.unregister('risky_tool');
```

---

## 安全：配置变更需要校验

热更新不等于随意更新 —— 必须有校验层：

```python
import jsonschema

CONFIG_SCHEMA = {
    "type": "object",
    "required": ["systemPrompt"],
    "properties": {
        "systemPrompt": {"type": "string", "maxLength": 10000},
        "model": {"type": "string", "enum": ["gpt-4o", "claude-sonnet-4-5"]},
        "maxTokens": {"type": "integer", "minimum": 100, "maximum": 8000},
        "toolPolicy": {
            "type": "object",
            "additionalProperties": {"type": "boolean"}
        }
    },
    "additionalProperties": False  # 禁止未知字段，防止注入
}

def safe_reload(new_config: dict) -> bool:
    try:
        jsonschema.validate(new_config, CONFIG_SCHEMA)
        return True
    except jsonschema.ValidationError as e:
        print(f"[config] 校验失败，拒绝更新: {e.message}")
        return False
```

---

## OpenClaw 实战：通过 `gateway` 工具热更新

在 OpenClaw 里，你可以直接对运行中的 Agent 做配置变更：

```bash
# 临时切换模型（不重启）
# 通过 gateway 工具发送配置补丁

# 查看当前配置
gateway action=config.get

# 局部更新
gateway action=config.patch path="agent.model" raw='"anthropic/claude-opus-4-5"'

# 触发重启（配置持久化后）
gateway action=restart note="模型切换完成"
```

这正是 OpenClaw 的核心设计：
- `config.patch` = 原子性局部更新
- `SIGUSR1` = 进程内热重载（不丢连接）
- `restart` = 最后手段，强制重启

---

## 总结

| 方案 | 实现复杂度 | 延迟 | 适用场景 |
|------|-----------|------|---------|
| 文件监听 | 低 | 秒级 | 单机开发/小规模 |
| 信号量 SIGUSR1 | 中 | 毫秒级 | 单进程生产 |
| Redis KV | 中 | 30s~分钟 | 多实例横向扩展 |
| Feature Flag | 高 | 即时 | 灰度发布 |

**核心原则**：
1. **配置与代码分离**：行为放配置，逻辑放代码
2. **更新失败要回滚**：绝不因为配置错误让 Agent 崩溃
3. **校验再更新**：热更新不等于跳过校验
4. **幂等性**：同一配置应用多次结果相同

> **下节预告**：Agent 多租户架构 —— 一套系统如何同时服务成千上万个不同租户的 Agent？
