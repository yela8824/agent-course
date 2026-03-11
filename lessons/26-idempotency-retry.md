# 第26课：幂等性与重试模式 (Idempotency & Retry Patterns)

## 为什么需要幂等性和重试？

Agent 在执行任务时会遇到各种瞬时故障：
- **API 过载** (overloaded_error) - 模型服务器繁忙
- **限流** (rate limit / 429) - 请求太频繁
- **服务器错误** (500/502/503/504) - 后端暂时不可用
- **网络抖动** (connection refused, fetch failed)

这些错误都是**暂时性**的，重试通常可以解决。但重试带来新问题：

> 如果一个工具执行到一半失败了，重试会不会造成重复操作？

这就是**幂等性**要解决的问题。

---

## 核心概念

### 幂等性 (Idempotency)

**幂等操作**：执行一次和执行多次，结果相同。

```
# 幂等操作示例
echo "hello" > file.txt     # 写入固定内容，多次执行结果一样
git checkout main           # 切到 main，已经在 main 就没变化
rm -f file.txt              # 删除文件，已删除就跳过

# 非幂等操作示例
echo "hello" >> file.txt    # 追加，每次执行都会多一行
curl -X POST /create-order  # 创建订单，每次都新建
counter++                   # 自增，每次结果不同
```

### 重试策略

**指数退避 (Exponential Backoff)**：每次重试等待时间翻倍

```
第1次重试: 等待 1 秒
第2次重试: 等待 2 秒  
第3次重试: 等待 4 秒
第4次重试: 等待 8 秒
```

避免在服务恢复时造成"请求风暴"。

---

## pi-mono 实现分析

### 1. 判断可重试错误

```typescript
// pi-mono/packages/coding-agent/src/core/agent-session.ts

/**
 * 检查错误是否可重试
 * Context overflow 不重试（由 compaction 处理）
 */
private _isRetryableError(message: AssistantMessage): boolean {
  if (message.stopReason !== "error" || !message.errorMessage) return false;

  // Context overflow 由压缩处理，不是重试
  const contextWindow = this.model?.contextWindow ?? 0;
  if (isContextOverflow(message, contextWindow)) return false;

  const err = message.errorMessage;
  
  // 匹配所有瞬时错误类型
  return /overloaded|rate.?limit|too many requests|429|500|502|503|504|service.?unavailable|server error|internal error|connection.?error|connection.?refused|fetch failed|terminated|retry delay/i.test(err);
}
```

**关键设计**：
- 区分**可重试错误**（瞬时故障）和**不可重试错误**（逻辑错误）
- Context overflow 用**压缩**而不是重试
- 用正则匹配多种错误模式，兼容不同 API 的错误格式

### 2. 指数退避重试

```typescript
// pi-mono/packages/coding-agent/src/core/agent-session.ts

private async _handleRetryableError(message: AssistantMessage): Promise<boolean> {
  const settings = this.settingsManager.getRetrySettings();
  if (!settings.enabled) {
    this._resolveRetry();
    return false;
  }

  this._retryAttempt++;

  // 超过最大重试次数
  if (this._retryAttempt > settings.maxRetries) {
    this._emit({
      type: "auto_retry_end",
      success: false,
      attempt: this._retryAttempt - 1,
      finalError: message.errorMessage,
    });
    this._retryAttempt = 0;
    this._resolveRetry();
    return false;
  }

  // 指数退避：baseDelayMs * 2^(attempt-1)
  const delayMs = settings.baseDelayMs * 2 ** (this._retryAttempt - 1);

  this._emit({
    type: "auto_retry_start",
    attempt: this._retryAttempt,
    maxAttempts: settings.maxRetries,
    delayMs,
    errorMessage: message.errorMessage || "Unknown error",
  });

  // 关键：从状态中移除失败的消息
  const messages = this.agent.state.messages;
  if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
    this.agent.replaceMessages(messages.slice(0, -1));
  }

  // 可中断的等待
  this._retryAbortController = new AbortController();
  try {
    await sleep(delayMs, this._retryAbortController.signal);
  } catch {
    // 用户中断了重试
    this._emit({
      type: "auto_retry_end",
      success: false,
      attempt: this._retryAttempt,
      finalError: "Retry cancelled",
    });
    this._retryAttempt = 0;
    this._resolveRetry();
    return false;
  }

  // 用 setTimeout 打破事件处理链
  setTimeout(() => {
    this.agent.continue().catch(() => {
      // 失败会在下一个 agent_end 捕获
    });
  }, 0);

  return true;
}
```

**设计要点**：

| 要点 | 说明 |
|------|------|
| 移除失败消息 | 重试前把错误响应从状态移除，避免影响下次请求 |
| 可中断等待 | 用 AbortController 让用户能取消等待中的重试 |
| setTimeout 解耦 | 避免在事件处理中嵌套调用，防止栈溢出 |
| 事件通知 | 发出 retry_start/retry_end 事件，让 UI 能显示重试状态 |

### 3. 重试配置

```typescript
// 默认配置
settingsManager.applyOverrides({ 
  retry: { 
    enabled: true, 
    maxRetries: 3, 
    baseDelayMs: 1000  // 1秒起步
  } 
});
```

---

## Checkpoint 模式 - 保证幂等性

当工具执行会修改文件时，如何保证重试安全？答案是 **Checkpoint（检查点）**。

### Git Stash Checkpoint

```typescript
// pi-mono/packages/coding-agent/examples/extensions/git-checkpoint.ts

export default function (pi: ExtensionAPI) {
  const checkpoints = new Map<string, string>();  // entryId -> git stash ref
  let currentEntryId: string | undefined;

  // 记录当前会话 entry ID
  pi.on("tool_result", async (_event, ctx) => {
    const leaf = ctx.sessionManager.getLeafEntry();
    if (leaf) currentEntryId = leaf.id;
  });

  // 在每个 turn 开始前创建检查点
  pi.on("turn_start", async () => {
    // git stash create 创建一个 stash 但不 apply
    const { stdout } = await pi.exec("git", ["stash", "create"]);
    const ref = stdout.trim();
    if (ref && currentEntryId) {
      checkpoints.set(currentEntryId, ref);
    }
  });

  // fork 时询问是否恢复代码状态
  pi.on("session_before_fork", async (event, ctx) => {
    const ref = checkpoints.get(event.entryId);
    if (!ref) return;

    if (!ctx.hasUI) return;  // 非交互模式不自动恢复

    const choice = await ctx.ui.select("Restore code state?", [
      "Yes, restore code to that point",
      "No, keep current code",
    ]);

    if (choice?.startsWith("Yes")) {
      await pi.exec("git", ["stash", "apply", ref]);
      ctx.ui.notify("Code restored to checkpoint", "info");
    }
  });

  // Agent 结束后清理检查点
  pi.on("agent_end", async () => {
    checkpoints.clear();
  });
}
```

**工作原理**：
1. 每个 turn 开始前，用 `git stash create` 保存当前代码状态
2. 如果需要回滚（fork 或重试），可以 `git stash apply` 恢复
3. Agent 结束后清理所有检查点

---

## 设计幂等工具的原则

### 1. 读操作天然幂等

```python
def read_file(path: str) -> str:
    """读文件 - 天然幂等"""
    with open(path) as f:
        return f.read()
```

### 2. 写操作用"覆盖"而非"追加"

```python
# ❌ 非幂等：追加
def log_message_bad(msg: str):
    with open("log.txt", "a") as f:
        f.write(msg + "\n")

# ✅ 幂等：完整覆盖
def write_file(path: str, content: str):
    with open(path, "w") as f:
        f.write(content)
```

### 3. 使用幂等 Key

```python
import hashlib

def create_resource(data: dict) -> str:
    """使用内容 hash 作为幂等 key"""
    content = json.dumps(data, sort_keys=True)
    idempotency_key = hashlib.sha256(content.encode()).hexdigest()[:16]
    
    # 检查是否已存在
    existing = db.get_by_key(idempotency_key)
    if existing:
        return existing.id  # 幂等：返回已有的
    
    # 不存在则创建
    return db.create(data, key=idempotency_key)
```

### 4. 检查-执行-确认模式

```python
def ensure_directory(path: str):
    """确保目录存在 - 幂等"""
    if os.path.exists(path):
        return  # 已存在，跳过
    os.makedirs(path)  # 不存在，创建
```

---

## 实战：Python 重试装饰器

```python
import time
import random
from functools import wraps
from typing import Type, Tuple

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    retryable_exceptions: Tuple[Type[Exception], ...] = (Exception,),
    jitter: bool = True
):
    """
    指数退避重试装饰器
    
    Args:
        max_retries: 最大重试次数
        base_delay: 基础延迟秒数
        retryable_exceptions: 可重试的异常类型
        jitter: 是否添加随机抖动（避免雷群效应）
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    
                    if attempt == max_retries:
                        raise  # 最后一次，不再重试
                    
                    # 指数退避
                    delay = base_delay * (2 ** attempt)
                    
                    # 添加随机抖动 (0.5x ~ 1.5x)
                    if jitter:
                        delay *= 0.5 + random.random()
                    
                    print(f"Attempt {attempt + 1} failed: {e}")
                    print(f"Retrying in {delay:.2f}s...")
                    time.sleep(delay)
            
            raise last_exception
        return wrapper
    return decorator

# 使用示例
@retry_with_backoff(
    max_retries=3,
    base_delay=1.0,
    retryable_exceptions=(ConnectionError, TimeoutError)
)
def call_api(url: str) -> dict:
    """调用外部 API，自动重试瞬时错误"""
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

---

## 最佳实践总结

| 场景 | 策略 |
|------|------|
| API 调用 | 指数退避 + 最大重试次数 |
| 文件写入 | 完整覆盖（不追加） |
| 数据库操作 | 幂等 Key 或 UPSERT |
| 状态修改 | 检查-执行-确认模式 |
| 代码修改 | Git checkpoint 保存恢复点 |
| 网络请求 | 添加 jitter 避免雷群 |
| 长时任务 | 分段检查点，支持断点续传 |

### 不要重试的情况

- **400 Bad Request** - 请求本身有问题
- **401/403** - 认证/权限问题
- **404** - 资源不存在
- **Context overflow** - 需要压缩，不是重试

---

## 下节预告

下一课我们将学习 **Agent Warmup & Cold Start（冷启动优化）** - 如何减少 Agent 首次响应的延迟，预加载资源和模型。

---

## 课后思考

1. 如果一个工具执行到一半崩溃（比如写了一半文件），如何设计才能安全重试？
2. 为什么要在延迟中添加随机 jitter？"雷群效应"是什么？
3. 对于发送消息这种操作，如何实现幂等性？（提示：消息 ID）
