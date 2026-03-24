# 126 - Agent 声明式工具注解（Declarative Tool Annotations & Decorator Pattern）

> **一句话总结：** 用 TypeScript 装饰器把权限、缓存、重试、超时元数据"写在工具头上"，运行时自动应用，消灭重复样板代码，让工具定义即文档。

---

## 为什么需要声明式注解？

传统做法：每个工具函数里手动写权限检查、try/catch 重试、缓存逻辑……

```typescript
// ❌ 到处都是样板代码
async function readFile(path: string) {
  // 权限检查
  if (!hasPermission('file:read')) throw new Error('forbidden');
  // 缓存检查
  const cached = cache.get(`readFile:${path}`);
  if (cached) return cached;
  // 重试逻辑
  for (let i = 0; i < 3; i++) {
    try {
      const result = await fs.readFile(path);
      cache.set(`readFile:${path}`, result, 60);
      return result;
    } catch (e) {
      if (i === 2) throw e;
      await sleep(1000 * 2 ** i);
    }
  }
}
```

声明式做法：把这些关切分离出来，用注解声明意图：

```typescript
// ✅ 意图清晰，零样板
@permission('file:read')
@cache({ ttl: 60 })
@retry({ maxAttempts: 3, backoff: 'exponential' })
async function readFile(path: string) {
  return fs.readFile(path);
}
```

---

## 核心概念：装饰器工厂

TypeScript 装饰器本质上是高阶函数，接受目标函数并返回包装后的版本。

```typescript
// 装饰器类型定义
type MethodDecorator = (
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) => PropertyDescriptor | void;

// 装饰器工厂 = 返回装饰器的函数
function cache(options: { ttl: number }) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    const store = new Map<string, { value: any; expires: number }>();

    descriptor.value = async function (...args: any[]) {
      const cacheKey = `${key}:${JSON.stringify(args)}`;
      const now = Date.now();
      const hit = store.get(cacheKey);
      
      if (hit && hit.expires > now) {
        console.log(`[cache] HIT ${cacheKey}`);
        return hit.value;
      }
      
      const result = await original.apply(this, args);
      store.set(cacheKey, { value: result, expires: now + options.ttl * 1000 });
      return result;
    };

    return descriptor;
  };
}
```

---

## 实战：构建完整的工具注解系统

### 1. 元数据注册表

```typescript
// tool-registry.ts
import 'reflect-metadata';

export const TOOL_META_KEY = Symbol('tool:meta');

export interface ToolMeta {
  name: string;
  description: string;
  permissions: string[];
  cache?: { ttl: number; key?: (...args: any[]) => string };
  retry?: { maxAttempts: number; backoff: 'fixed' | 'exponential'; delayMs: number };
  timeout?: number;
  rateLimit?: { maxPerMinute: number };
  idempotent?: boolean;
}

// 注册所有带注解的工具
const toolRegistry = new Map<string, { fn: Function; meta: ToolMeta }>();

export function getRegisteredTools() {
  return Array.from(toolRegistry.values());
}
```

### 2. 核心装饰器集合

```typescript
// decorators.ts
import 'reflect-metadata';

// ─── @tool：标记这是一个 Agent 工具 ───
export function tool(name: string, description: string) {
  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const existing = Reflect.getMetadata(TOOL_META_KEY, target, key) || {};
    Reflect.defineMetadata(TOOL_META_KEY, { ...existing, name, description }, target, key);
    // 自动注册到工具列表
    toolRegistry.set(name, { fn: descriptor.value, meta: { ...existing, name, description } });
    return descriptor;
  };
}

// ─── @permission：权限要求 ───
export function permission(...perms: string[]) {
  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;

    descriptor.value = async function (this: any, ...args: any[]) {
      const session = this.session || getCurrentSession();
      const missing = perms.filter(p => !session.hasPermission(p));
      if (missing.length > 0) {
        throw new PermissionError(`Missing permissions: ${missing.join(', ')}`);
      }
      return original.apply(this, args);
    };

    // 保存元数据供审计用
    const existing = Reflect.getMetadata(TOOL_META_KEY, target, key) || {};
    Reflect.defineMetadata(TOOL_META_KEY, { ...existing, permissions: perms }, target, key);
    return descriptor;
  };
}

// ─── @retry：自动重试 ───
export function retry(options: { maxAttempts?: number; backoff?: 'fixed' | 'exponential'; delayMs?: number } = {}) {
  const { maxAttempts = 3, backoff = 'exponential', delayMs = 500 } = options;

  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      let lastError: Error;
      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          return await original.apply(this, args);
        } catch (err: any) {
          lastError = err;
          if (attempt === maxAttempts) break;
          
          const wait = backoff === 'exponential'
            ? delayMs * Math.pow(2, attempt - 1)
            : delayMs;
          
          console.warn(`[retry] ${key} attempt ${attempt}/${maxAttempts} failed, retrying in ${wait}ms...`);
          await new Promise(r => setTimeout(r, wait));
        }
      }
      throw lastError!;
    };

    return descriptor;
  };
}

// ─── @timeout：超时控制 ───
export function timeout(ms: number) {
  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const timer = new Promise<never>((_, reject) =>
        setTimeout(() => reject(new TimeoutError(`${key} timed out after ${ms}ms`)), ms)
      );
      return Promise.race([original.apply(this, args), timer]);
    };

    return descriptor;
  };
}

// ─── @rateLimit：速率限制 ───
export function rateLimit(maxPerMinute: number) {
  const window: number[] = [];

  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const now = Date.now();
      // 清理一分钟前的记录
      while (window.length > 0 && window[0] < now - 60_000) window.shift();

      if (window.length >= maxPerMinute) {
        const waitMs = window[0] + 60_000 - now;
        throw new RateLimitError(`${key} rate limit exceeded, retry in ${waitMs}ms`);
      }

      window.push(now);
      return original.apply(this, args);
    };

    return descriptor;
  };
}

// ─── @idempotent：幂等保障（基于请求 hash 去重）───
export function idempotent(ttlMs: number = 5000) {
  const inflight = new Map<string, Promise<any>>();

  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const hash = stableHash(args);
      if (inflight.has(hash)) {
        console.log(`[idempotent] ${key} dedup hit, reusing inflight`);
        return inflight.get(hash);
      }

      const promise = original.apply(this, args).finally(() => {
        setTimeout(() => inflight.delete(hash), ttlMs);
      });
      inflight.set(hash, promise);
      return promise;
    };

    return descriptor;
  };
}
```

### 3. 在 Agent 工具类中组合使用

```typescript
// tools/file-tools.ts
class FileTools {
  constructor(private session: AgentSession) {}

  @tool('read_file', '读取文件内容')
  @permission('file:read')
  @cache({ ttl: 30 })
  @retry({ maxAttempts: 3 })
  @timeout(5000)
  async readFile(path: string): Promise<string> {
    return fs.promises.readFile(path, 'utf-8');
  }

  @tool('write_file', '写入文件内容')
  @permission('file:write')
  @retry({ maxAttempts: 2, backoff: 'fixed', delayMs: 200 })
  @timeout(10000)
  async writeFile(path: string, content: string): Promise<void> {
    await fs.promises.writeFile(path, content, 'utf-8');
  }

  @tool('search_files', '在文件中搜索内容')
  @permission('file:read')
  @rateLimit(30)   // 每分钟最多 30 次
  @cache({ ttl: 10 })
  @timeout(15000)
  async searchFiles(pattern: string, dir: string): Promise<string[]> {
    return globSearch(pattern, dir);
  }
}

// tools/web-tools.ts
class WebTools {
  @tool('fetch_url', '获取网页内容')
  @permission('network:fetch')
  @idempotent(3000)  // 3秒内相同 URL 去重
  @retry({ maxAttempts: 3, backoff: 'exponential', delayMs: 1000 })
  @timeout(30000)
  @rateLimit(20)
  async fetchUrl(url: string): Promise<string> {
    const res = await fetch(url);
    return res.text();
  }
}
```

### 4. 从注解自动生成 LLM 工具 Schema

```typescript
// schema-generator.ts
export function generateToolSchema(toolClass: any): ClaudeToolDefinition[] {
  const instance = new toolClass();
  const methods = Object.getOwnPropertyNames(toolClass.prototype);

  return methods
    .filter(method => method !== 'constructor')
    .map(method => {
      const meta: ToolMeta = Reflect.getMetadata(TOOL_META_KEY, instance, method);
      if (!meta?.name) return null;

      // 从 TypeScript 类型反射生成参数 Schema（需 reflect-metadata）
      const paramTypes = Reflect.getMetadata('design:paramtypes', instance, method) || [];
      
      return {
        name: meta.name,
        description: meta.description,
        // 自动附加权限说明
        ...(meta.permissions.length > 0 && {
          description: `${meta.description} [需要权限: ${meta.permissions.join(', ')}]`
        }),
        input_schema: buildSchema(paramTypes),
      };
    })
    .filter(Boolean);
}

// 在 Agent 初始化时自动注册所有工具
async function buildAgentTools(session: AgentSession) {
  const fileTools = new FileTools(session);
  const webTools = new WebTools(session);
  
  return [
    ...generateToolSchema(FileTools),
    ...generateToolSchema(WebTools),
  ];
}
```

---

## OpenClaw Skills 中的声明式风格

OpenClaw 的 SKILL.md 本质上就是声明式注解的文档形式——描述"这个工具在什么情况下用、有什么约束"，Agent 加载后自动应用：

```markdown
# SKILL.md
## Triggers
- user asks about weather
- location + temperature mentioned

## Constraints  
- No API key needed
- Rate limit: 60 req/min (wttr.in)
- Cache TTL: 10 minutes (weather changes slowly)

## Permissions required
- network:fetch
```

而 pi-mono 的 Tool 注册则用 Zod Schema 作为"类型注解"：

```typescript
// pi-mono 风格：Schema 即注解
const readFileTool = tool({
  name: 'read_file',
  description: '读取文件内容',
  schema: z.object({
    path: z.string().describe('文件路径'),
  }),
  // 权限/缓存/重试通过 middleware 链应用
  middleware: [permissionMiddleware('file:read'), cacheMiddleware(30)],
  execute: async ({ path }) => fs.readFile(path, 'utf-8'),
});
```

---

## 装饰器执行顺序

多个装饰器叠加时，**从下往上执行**（最近的最先包装）：

```typescript
@permission('file:read')   // 4. 最外层：先检查权限
@cache({ ttl: 60 })        // 3. 再看缓存
@retry({ maxAttempts: 3 }) // 2. 再重试
@timeout(5000)             // 1. 最内层：直接包裹原函数
async readFile(path: string) { ... }
```

调用链：`permission → cache → retry → timeout → 原函数`

---

## 单元测试注解行为

```typescript
describe('@retry decorator', () => {
  it('should retry on failure and eventually succeed', async () => {
    let callCount = 0;

    class TestTools {
      @retry({ maxAttempts: 3, backoff: 'fixed', delayMs: 10 })
      async flakyTool() {
        callCount++;
        if (callCount < 3) throw new Error('temporary failure');
        return 'success';
      }
    }

    const tools = new TestTools();
    const result = await tools.flakyTool();
    
    expect(result).toBe('success');
    expect(callCount).toBe(3); // 失败2次，第3次成功
  });

  it('should exhaust retries and throw', async () => {
    class TestTools {
      @retry({ maxAttempts: 2, delayMs: 10 })
      async alwaysFails() {
        throw new Error('permanent failure');
      }
    }

    await expect(new TestTools().alwaysFails())
      .rejects.toThrow('permanent failure');
  });
});

describe('@idempotent decorator', () => {
  it('should deduplicate concurrent identical calls', async () => {
    let execCount = 0;
    
    class TestTools {
      @idempotent(1000)
      async expensiveOp(id: string) {
        execCount++;
        await sleep(50);
        return `result-${id}`;
      }
    }

    const tools = new TestTools();
    // 并发发出 5 个相同请求
    const results = await Promise.all(Array(5).fill(null).map(() => tools.expensiveOp('abc')));
    
    expect(execCount).toBe(1);     // 只真正执行一次
    expect(results).toEqual(Array(5).fill('result-abc'));
  });
});
```

---

## 关键收益

| 传统方式 | 声明式注解 |
|---------|-----------|
| 样板代码分散在每个工具 | 元数据集中声明，一目了然 |
| 权限/缓存/重试逻辑混杂 | 关切分离，单独维护 |
| 难以全局审计工具策略 | Reflect.getMetadata 统一查询 |
| Schema 和行为容易不同步 | 注解即文档即行为 |
| 测试需要模拟所有横切关注 | 每个装饰器可独立单测 |

---

## 小结

声明式工具注解是一种**关注点分离**的设计哲学：
- 工具函数只关心业务逻辑
- 横切关注（权限/缓存/重试/超时/限流）由装饰器统一处理
- 元数据驱动 Schema 自动生成，保持定义与行为同步

在 Agent 系统中，这个模式让工具集快速扩张时仍然可维护——新增一个工具，只需声明它的"脾气"，基础设施自动到位。
