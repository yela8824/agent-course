# 102 - Agent 工具结果转换管道（Tool Result Transformation Pipeline）

> **核心思路：** 工具返回的原始数据往往很"脏"——字段不一致、内容超长、格式混乱、夹带敏感信息。与其让 LLM 直接消化这些原始数据，不如在中间加一层**转换管道**，把原始工具结果标准化、压缩、增强，再送入上下文。

---

## 为什么需要这层管道？

```
[Tool Call] ──raw──▶ [Transform Pipeline] ──clean──▶ [LLM Context]
```

没有管道时的常见问题：

| 问题 | 例子 |
|------|------|
| 内容爆炸 | `read_file` 返回 50KB 代码，直接塞进 context |
| 格式混乱 | 有的工具返回 JSON，有的返回纯文本，有的返回 HTML |
| 脏数据 | API 返回 `{"data": null, "error": null, "status": "ok"}` |
| 敏感泄露 | 数据库查询结果含密码 hash、手机号 |
| 无用噪音 | HTTP 响应带 200 个 header，LLM 全看到了 |

管道让你在**一个地方**统一处理所有这些问题。

---

## 管道结构

```typescript
// pi-mono 风格：每个 transformer 是独立的纯函数
type Transformer = (result: ToolResult, ctx: TransformContext) => ToolResult;

interface ToolResult {
  toolName: string;
  callId: string;
  content: string | object;
  metadata?: Record<string, unknown>;
  error?: string;
}

interface TransformContext {
  userId: string;
  sessionId: string;
  toolSchema?: ToolSchema;
  maxTokens?: number;
}
```

### 核心管道实现

```typescript
class ToolResultPipeline {
  private transformers: Transformer[] = [];

  use(transformer: Transformer): this {
    this.transformers.push(transformer);
    return this;
  }

  async process(result: ToolResult, ctx: TransformContext): Promise<ToolResult> {
    let current = result;
    for (const transformer of this.transformers) {
      try {
        current = await transformer(current, ctx);
      } catch (err) {
        // 单个 transformer 失败不影响后续，记录日志后继续
        console.error(`[Pipeline] transformer failed: ${err}`);
      }
    }
    return current;
  }
}

// 全局管道实例
export const pipeline = new ToolResultPipeline()
  .use(normalizeFormat)    // 1. 格式标准化
  .use(truncateContent)    // 2. 内容截断
  .use(stripSensitive)     // 3. 敏感字段清除
  .use(enrichMetadata)     // 4. 元数据增强
  .use(wrapError);         // 5. 错误包装
```

---

## 五个核心 Transformer

### 1. 格式标准化（normalizeFormat）

```typescript
// 无论工具返回啥格式，统一转成 string 送 LLM
function normalizeFormat(result: ToolResult): ToolResult {
  let content = result.content;

  if (typeof content === 'object' && content !== null) {
    // 去掉 null 字段，减少噪音
    const cleaned = Object.fromEntries(
      Object.entries(content).filter(([_, v]) => v !== null && v !== undefined)
    );
    content = JSON.stringify(cleaned, null, 2);
  } else if (typeof content !== 'string') {
    content = String(content);
  }

  // HTML 转纯文本（fetch 工具常见）
  if (typeof content === 'string' && content.includes('<!DOCTYPE')) {
    content = htmlToText(content); // 你自己实现或用库
  }

  return { ...result, content };
}
```

### 2. 内容截断（truncateContent）

```typescript
// 按 token 预算截断，保头保尾（最有价值的部分）
function truncateContent(
  result: ToolResult,
  ctx: TransformContext
): ToolResult {
  const MAX_CHARS = ctx.maxTokens ? ctx.maxTokens * 4 : 8000; // 粗估 1 token ≈ 4 chars
  const content = String(result.content);

  if (content.length <= MAX_CHARS) return result;

  // 头部 60% + 尾部 40%，中间加省略提示
  const headLen = Math.floor(MAX_CHARS * 0.6);
  const tailLen = MAX_CHARS - headLen;
  const truncated =
    content.slice(0, headLen) +
    `\n\n... [truncated ${content.length - MAX_CHARS} chars] ...\n\n` +
    content.slice(-tailLen);

  return {
    ...result,
    content: truncated,
    metadata: { ...result.metadata, truncated: true, originalLength: content.length },
  };
}
```

### 3. 敏感字段清除（stripSensitive）

```typescript
const SENSITIVE_PATTERNS = [
  /password["']?\s*[:=]\s*["']?[\w!@#$%^&*]+/gi,
  /api[_-]?key["']?\s*[:=]\s*["']?[\w-]{20,}/gi,
  /bearer\s+[\w-]{20,}/gi,
  /\b\d{11}\b/g,    // 手机号（11位数字）
];

function stripSensitive(result: ToolResult): ToolResult {
  let content = String(result.content);

  for (const pattern of SENSITIVE_PATTERNS) {
    content = content.replace(pattern, '[REDACTED]');
  }

  return { ...result, content };
}
```

### 4. 元数据增强（enrichMetadata）

```typescript
// 给 LLM 加一点"旁白"，帮助它理解结果的上下文
function enrichMetadata(result: ToolResult, ctx: TransformContext): ToolResult {
  const now = Date.now();
  return {
    ...result,
    metadata: {
      ...result.metadata,
      toolName: result.toolName,
      fetchedAt: new Date(now).toISOString(),
      sessionId: ctx.sessionId,
    },
    // 在 content 前面加一行"旁白"
    content: `[Tool: ${result.toolName} | ${new Date(now).toLocaleTimeString()}]\n${result.content}`,
  };
}
```

### 5. 错误包装（wrapError）

```typescript
// 把技术错误转成 LLM 友好的描述
function wrapError(result: ToolResult): ToolResult {
  if (!result.error) return result;

  const friendlyErrors: Record<string, string> = {
    ECONNREFUSED: '服务暂时不可用，请稍后再试',
    ETIMEDOUT: '请求超时，网络可能不稳定',
    '404': '资源不存在',
    '429': '请求太频繁，已触发限流',
    '500': '服务器内部错误',
  };

  const friendly = Object.entries(friendlyErrors).find(([code]) =>
    result.error!.includes(code)
  );

  return {
    ...result,
    content: `工具执行失败：${friendly ? friendly[1] : result.error}。请根据实际情况决定是否重试或换个方案。`,
    error: undefined, // 清除原始错误，防止 LLM 看到技术细节后产生幻觉
  };
}
```

---

## 接入 Agent Loop

```typescript
// 在工具执行后、注入 LLM context 前调用管道
async function executeToolWithPipeline(
  toolCall: ToolCall,
  ctx: TransformContext
): Promise<ToolResult> {
  // 1. 执行原始工具
  let result: ToolResult;
  try {
    const raw = await dispatchTool(toolCall);
    result = { toolName: toolCall.name, callId: toolCall.id, content: raw, error: undefined };
  } catch (err) {
    result = { toolName: toolCall.name, callId: toolCall.id, content: '', error: String(err) };
  }

  // 2. 过管道
  const transformed = await pipeline.process(result, ctx);

  // 3. 返回给 Agent Loop 注入 context
  return transformed;
}
```

**OpenClaw 实现参考：**  
`openclaw/src/agent/tool-executor.ts` 中的 `executeToolCall` 就在这里做了类似的 truncation + error normalization。  
`learn-claude-code` 的 `tool_handler.py` 对工具结果也有 `_format_result()` 函数做类似处理。

---

## 按工具类型定制管道

不同工具需要不同的转换策略：

```typescript
const TOOL_SPECIFIC_CONFIG: Record<string, Partial<TransformContext>> = {
  read_file:   { maxTokens: 2000 },  // 文件内容严格截断
  web_fetch:   { maxTokens: 3000 },  // 网页内容适度截断
  exec:        { maxTokens: 1000 },  // 命令输出最短
  memory_get:  { maxTokens: 8000 },  // 记忆内容尽量保全
  web_search:  { maxTokens: 4000 },  // 搜索结果中等
};

// 合并到 ctx 中
function getCtxForTool(toolName: string, baseCtx: TransformContext): TransformContext {
  return { ...baseCtx, ...TOOL_SPECIFIC_CONFIG[toolName] };
}
```

---

## 效果对比

| | 无管道 | 有管道 |
|---|---|---|
| read_file(50KB 文件) | 塞 12500 tokens | 压缩到 500 tokens，保头保尾 |
| DB 查询含手机号 | 手机号泄露给 LLM | 全部 `[REDACTED]` |
| API 返回 HTML 页面 | LLM 看到一堆 `<div>` | 干净纯文本 |
| 工具 timeout 报错 | `ETIMEDOUT connect ECONNREFUSED` | `"请求超时，请稍后重试"` |
| API 返回 `{data:null}` | LLM 困惑 null 是啥意思 | 空字段被过滤，输出 `{}` |

---

## 关键设计原则

1. **纯函数优先** - 每个 transformer 无副作用，输入输出明确，易于单测
2. **失败隔离** - 单个 transformer 异常不 crash 整个管道
3. **可观测** - 记录每个 transformer 的耗时和输出大小变化
4. **按需配置** - 不同工具、不同用户、不同场景可以走不同的管道配置
5. **管道顺序有意义** - 先格式化，再截断（否则截断的是乱码），再清敏感（基于标准格式），最后增强

---

## 小结

工具结果转换管道解决了 Agent 开发中一个常被忽视的问题：**原始工具输出不适合直接送 LLM**。通过在工具执行层和 LLM 上下文注入层之间加一个标准化的管道，你可以：

- 统一格式，减少 LLM 困惑
- 截断超长内容，节省 token
- 清除敏感数据，满足合规要求
- 友好化错误信息，提升 Agent 决策质量
- 在一个地方解决所有原始数据问题，业务逻辑保持干净

这是让 Agent 稳定运行在生产环境的**基础设施层**，不显山露水，但缺了它问题一堆。
