# 第 13 课：State Persistence - 状态持久化

> Agent 重启后不应该失忆，状态持久化是 Agent 可靠性的关键。

## 为什么需要状态持久化？

想象一下：你的 Agent 正在执行一个复杂的多步骤任务，突然进程崩溃了。没有状态持久化，所有进度都会丢失。用户得重新开始。

状态持久化解决三个核心问题：

1. **崩溃恢复** - 进程挂了，重启后能继续
2. **会话延续** - 用户关闭浏览器，下次回来状态还在
3. **水平扩展** - 多个 Agent 实例共享状态

## 状态的分类

### 按生命周期分

```
┌─────────────────────────────────────────────────────────┐
│                    Agent State 层次                      │
├─────────────────────────────────────────────────────────┤
│  Ephemeral (短暂)    │ 当前工具调用的中间结果           │
│  Session (会话)      │ 当前对话的消息历史               │
│  Persistent (持久)   │ 用户偏好、长期记忆               │
│  Global (全局)       │ 系统配置、共享知识库             │
└─────────────────────────────────────────────────────────┘
```

### 按结构分

| 类型 | 示例 | 存储方式 |
|------|------|----------|
| 结构化 | 用户设置、任务队列 | JSON/数据库 |
| 半结构化 | 对话历史 | JSON Lines |
| 非结构化 | 记忆文档、笔记 | Markdown 文件 |

## 持久化策略

### 策略 1：文件系统

最简单直接，适合单机部署。

```typescript
// OpenClaw 的文件存储方式
interface FileStateStore {
  // 会话状态存在 ~/.openclaw/sessions/
  sessionPath: string;
  // 记忆存在 workspace/memory/
  memoryPath: string;
  // 配置存在 ~/.openclaw/config.yaml
  configPath: string;
}

// 写入状态
async function saveSessionState(
  sessionId: string, 
  messages: Message[]
): Promise<void> {
  const path = `~/.openclaw/sessions/${sessionId}.json`;
  await fs.writeFile(path, JSON.stringify({
    id: sessionId,
    messages,
    updatedAt: Date.now(),
  }, null, 2));
}

// 读取状态
async function loadSessionState(
  sessionId: string
): Promise<SessionState | null> {
  const path = `~/.openclaw/sessions/${sessionId}.json`;
  try {
    const data = await fs.readFile(path, 'utf-8');
    return JSON.parse(data);
  } catch {
    return null; // 不存在
  }
}
```

**优点**：简单、无依赖、易调试（直接看文件）
**缺点**：不支持并发写、不适合分布式

### 策略 2：SQLite

单机部署的升级版，支持事务和查询。

```typescript
// pi-mono 风格的 SQLite 存储
import Database from 'better-sqlite3';

class SqliteStateStore {
  private db: Database.Database;

  constructor(dbPath: string) {
    this.db = new Database(dbPath);
    this.init();
  }

  private init() {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS sessions (
        id TEXT PRIMARY KEY,
        data TEXT NOT NULL,
        created_at INTEGER NOT NULL,
        updated_at INTEGER NOT NULL
      );
      
      CREATE TABLE IF NOT EXISTS messages (
        id TEXT PRIMARY KEY,
        session_id TEXT NOT NULL,
        role TEXT NOT NULL,
        content TEXT NOT NULL,
        created_at INTEGER NOT NULL,
        FOREIGN KEY (session_id) REFERENCES sessions(id)
      );
      
      CREATE INDEX IF NOT EXISTS idx_messages_session 
        ON messages(session_id, created_at);
    `);
  }

  saveMessage(sessionId: string, message: Message): void {
    const stmt = this.db.prepare(`
      INSERT INTO messages (id, session_id, role, content, created_at)
      VALUES (?, ?, ?, ?, ?)
    `);
    
    stmt.run(
      message.id,
      sessionId,
      message.role,
      JSON.stringify(message.content),
      Date.now()
    );
  }

  getMessages(sessionId: string, limit = 100): Message[] {
    const stmt = this.db.prepare(`
      SELECT * FROM messages 
      WHERE session_id = ? 
      ORDER BY created_at DESC 
      LIMIT ?
    `);
    
    return stmt.all(sessionId, limit).reverse().map(row => ({
      id: row.id,
      role: row.role,
      content: JSON.parse(row.content),
    }));
  }
}
```

### 策略 3：Redis/KV 存储

分布式场景的选择。

```typescript
// 分布式状态存储
class RedisStateStore {
  private redis: Redis;

  async saveSession(sessionId: string, state: SessionState): Promise<void> {
    const key = `session:${sessionId}`;
    await this.redis.set(key, JSON.stringify(state), 'EX', 86400); // 24h TTL
  }

  async getSession(sessionId: string): Promise<SessionState | null> {
    const key = `session:${sessionId}`;
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }

  // 使用 List 存储消息（支持分页）
  async appendMessage(sessionId: string, message: Message): Promise<void> {
    const key = `messages:${sessionId}`;
    await this.redis.rpush(key, JSON.stringify(message));
    await this.redis.expire(key, 86400);
  }

  async getRecentMessages(
    sessionId: string, 
    count: number
  ): Promise<Message[]> {
    const key = `messages:${sessionId}`;
    const items = await this.redis.lrange(key, -count, -1);
    return items.map(item => JSON.parse(item));
  }
}
```

## Checkpoint 机制

对于长时间运行的任务，Checkpoint 是救命稻草。

### 设计思路

```typescript
interface Checkpoint {
  id: string;
  taskId: string;
  step: number;           // 当前步骤
  totalSteps: number;     // 总步骤
  state: unknown;         // 步骤特定状态
  createdAt: number;
}

class CheckpointManager {
  private store: StateStore;
  
  async save(checkpoint: Checkpoint): Promise<void> {
    const key = `checkpoint:${checkpoint.taskId}`;
    await this.store.set(key, checkpoint);
  }
  
  async load(taskId: string): Promise<Checkpoint | null> {
    const key = `checkpoint:${taskId}`;
    return this.store.get(key);
  }
  
  async delete(taskId: string): Promise<void> {
    const key = `checkpoint:${taskId}`;
    await this.store.delete(key);
  }
}
```

### 实战：可恢复的多步骤任务

```typescript
async function executeWithCheckpoint(
  task: Task,
  steps: Step[],
  checkpointManager: CheckpointManager
): Promise<Result> {
  // 1. 尝试恢复
  let checkpoint = await checkpointManager.load(task.id);
  let startStep = checkpoint?.step ?? 0;
  let state = checkpoint?.state ?? {};

  console.log(checkpoint 
    ? `Resuming from step ${startStep}` 
    : 'Starting fresh'
  );

  // 2. 执行步骤
  for (let i = startStep; i < steps.length; i++) {
    const step = steps[i];
    
    try {
      // 执行步骤
      state = await step.execute(state);
      
      // 保存 checkpoint
      await checkpointManager.save({
        id: `${task.id}-${i}`,
        taskId: task.id,
        step: i + 1,  // 下一步
        totalSteps: steps.length,
        state,
        createdAt: Date.now(),
      });
      
    } catch (error) {
      // 失败时 checkpoint 已保存，下次可恢复
      console.error(`Step ${i} failed, checkpoint saved`);
      throw error;
    }
  }

  // 3. 完成后清理
  await checkpointManager.delete(task.id);
  return state as Result;
}
```

## OpenClaw 的状态管理实践

OpenClaw 采用混合策略：

```yaml
# 状态存储分层
storage:
  # 会话消息 - JSON 文件
  sessions: ~/.openclaw/sessions/
  
  # 工作区记忆 - Markdown 文件
  memory: 
    - workspace/MEMORY.md      # 长期记忆
    - workspace/memory/*.md    # 日记
  
  # 配置 - YAML 文件
  config: ~/.openclaw/config.yaml
  
  # Cron 任务 - SQLite
  cron: ~/.openclaw/cron.db
```

### 会话恢复流程

```typescript
async function resumeSession(sessionKey: string): Promise<Session> {
  // 1. 加载会话元数据
  const meta = await loadSessionMeta(sessionKey);
  if (!meta) {
    return createNewSession(sessionKey);
  }
  
  // 2. 加载消息历史
  const messages = await loadMessages(sessionKey);
  
  // 3. 重建上下文
  const context = await rebuildContext(messages, {
    maxTokens: 100000,  // 上下文窗口限制
    compactIfNeeded: true,
  });
  
  // 4. 恢复工具状态
  const toolState = await loadToolState(sessionKey);
  
  return {
    key: sessionKey,
    messages: context.messages,
    toolState,
    resumedAt: Date.now(),
  };
}
```

## 状态一致性

### Write-Ahead Logging (WAL)

借鉴数据库的 WAL 思想：

```typescript
class WalStateStore {
  private walPath: string;
  private statePath: string;
  
  async append(operation: Operation): Promise<void> {
    // 1. 先写 WAL
    await fs.appendFile(
      this.walPath,
      JSON.stringify(operation) + '\n'
    );
    
    // 2. 再更新状态
    await this.applyOperation(operation);
  }
  
  async recover(): Promise<void> {
    // 从 WAL 重放未完成的操作
    const wal = await fs.readFile(this.walPath, 'utf-8');
    const operations = wal.split('\n')
      .filter(Boolean)
      .map(line => JSON.parse(line));
    
    for (const op of operations) {
      await this.applyOperation(op);
    }
    
    // 清空 WAL
    await fs.truncate(this.walPath, 0);
  }
}
```

### 原子性保证

```typescript
// 使用临时文件 + 重命名保证原子性
async function atomicWrite(
  path: string, 
  content: string
): Promise<void> {
  const tempPath = `${path}.tmp.${Date.now()}`;
  
  try {
    await fs.writeFile(tempPath, content);
    await fs.rename(tempPath, path);  // 原子操作
  } catch (error) {
    // 清理临时文件
    await fs.unlink(tempPath).catch(() => {});
    throw error;
  }
}
```

## 实战技巧

### 1. 选择合适的存储

```
单机 + 简单需求 → 文件系统
单机 + 需要查询 → SQLite
分布式 + 高并发 → Redis
分布式 + 持久化 → PostgreSQL + Redis 缓存
```

### 2. 状态版本化

```typescript
interface VersionedState {
  version: number;  // Schema 版本
  data: unknown;
}

function migrateState(state: VersionedState): VersionedState {
  let current = state;
  
  // 逐版本迁移
  while (current.version < CURRENT_VERSION) {
    current = migrations[current.version](current);
  }
  
  return current;
}
```

### 3. 定期清理

```typescript
// Cron 任务定期清理过期状态
async function cleanupOldSessions(): Promise<void> {
  const cutoff = Date.now() - 7 * 24 * 60 * 60 * 1000; // 7 天
  
  const sessions = await listSessions();
  for (const session of sessions) {
    if (session.updatedAt < cutoff) {
      await deleteSession(session.id);
    }
  }
}
```

### 4. 敏感数据处理

```typescript
// 加密敏感字段
async function saveState(state: State): Promise<void> {
  const encrypted = {
    ...state,
    credentials: await encrypt(state.credentials),
  };
  await store.set(state.id, encrypted);
}
```

## 总结

状态持久化的核心原则：

| 原则 | 说明 |
|------|------|
| **Fail-safe** | 任何时刻崩溃都不丢数据 |
| **Idempotent** | 重复恢复结果一致 |
| **Incremental** | 增量保存，不是全量 |
| **Versioned** | 支持 Schema 演进 |

状态持久化看起来简单，但做好不容易。关键是：

1. **明确状态分类** - 哪些必须持久化，哪些可以丢
2. **选择合适存储** - 根据场景选技术
3. **保证一致性** - 用 WAL 或原子操作
4. **考虑恢复** - 写代码时就想好怎么恢复

下节课我们讲 **Rate Limiting & Throttling（限流）** - Agent 如何优雅地处理 API 限制。

---

## 练习

1. 实现一个支持 checkpoint 的文件下载器（支持断点续传）
2. 为你的 Agent 添加会话状态持久化
3. 设计一个状态迁移系统，支持 Schema 变更

## 参考

- [SQLite WAL Mode](https://www.sqlite.org/wal.html)
- [Redis Persistence](https://redis.io/docs/management/persistence/)
- [OpenClaw Session Management](https://github.com/openclaw/openclaw)
