# 第十课：Session Management 会话管理

> Agent 不是一次性对话，而是持续的关系。会话管理是让 Agent "认识你" 的基础设施。

## 为什么会话管理很重要？

想象一下：

```
用户: 帮我订今晚的餐厅
Agent: 好的，请问您在哪个城市？
用户: 悉尼
Agent: 好的，您喜欢什么菜系？

--- 网络断开，用户重新连接 ---

用户: 中餐吧
Agent: 您好，请问有什么可以帮您？  // 💀 完全忘了之前的对话
```

没有会话管理，Agent 就像金鱼一样，7 秒钟就忘了所有事。

## 会话的核心概念

### 1. Session vs Conversation

```typescript
// 会话 (Session) - 技术层面
interface Session {
  id: string;              // 唯一标识
  userId: string;          // 谁的会话
  channelId: string;       // 哪个渠道（Telegram/Discord/Web）
  state: SessionState;     // 会话状态
  createdAt: Date;
  lastActiveAt: Date;
  messages: Message[];     // 消息历史
  metadata: Record<string, any>;  // 自定义数据
}

// 对话 (Conversation) - 业务层面
// 一个 Session 可能包含多个 Conversation
// 比如：用户先聊工作，再聊生活，但在同一个 Session
```

### 2. Session 生命周期

```
创建 → 活跃 → 空闲 → 恢复/过期
 ↓      ↓      ↓       ↓
init  active  idle   resume/expire
```

## OpenClaw 的会话设计

### 会话类型

```typescript
// OpenClaw 支持三种会话模式
type SessionKind = 
  | 'main'      // 主会话：用户直接对话
  | 'isolated'  // 隔离会话：子任务执行
  | 'shared';   // 共享会话：群聊等多人场景
```

### 会话标识

```typescript
// OpenClaw 的 sessionKey 设计
// 格式: {channel}:{channelId}:{userId}
const sessionKey = `telegram:67431246:67431246`;
//                  渠道      群组ID    用户ID

// 为什么这样设计？
// 1. 同一用户在不同渠道有不同会话
// 2. 同一用户在不同群组有不同会话
// 3. 私聊和群聊分开
```

## 实现：会话存储

### 内存存储（开发用）

```typescript
// 最简单的实现
class MemorySessionStore {
  private sessions = new Map<string, Session>();
  
  async get(sessionKey: string): Promise<Session | null> {
    return this.sessions.get(sessionKey) || null;
  }
  
  async set(sessionKey: string, session: Session): Promise<void> {
    session.lastActiveAt = new Date();
    this.sessions.set(sessionKey, session);
  }
  
  async delete(sessionKey: string): Promise<void> {
    this.sessions.delete(sessionKey);
  }
}
```

### Redis 存储（生产用）

```typescript
// pi-mono 风格的实现
class RedisSessionStore {
  constructor(private redis: Redis, private ttlSeconds = 86400) {}
  
  async get(sessionKey: string): Promise<Session | null> {
    const data = await this.redis.get(`session:${sessionKey}`);
    if (!data) return null;
    
    const session = JSON.parse(data);
    // 反序列化时恢复 Date 对象
    session.createdAt = new Date(session.createdAt);
    session.lastActiveAt = new Date(session.lastActiveAt);
    return session;
  }
  
  async set(sessionKey: string, session: Session): Promise<void> {
    session.lastActiveAt = new Date();
    await this.redis.setex(
      `session:${sessionKey}`,
      this.ttlSeconds,
      JSON.stringify(session)
    );
  }
  
  // 滑动过期：每次访问都续期
  async touch(sessionKey: string): Promise<void> {
    await this.redis.expire(`session:${sessionKey}`, this.ttlSeconds);
  }
}
```

### SQLite 存储（本地 Agent）

```typescript
// OpenClaw 本地运行时的选择
class SqliteSessionStore {
  constructor(private db: Database) {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS sessions (
        session_key TEXT PRIMARY KEY,
        user_id TEXT NOT NULL,
        channel TEXT NOT NULL,
        messages TEXT NOT NULL,  -- JSON array
        metadata TEXT,           -- JSON object
        created_at INTEGER NOT NULL,
        last_active_at INTEGER NOT NULL
      )
    `);
  }
  
  async get(sessionKey: string): Promise<Session | null> {
    const row = this.db.prepare(`
      SELECT * FROM sessions WHERE session_key = ?
    `).get(sessionKey);
    
    if (!row) return null;
    
    return {
      id: row.session_key,
      messages: JSON.parse(row.messages),
      metadata: JSON.parse(row.metadata || '{}'),
      // ...
    };
  }
}
```

## 会话隔离：为什么需要 Isolated Sessions？

### 问题场景

```
用户: 帮我分析这个 100MB 的日志文件

--- 同时 ---

用户: 现在几点了？
```

如果只有一个会话，"现在几点了" 要等日志分析完才能回复。

### 解决方案：子会话

```typescript
// OpenClaw 的 sessions_spawn
interface SpawnOptions {
  task: string;           // 子任务描述
  label?: string;         // 会话标签
  model?: string;         // 可以用不同模型
  cleanup?: 'delete' | 'keep';  // 完成后是否删除
  runTimeoutSeconds?: number;
}

// 主会话可以 spawn 子会话
async function handleLongTask(request: string) {
  // 启动隔离会话处理长任务
  await sessions_spawn({
    task: `分析用户上传的日志文件: ${request}`,
    label: 'log-analysis',
    model: 'fast',  // 用便宜的模型处理
    cleanup: 'delete'
  });
  
  // 主会话立即返回
  return "好的，我在后台分析日志，完成后通知您";
}
```

### 会话通信

```typescript
// 主会话 → 子会话
await sessions_send({
  label: 'log-analysis',
  message: '用户追加了一个文件'
});

// 子会话 → 主会话
// 子会话完成后自动把结果发回主会话
```

## 会话状态管理

### 状态机设计

```typescript
type SessionState = 
  | 'idle'       // 等待用户输入
  | 'thinking'   // Agent 思考中
  | 'executing'  // 执行工具
  | 'streaming'  // 流式输出中
  | 'waiting';   // 等待外部响应

class SessionStateMachine {
  private state: SessionState = 'idle';
  
  transition(event: string): void {
    const transitions: Record<string, Record<string, SessionState>> = {
      'idle': {
        'user_message': 'thinking',
        'system_event': 'thinking'
      },
      'thinking': {
        'tool_call': 'executing',
        'start_response': 'streaming',
        'error': 'idle'
      },
      'executing': {
        'tool_complete': 'thinking',
        'tool_error': 'thinking'
      },
      'streaming': {
        'stream_end': 'idle'
      }
    };
    
    const nextState = transitions[this.state]?.[event];
    if (nextState) {
      this.state = nextState;
    }
  }
}
```

### 并发控制

```typescript
// 防止同一会话并发处理多条消息
class SessionLock {
  private locks = new Map<string, Promise<void>>();
  
  async acquire(sessionKey: string): Promise<() => void> {
    // 等待之前的锁释放
    while (this.locks.has(sessionKey)) {
      await this.locks.get(sessionKey);
    }
    
    // 创建新锁
    let release: () => void;
    const lock = new Promise<void>(resolve => {
      release = resolve;
    });
    
    this.locks.set(sessionKey, lock);
    
    return () => {
      this.locks.delete(sessionKey);
      release!();
    };
  }
}

// 使用
async function handleMessage(sessionKey: string, message: string) {
  const unlock = await sessionLock.acquire(sessionKey);
  try {
    // 处理消息...
  } finally {
    unlock();
  }
}
```

## 跨渠道会话

### 场景

用户可能通过多个渠道联系 Agent：
- Telegram 私聊
- Discord 服务器
- Web 界面
- API 调用

### 用户身份映射

```typescript
// 问题：同一个人在不同渠道有不同 ID
// Telegram: 67431246
// Discord: 123456789012345678
// Web: user@example.com

// 解决：统一用户 ID
interface UserIdentity {
  unifiedId: string;  // 内部统一 ID
  identities: {
    channel: string;
    externalId: string;
  }[];
}

// OpenClaw 的做法：通过 User 配置绑定
// user:
//   telegram: 67431246
//   discord: 123456789012345678
```

### 会话合并策略

```typescript
// 策略 1: 独立会话（默认）
// 每个渠道独立，互不影响

// 策略 2: 上下文共享
// 不同渠道可以访问共同的 memory
interface SharedContext {
  memory: MemoryStore;     // 共享记忆
  preferences: UserPrefs;  // 共享偏好
}

// 策略 3: 会话迁移
// 用户可以 "继续上次的对话"
async function continueSession(
  userId: string,
  fromChannel: string,
  toChannel: string
): Promise<Session> {
  const oldSession = await store.get(`${fromChannel}:${userId}`);
  const newSession = await store.get(`${toChannel}:${userId}`);
  
  // 把旧会话的消息历史复制到新会话
  newSession.messages = [
    ...oldSession.messages.slice(-10),  // 最近 10 条
    ...newSession.messages
  ];
  
  return newSession;
}
```

## 会话清理

### 过期策略

```typescript
// 定时清理过期会话
class SessionCleaner {
  constructor(
    private store: SessionStore,
    private maxIdleMs = 24 * 60 * 60 * 1000  // 24 小时
  ) {}
  
  async cleanup(): Promise<number> {
    const now = Date.now();
    let cleaned = 0;
    
    for await (const session of this.store.iterate()) {
      const idleTime = now - session.lastActiveAt.getTime();
      
      if (idleTime > this.maxIdleMs) {
        // 过期前可以做些事情
        await this.beforeExpire(session);
        await this.store.delete(session.id);
        cleaned++;
      }
    }
    
    return cleaned;
  }
  
  private async beforeExpire(session: Session): Promise<void> {
    // 保存重要信息到长期记忆
    const importantMessages = session.messages.filter(m => 
      m.metadata?.important
    );
    
    if (importantMessages.length > 0) {
      await memory.save(session.userId, importantMessages);
    }
  }
}
```

### 主动清理

```typescript
// 用户可以主动清除会话
async function handleClearCommand(session: Session): Promise<string> {
  // 保留会话，但清空消息历史
  session.messages = [];
  session.metadata = {};
  await store.set(session.id, session);
  
  return "好的，我已经忘记了之前的对话。让我们重新开始！";
}
```

## 实战：完整会话管理器

```typescript
class SessionManager {
  constructor(
    private store: SessionStore,
    private lock: SessionLock,
    private cleaner: SessionCleaner
  ) {
    // 每小时清理一次
    setInterval(() => this.cleaner.cleanup(), 60 * 60 * 1000);
  }
  
  async getOrCreate(sessionKey: string, userId: string): Promise<Session> {
    let session = await this.store.get(sessionKey);
    
    if (!session) {
      session = {
        id: sessionKey,
        userId,
        channelId: sessionKey.split(':')[1],
        state: 'idle',
        messages: [],
        metadata: {},
        createdAt: new Date(),
        lastActiveAt: new Date()
      };
      await this.store.set(sessionKey, session);
    }
    
    return session;
  }
  
  async addMessage(
    sessionKey: string, 
    message: Message
  ): Promise<void> {
    const unlock = await this.lock.acquire(sessionKey);
    try {
      const session = await this.store.get(sessionKey);
      if (!session) throw new Error('Session not found');
      
      session.messages.push(message);
      
      // 限制消息历史长度
      if (session.messages.length > 100) {
        session.messages = session.messages.slice(-50);
      }
      
      await this.store.set(sessionKey, session);
    } finally {
      unlock();
    }
  }
  
  async getContext(sessionKey: string): Promise<Message[]> {
    const session = await this.store.get(sessionKey);
    if (!session) return [];
    
    // 返回适合放入 prompt 的消息
    return session.messages.slice(-20);  // 最近 20 条
  }
}
```

## 关键设计决策

### 1. 消息存储粒度

```typescript
// 方案 A: 只存关键信息
interface MinimalMessage {
  role: 'user' | 'assistant';
  content: string;
  timestamp: number;
}

// 方案 B: 完整信息（推荐）
interface FullMessage {
  role: 'user' | 'assistant' | 'system' | 'tool';
  content: string;
  toolCalls?: ToolCall[];      // 工具调用
  toolResults?: ToolResult[];  // 工具结果
  metadata?: Record<string, any>;
  timestamp: number;
}

// 方案 B 虽然占空间，但调试时非常有用
```

### 2. 何时持久化？

```typescript
// 方案 A: 每条消息都存（安全但慢）
// 方案 B: Agent 回复完再存（有丢失风险）
// 方案 C: Write-Ahead Log + 定期快照（生产推荐）

class WALSessionStore {
  // 先写 WAL
  async append(sessionKey: string, message: Message): Promise<void> {
    await this.wal.append(`${sessionKey}:${Date.now()}`, message);
  }
  
  // 定期合并到主存储
  async checkpoint(): Promise<void> {
    const entries = await this.wal.readAll();
    // group by session, merge, save
    await this.wal.clear();
  }
}
```

### 3. 隐私考量

```typescript
// 不要在会话中存储敏感信息
async function sanitizeMessage(message: Message): Promise<Message> {
  return {
    ...message,
    content: message.content
      .replace(/\b\d{16}\b/g, '[CARD]')       // 信用卡
      .replace(/\b\d{3}-\d{2}-\d{4}\b/g, '[SSN]')  // SSN
      .replace(/password[=:]\s*\S+/gi, 'password=[REDACTED]')
  };
}
```

## 总结

| 概念 | 说明 |
|------|------|
| Session | 用户与 Agent 的一次交互周期 |
| SessionKey | 唯一标识，通常包含渠道+用户信息 |
| Isolated Session | 独立执行的子会话，不阻塞主会话 |
| Session Lock | 防止并发冲突 |
| Session State | 会话当前状态（idle/thinking/executing） |

**核心原则**：
1. 会话是有状态的，必须持久化
2. 并发访问要加锁
3. 子任务用隔离会话
4. 定期清理过期会话
5. 隐私数据要脱敏

下一课我们讲 **Observability & Tracing**，如何追踪 Agent 的每一个决策。
