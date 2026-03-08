# 07 - Context Window 管理：在有限空间内高效工作

> 每个 LLM 都有 context window 限制。如何在有限的 token 空间内高效工作，是 Agent 开发的核心挑战之一。

## 为什么 Context Window 管理重要？

想象你在和一个只能记住最近 10 分钟对话的人交流。如果对话超过 10 分钟，早期内容就会被遗忘。LLM 的 context window 就是这个"短期记忆"的上限。

- **GPT-4**: ~128K tokens
- **Claude 3**: ~200K tokens  
- **Claude 4**: ~200K tokens

听起来很大？但在实际 Agent 应用中：
- System prompt 可能占 2-5K tokens
- 工具定义可能占 5-10K tokens
- 每次工具调用结果可能 1-50K tokens
- 对话历史累积起来非常快

一个活跃的 coding agent，几轮交互就可能用掉 50% 的 context。

## 核心策略

### 1. Token 计数与监控

首先，你需要知道当前用了多少 tokens：

```typescript
// pi-mono 的 token 计数方式
import { countTokens } from './tokenizer';

function getContextUsage(messages: Message[]): number {
  let total = 0;
  for (const msg of messages) {
    total += countTokens(msg.content);
    if (msg.tool_calls) {
      total += countTokens(JSON.stringify(msg.tool_calls));
    }
  }
  return total;
}

// 在每次 API 调用前检查
const usage = getContextUsage(messages);
const limit = model.contextWindow * 0.85; // 留 15% 余量

if (usage > limit) {
  messages = compactMessages(messages);
}
```

### 2. 消息压缩 (Context Compact)

当接近限制时，需要压缩历史消息：

```typescript
// OpenClaw 的压缩策略
async function compactMessages(
  messages: Message[],
  targetTokens: number
): Promise<Message[]> {
  // 策略 1: 保留系统消息和最近 N 条
  const systemMsg = messages.find(m => m.role === 'system');
  const recentMessages = messages.slice(-10);
  
  // 策略 2: 用 LLM 总结中间部分
  const middleMessages = messages.slice(1, -10);
  if (middleMessages.length > 0) {
    const summary = await summarizeConversation(middleMessages);
    return [
      systemMsg,
      { role: 'system', content: `[对话历史摘要]\n${summary}` },
      ...recentMessages
    ];
  }
  
  return [systemMsg, ...recentMessages];
}
```

### 3. 工具结果截断

工具返回的结果可能非常大（比如读取一个大文件），需要智能截断：

```typescript
// learn-claude-code 的截断策略
function truncateToolResult(result: string, maxTokens: number): string {
  const tokens = countTokens(result);
  
  if (tokens <= maxTokens) {
    return result;
  }
  
  // 对于文件内容，保留开头和结尾
  const lines = result.split('\n');
  const keepLines = Math.floor(maxTokens / 10); // 粗略估计
  
  if (lines.length > keepLines * 2) {
    const head = lines.slice(0, keepLines).join('\n');
    const tail = lines.slice(-keepLines).join('\n');
    return `${head}\n\n... [truncated ${lines.length - keepLines * 2} lines] ...\n\n${tail}`;
  }
  
  // 简单截断
  return result.slice(0, maxTokens * 4) + '\n... [truncated]';
}
```

### 4. 滑动窗口策略

对于长时间运行的 Agent，使用滑动窗口维护对话：

```typescript
class SlidingWindowContext {
  private messages: Message[] = [];
  private maxTokens: number;
  private systemPrompt: Message;
  
  constructor(systemPrompt: Message, maxTokens: number) {
    this.systemPrompt = systemPrompt;
    this.maxTokens = maxTokens;
  }
  
  addMessage(msg: Message) {
    this.messages.push(msg);
    this.trimIfNeeded();
  }
  
  private trimIfNeeded() {
    while (this.getTotalTokens() > this.maxTokens && this.messages.length > 2) {
      // 移除最早的用户-助手对（保持对话完整性）
      const firstUserIdx = this.messages.findIndex(m => m.role === 'user');
      if (firstUserIdx >= 0) {
        // 找到对应的 assistant 回复
        const assistantIdx = this.messages.findIndex(
          (m, i) => i > firstUserIdx && m.role === 'assistant'
        );
        if (assistantIdx >= 0) {
          this.messages.splice(firstUserIdx, assistantIdx - firstUserIdx + 1);
        }
      }
    }
  }
  
  getMessages(): Message[] {
    return [this.systemPrompt, ...this.messages];
  }
}
```

### 5. 分层记忆架构

高级方案：将记忆分层，只在需要时加载：

```typescript
// OpenClaw 的分层记忆
interface MemoryLayer {
  immediate: Message[];      // 当前对话（总在 context 中）
  working: string;           // 工作记忆摘要（总在 context 中）
  episodic: Memory[];        // 按需检索的长期记忆
  semantic: VectorStore;     // 向量数据库，语义搜索
}

async function buildContext(
  query: string,
  memory: MemoryLayer,
  maxTokens: number
): Promise<Message[]> {
  const messages: Message[] = [];
  let usedTokens = 0;
  
  // 1. 系统提示 + 工作记忆（必须包含）
  const systemWithWorking = buildSystemPrompt(memory.working);
  messages.push(systemWithWorking);
  usedTokens += countTokens(systemWithWorking.content);
  
  // 2. 语义搜索相关记忆
  const relevantMemories = await memory.semantic.search(query, 5);
  for (const mem of relevantMemories) {
    if (usedTokens + countTokens(mem.content) < maxTokens * 0.3) {
      messages.push({ role: 'system', content: `[相关记忆] ${mem.content}` });
      usedTokens += countTokens(mem.content);
    }
  }
  
  // 3. 即时对话（尽可能多保留）
  const remainingTokens = maxTokens - usedTokens;
  const trimmedImmediate = fitMessages(memory.immediate, remainingTokens);
  messages.push(...trimmedImmediate);
  
  return messages;
}
```

## 实战：OpenClaw 的 Context 管理

OpenClaw 如何处理 context：

```typescript
// openclaw/src/agent/context.ts (简化版)
export class ContextManager {
  private config: ContextConfig;
  
  async prepareContext(session: Session): Promise<Message[]> {
    const messages: Message[] = [];
    
    // 1. 构建系统提示
    const systemPrompt = await this.buildSystemPrompt(session);
    messages.push({ role: 'system', content: systemPrompt });
    
    // 2. 注入相关技能文档（按需）
    if (session.activeSkill) {
      const skillDoc = await this.loadSkill(session.activeSkill);
      messages.push({ role: 'system', content: skillDoc });
    }
    
    // 3. 检查是否需要压缩
    const historyTokens = this.countHistoryTokens(session.messages);
    const budget = this.config.maxContextTokens - this.countTokens(messages);
    
    if (historyTokens > budget) {
      // 触发压缩
      const compacted = await this.compact(session.messages, budget);
      messages.push(...compacted);
    } else {
      messages.push(...session.messages);
    }
    
    return messages;
  }
  
  private async compact(
    messages: Message[], 
    budget: number
  ): Promise<Message[]> {
    // 保留最近的消息
    const recentCount = 10;
    const recent = messages.slice(-recentCount);
    const recentTokens = this.countHistoryTokens(recent);
    
    if (recentTokens >= budget) {
      // 连最近的都放不下，截断工具结果
      return this.truncateToolResults(recent, budget);
    }
    
    // 总结早期对话
    const earlier = messages.slice(0, -recentCount);
    const summaryBudget = budget - recentTokens;
    const summary = await this.summarize(earlier, summaryBudget);
    
    return [
      { role: 'system', content: `[Earlier conversation summary]\n${summary}` },
      ...recent
    ];
  }
}
```

## 常见问题与解决方案

### 问题 1: 工具调用链过长

当 Agent 连续调用很多工具，每个结果都占用 context：

```typescript
// 解决方案：只保留最终结果，移除中间步骤
function compactToolChain(messages: Message[]): Message[] {
  const result: Message[] = [];
  let i = 0;
  
  while (i < messages.length) {
    const msg = messages[i];
    
    if (msg.role === 'assistant' && msg.tool_calls) {
      // 找到这个工具调用链的结束
      const chainEnd = findChainEnd(messages, i);
      
      // 只保留最后一个工具结果
      const lastToolResult = messages[chainEnd];
      result.push({
        role: 'system',
        content: `[Tool chain result: ${msg.tool_calls.map(t => t.function.name).join(' → ')}]\n${lastToolResult.content}`
      });
      
      i = chainEnd + 1;
    } else {
      result.push(msg);
      i++;
    }
  }
  
  return result;
}
```

### 问题 2: 大文件处理

读取大文件会撑爆 context：

```typescript
// 解决方案：渐进式读取
const read = tool({
  name: 'read',
  parameters: z.object({
    path: z.string(),
    offset: z.number().optional(),  // 从第几行开始
    limit: z.number().optional(),   // 读取多少行
  }),
  execute: async ({ path, offset = 0, limit = 200 }) => {
    const content = await fs.readFile(path, 'utf-8');
    const lines = content.split('\n');
    
    const totalLines = lines.length;
    const selectedLines = lines.slice(offset, offset + limit);
    
    return {
      content: selectedLines.join('\n'),
      totalLines,
      offset,
      hasMore: offset + limit < totalLines,
    };
  }
});
```

### 问题 3: 递归摘要

对话太长，摘要本身也很长：

```typescript
// 解决方案：层次化摘要
async function hierarchicalSummary(
  messages: Message[],
  targetTokens: number
): Promise<string> {
  // 分块
  const chunks = chunkMessages(messages, 50);
  
  // 每块摘要
  const chunkSummaries = await Promise.all(
    chunks.map(chunk => summarize(chunk, 500))
  );
  
  // 如果摘要总数还是太长，再摘要一次
  const totalSummaryTokens = countTokens(chunkSummaries.join('\n'));
  if (totalSummaryTokens > targetTokens) {
    return await summarize(chunkSummaries.join('\n\n'), targetTokens);
  }
  
  return chunkSummaries.join('\n\n');
}
```

## Token 计数库

几个常用的 token 计数工具：

```typescript
// 使用 tiktoken (OpenAI 官方)
import { encoding_for_model } from 'tiktoken';

const enc = encoding_for_model('gpt-4');
function countTokens(text: string): number {
  return enc.encode(text).length;
}

// 使用 @anthropic-ai/tokenizer (Claude)
import { countTokens } from '@anthropic-ai/tokenizer';

// 通用估算（快速但不精确）
function estimateTokens(text: string): number {
  // 英文约 4 字符一个 token，中文约 1.5 字符一个 token
  const englishChars = text.replace(/[\u4e00-\u9fff]/g, '').length;
  const chineseChars = (text.match(/[\u4e00-\u9fff]/g) || []).length;
  return Math.ceil(englishChars / 4 + chineseChars / 1.5);
}
```

## 最佳实践总结

1. **预留余量**: 不要用满 context，留 10-20% 给输出
2. **监控使用**: 每次调用前检查 token 使用情况
3. **智能截断**: 根据内容类型选择截断策略（代码保留头尾，日志保留最新）
4. **分层存储**: 热数据在 context，温数据按需加载，冷数据向量搜索
5. **增量处理**: 大文件分块读取，不要一次全加载
6. **定期压缩**: 长对话定期触发摘要，不要等到爆了再处理

## 练习

1. 实现一个 token 计数器，支持准确计数和快速估算两种模式
2. 设计一个对话压缩算法，保留关键信息同时压缩到目标大小
3. 实现分页读取工具，支持 offset/limit 参数

---

下一课我们将讲解 **Rate Limiting 与重试策略**，处理 API 限流和错误恢复。
