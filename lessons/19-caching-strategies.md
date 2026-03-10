# 第 19 课：Caching Strategies - Agent 缓存策略

> 缓存是优化 Agent 性能和降低成本的核心武器

## 为什么 Agent 需要缓存？

Agent 面临几个独特挑战：

1. **LLM 调用昂贵** - 每次调用都要花钱（tokens 计费）
2. **延迟敏感** - 用户不想等 3-5 秒看结果
3. **重复请求多** - 很多问题其实问过类似的
4. **工具调用成本** - API rate limit、网络延迟

合理的缓存策略能同时解决这四个问题。

## 缓存的层次结构

Agent 系统中有多层缓存机会：

```
┌─────────────────────────────────────────┐
│         Application Layer               │
│  (Session Cache, Response Cache)        │
├─────────────────────────────────────────┤
│         Model Layer                     │
│  (Prompt Cache, Embedding Cache)        │
├─────────────────────────────────────────┤
│         Tool Layer                      │
│  (Tool Result Cache, API Cache)         │
├─────────────────────────────────────────┤
│         Infrastructure Layer            │
│  (HTTP Cache, DNS Cache)                │
└─────────────────────────────────────────┘
```

## 1. Prompt Cache（提示词缓存）

### Anthropic 的 Prompt Caching

Anthropic 原生支持 prompt caching，这是最有价值的缓存机制：

```typescript
// pi-mono 风格的实现
interface CacheControl {
  type: 'ephemeral';
}

interface ContentBlock {
  type: 'text';
  text: string;
  cache_control?: CacheControl;
}

// 标记系统提示词为可缓存
const systemMessage: ContentBlock = {
  type: 'text',
  text: `你是一个代码助手...（很长的系统提示）`,
  cache_control: { type: 'ephemeral' }
};
```

**工作原理**：
- 相同的前缀 tokens 会被缓存
- 缓存命中时，只计费写入，不计费读取
- 缓存有效期约 5 分钟

**OpenClaw 实现**：

```typescript
// 构建消息时添加 cache_control
function buildMessages(systemPrompt: string, history: Message[]): AnthropicMessage[] {
  return [
    {
      role: 'system',
      content: [{
        type: 'text',
        text: systemPrompt,
        // 系统提示词标记为缓存
        cache_control: { type: 'ephemeral' }
      }]
    },
    // 历史消息的前 N 条也可以缓存
    ...history.slice(0, -2).map(msg => ({
      ...msg,
      content: addCacheControl(msg.content)
    })),
    // 最近的消息不缓存（变化太快）
    ...history.slice(-2)
  ];
}
```

### 成本节省

假设系统提示词 2000 tokens，每次对话 10 轮：

```
不缓存：2000 × 10 = 20,000 input tokens
缓存后：2000 × 1 (首次) + 2000 × 9 × 0.1 (缓存) = 3,800 tokens

节省：81%
```

## 2. Tool Result Cache（工具结果缓存）

工具调用经常返回相同结果，特别是：
- 天气查询（同一小时内）
- 文件读取（文件没变）
- API 数据（有 TTL）

### 基础实现

```typescript
// learn-claude-code 风格
class ToolCache {
  private cache: Map<string, CacheEntry> = new Map();
  
  private generateKey(toolName: string, params: unknown): string {
    return `${toolName}:${JSON.stringify(params)}`;
  }
  
  async execute(
    toolName: string, 
    params: unknown, 
    executor: () => Promise<ToolResult>
  ): Promise<ToolResult> {
    const key = this.generateKey(toolName, params);
    const cached = this.cache.get(key);
    
    // 检查缓存
    if (cached && !this.isExpired(cached)) {
      console.log(`[Cache HIT] ${toolName}`);
      return cached.result;
    }
    
    // 执行工具
    console.log(`[Cache MISS] ${toolName}`);
    const result = await executor();
    
    // 存入缓存
    this.cache.set(key, {
      result,
      timestamp: Date.now(),
      ttl: this.getTTL(toolName)
    });
    
    return result;
  }
  
  private getTTL(toolName: string): number {
    // 不同工具不同 TTL
    const ttlConfig: Record<string, number> = {
      'web_search': 3600_000,      // 1 小时
      'weather': 1800_000,          // 30 分钟
      'read_file': 60_000,          // 1 分钟（文件可能变）
      'web_fetch': 300_000,         // 5 分钟
      'default': 0                  // 不缓存
    };
    return ttlConfig[toolName] ?? ttlConfig.default;
  }
  
  private isExpired(entry: CacheEntry): boolean {
    return Date.now() - entry.timestamp > entry.ttl;
  }
}
```

### 内容感知缓存

有些工具结果依赖内容本身，需要更智能的缓存策略：

```typescript
// 文件读取：用文件 hash 作为缓存 key
class FileReadCache {
  private cache: Map<string, { hash: string; content: string }> = new Map();
  
  async readFile(path: string): Promise<string> {
    const stat = await fs.stat(path);
    const hash = `${stat.mtime.getTime()}-${stat.size}`;
    
    const cached = this.cache.get(path);
    if (cached && cached.hash === hash) {
      return cached.content;  // 文件没变，用缓存
    }
    
    const content = await fs.readFile(path, 'utf-8');
    this.cache.set(path, { hash, content });
    return content;
  }
}
```

## 3. Semantic Cache（语义缓存）

用户问题经常意思相同但表述不同：
- "天气怎么样" vs "今天天气如何" vs "今天热吗"

语义缓存通过 embedding 相似度来匹配：

```typescript
class SemanticCache {
  private entries: Array<{
    embedding: number[];
    query: string;
    response: string;
    timestamp: number;
  }> = [];
  
  private similarityThreshold = 0.92;  // 相似度阈值
  
  async get(query: string): Promise<string | null> {
    const queryEmbedding = await this.getEmbedding(query);
    
    for (const entry of this.entries) {
      const similarity = this.cosineSimilarity(queryEmbedding, entry.embedding);
      
      if (similarity >= this.similarityThreshold) {
        console.log(`[Semantic Cache HIT] similarity=${similarity.toFixed(3)}`);
        return entry.response;
      }
    }
    
    return null;
  }
  
  async set(query: string, response: string): Promise<void> {
    const embedding = await this.getEmbedding(query);
    
    this.entries.push({
      embedding,
      query,
      response,
      timestamp: Date.now()
    });
    
    // 限制缓存大小
    if (this.entries.length > 1000) {
      this.entries = this.entries.slice(-500);
    }
  }
  
  private cosineSimilarity(a: number[], b: number[]): number {
    let dotProduct = 0;
    let normA = 0;
    let normB = 0;
    
    for (let i = 0; i < a.length; i++) {
      dotProduct += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
  }
}
```

### 注意事项

语义缓存要小心：
1. **上下文依赖** - "继续" 这种请求不能缓存
2. **时间敏感** - "现在几点" 不能缓存
3. **用户相关** - 不同用户不能共享缓存

```typescript
// 判断是否可以语义缓存
function isCacheable(query: string, context: SessionContext): boolean {
  // 上下文依赖的请求
  const contextDependent = ['继续', '然后呢', '再来一个', '改一下'];
  if (contextDependent.some(kw => query.includes(kw))) {
    return false;
  }
  
  // 时间敏感的请求
  const timeSensitive = ['现在', '今天', '刚才', '最新'];
  if (timeSensitive.some(kw => query.includes(kw))) {
    return false;
  }
  
  // 涉及个人信息的请求
  if (query.includes('我的') || query.includes('我')) {
    return false;
  }
  
  return true;
}
```

## 4. Session Context Cache（会话上下文缓存）

长对话的上下文压缩后可以缓存：

```typescript
// OpenClaw 风格
interface SessionCache {
  sessionId: string;
  compressedContext: string;
  messageCount: number;
  lastUpdated: number;
}

class SessionContextCache {
  private cache: Map<string, SessionCache> = new Map();
  
  // 保存压缩后的上下文
  saveCompressedContext(sessionId: string, context: string, messageCount: number) {
    this.cache.set(sessionId, {
      sessionId,
      compressedContext: context,
      messageCount,
      lastUpdated: Date.now()
    });
  }
  
  // 恢复会话时直接用压缩版
  getContext(sessionId: string): string | null {
    const cached = this.cache.get(sessionId);
    if (!cached) return null;
    
    // 超过 24 小时的缓存失效
    if (Date.now() - cached.lastUpdated > 86400_000) {
      this.cache.delete(sessionId);
      return null;
    }
    
    return cached.compressedContext;
  }
}
```

## 5. 分布式缓存

生产环境需要跨实例共享缓存：

```typescript
// Redis 缓存实现
import Redis from 'ioredis';

class DistributedCache {
  private redis: Redis;
  private prefix = 'agent:cache:';
  
  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);
  }
  
  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get(this.prefix + key);
    return data ? JSON.parse(data) : null;
  }
  
  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    await this.redis.setex(
      this.prefix + key,
      ttlSeconds,
      JSON.stringify(value)
    );
  }
  
  // 带锁的缓存（防止缓存击穿）
  async getOrSet<T>(
    key: string, 
    factory: () => Promise<T>, 
    ttlSeconds: number
  ): Promise<T> {
    // 先尝试获取
    const cached = await this.get<T>(key);
    if (cached) return cached;
    
    // 获取锁
    const lockKey = `${this.prefix}lock:${key}`;
    const acquired = await this.redis.set(lockKey, '1', 'EX', 10, 'NX');
    
    if (!acquired) {
      // 没拿到锁，等一下再试
      await new Promise(r => setTimeout(r, 100));
      return this.getOrSet(key, factory, ttlSeconds);
    }
    
    try {
      const value = await factory();
      await this.set(key, value, ttlSeconds);
      return value;
    } finally {
      await this.redis.del(lockKey);
    }
  }
}
```

## 6. 缓存策略对比

| 缓存类型 | 命中率 | 实现复杂度 | 成本节省 | 适用场景 |
|---------|-------|-----------|---------|---------|
| Prompt Cache | 高 | 低 | 50-80% | 所有 Agent |
| Tool Result | 中 | 低 | 20-40% | 调用外部 API |
| Semantic Cache | 中 | 高 | 30-50% | 通用问答 |
| Session Cache | 高 | 中 | 40-60% | 长对话 |
| 分布式缓存 | 高 | 高 | 变化大 | 多实例部署 |

## 7. 缓存失效策略

缓存最难的部分是知道何时失效：

```typescript
// 多维度失效策略
class CacheInvalidator {
  // TTL 失效
  checkTTL(entry: CacheEntry): boolean {
    return Date.now() - entry.timestamp > entry.ttl;
  }
  
  // 依赖失效（文件变了，缓存也失效）
  checkDependency(entry: CacheEntry): boolean {
    if (!entry.dependencies) return false;
    
    for (const dep of entry.dependencies) {
      if (dep.type === 'file') {
        const currentMtime = fs.statSync(dep.path).mtime.getTime();
        if (currentMtime !== dep.mtime) return true;
      }
    }
    return false;
  }
  
  // 版本失效（系统提示词改了）
  checkVersion(entry: CacheEntry, currentVersion: string): boolean {
    return entry.version !== currentVersion;
  }
  
  // 综合判断
  shouldInvalidate(entry: CacheEntry, context: CacheContext): boolean {
    return this.checkTTL(entry) 
        || this.checkDependency(entry)
        || this.checkVersion(entry, context.version);
  }
}
```

## 8. 实战：OpenClaw 的缓存架构

```
┌──────────────────────────────────────────────────────────┐
│                      Gateway                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Prompt Cache │  │ Session Cache│  │ Tool Cache   │   │
│  │ (Anthropic)  │  │ (Memory)     │  │ (Memory/TTL) │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│                            │                              │
│                      ┌─────┴─────┐                       │
│                      │   Redis   │ (可选，多实例时)       │
│                      └───────────┘                       │
└──────────────────────────────────────────────────────────┘
```

**关键设计**：
1. Prompt Cache 交给 Anthropic 原生处理
2. Session Cache 用内存，重启时从文件恢复
3. Tool Cache 区分 TTL（天气短，文档长）
4. 多实例部署时启用 Redis

## 作业

1. 给你的 Agent 添加 Tool Result Cache，观察命中率
2. 实现一个简单的语义缓存，测试效果
3. 计算引入缓存后的 token 节省量

## 下一课预告

下一课我们讲 **Multi-Modal Agents** - 如何让 Agent 处理图片、音频和视频。

---

*第 19 课完*
