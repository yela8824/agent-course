# 128 - Agent 协议适配器模式（Protocol Adapter Pattern）

> 让 Agent 代码与 LLM Provider 解耦，随时切换 Anthropic / OpenAI / Gemini 不改一行业务逻辑。

---

## 为什么需要协议适配器？

你写好了一个 Agent，跑在 Anthropic Claude 上；老板突然说"切 GPT-4o"，或者 Claude 限速了要临时切 Gemini——如果消息格式、工具调用格式、响应格式都硬编码在 Agent 里，这就是一场噩梦。

三家主流 Provider 的格式对比：

| 维度 | Anthropic | OpenAI | Gemini |
|------|-----------|--------|--------|
| 消息角色 | `user/assistant` | `user/assistant/system` | `user/model` |
| 工具定义 | `tools[]` + `input_schema` | `functions[]` + `parameters` | `functionDeclarations[]` |
| 工具调用 | `tool_use` block | `function_call` in message | `functionCall` part |
| 工具结果 | `tool_result` block | `function` role message | `functionResponse` part |
| 停止原因 | `end_turn` / `tool_use` | `stop` / `function_call` | `STOP` / `FUNCTION_CALL` |

不统一 = 痛苦。适配器模式解决这个问题。

---

## 核心架构

```
Agent Core (业务逻辑)
      ↓
[统一内部格式 NormalizedMessage]
      ↓
   Adapter Layer
   ┌────────────────────────────┐
   │ AnthropicAdapter           │
   │ OpenAIAdapter              │
   │ GeminiAdapter              │
   └────────────────────────────┘
      ↓
LLM Provider API
```

Agent 核心只操作 `NormalizedMessage`，Adapter 负责双向翻译。

---

## 统一内部格式定义

```typescript
// types/normalized.ts

export type MessageRole = 'user' | 'assistant' | 'system' | 'tool_result';

export interface NormalizedMessage {
  role: MessageRole;
  content: NormalizedContent[];
}

export type NormalizedContent =
  | { type: 'text'; text: string }
  | { type: 'image'; url?: string; base64?: string; mimeType: string }
  | { type: 'tool_call'; id: string; name: string; input: Record<string, unknown> }
  | { type: 'tool_result'; toolCallId: string; content: string; isError?: boolean };

export interface NormalizedTool {
  name: string;
  description: string;
  inputSchema: {
    type: 'object';
    properties: Record<string, unknown>;
    required?: string[];
  };
}

export interface NormalizedResponse {
  messages: NormalizedMessage[];
  stopReason: 'end_turn' | 'tool_use' | 'max_tokens' | 'stop';
  usage: { inputTokens: number; outputTokens: number };
  model: string;
}

export interface LLMAdapter {
  readonly providerId: string;
  toProviderMessages(messages: NormalizedMessage[]): unknown[];
  toProviderTools(tools: NormalizedTool[]): unknown[];
  fromProviderResponse(raw: unknown): NormalizedResponse;
}
```

---

## Anthropic 适配器

```typescript
// adapters/anthropic.ts
import Anthropic from '@anthropic-ai/sdk';
import type { LLMAdapter, NormalizedMessage, NormalizedTool, NormalizedResponse } from '../types/normalized';

export class AnthropicAdapter implements LLMAdapter {
  readonly providerId = 'anthropic';

  toProviderMessages(messages: NormalizedMessage[]): Anthropic.MessageParam[] {
    const result: Anthropic.MessageParam[] = [];

    // Anthropic 不支持 system role in messages array（要单独传），过滤掉
    for (const msg of messages) {
      if (msg.role === 'system') continue;

      const content: Anthropic.ContentBlock[] = [];

      for (const c of msg.content) {
        if (c.type === 'text') {
          content.push({ type: 'text', text: c.text });
        } else if (c.type === 'tool_call') {
          content.push({
            type: 'tool_use',
            id: c.id,
            name: c.name,
            input: c.input,
          });
        } else if (c.type === 'tool_result') {
          content.push({
            type: 'tool_result',
            tool_use_id: c.toolCallId,
            content: c.content,
            is_error: c.isError,
          });
        }
      }

      // Anthropic 要求 tool_result 在 user 消息里
      const role = msg.role === 'tool_result' ? 'user' : msg.role as 'user' | 'assistant';
      result.push({ role, content });
    }

    return result;
  }

  toProviderTools(tools: NormalizedTool[]): Anthropic.Tool[] {
    return tools.map(t => ({
      name: t.name,
      description: t.description,
      input_schema: t.inputSchema,
    }));
  }

  fromProviderResponse(raw: Anthropic.Message): NormalizedResponse {
    const contents = raw.content.map(block => {
      if (block.type === 'text') {
        return { type: 'text' as const, text: block.text };
      } else if (block.type === 'tool_use') {
        return {
          type: 'tool_call' as const,
          id: block.id,
          name: block.name,
          input: block.input as Record<string, unknown>,
        };
      }
      throw new Error(`Unknown block type: ${(block as any).type}`);
    });

    const stopReasonMap: Record<string, NormalizedResponse['stopReason']> = {
      end_turn: 'end_turn',
      tool_use: 'tool_use',
      max_tokens: 'max_tokens',
    };

    return {
      messages: [{ role: 'assistant', content: contents }],
      stopReason: stopReasonMap[raw.stop_reason ?? 'end_turn'] ?? 'end_turn',
      usage: { inputTokens: raw.usage.input_tokens, outputTokens: raw.usage.output_tokens },
      model: raw.model,
    };
  }
}
```

---

## OpenAI 适配器

```typescript
// adapters/openai.ts
import type OpenAI from 'openai';
import type { LLMAdapter, NormalizedMessage, NormalizedTool, NormalizedResponse } from '../types/normalized';

export class OpenAIAdapter implements LLMAdapter {
  readonly providerId = 'openai';

  toProviderMessages(messages: NormalizedMessage[]): OpenAI.ChatCompletionMessageParam[] {
    const result: OpenAI.ChatCompletionMessageParam[] = [];

    for (const msg of messages) {
      if (msg.role === 'system') {
        const text = msg.content.find(c => c.type === 'text')?.text ?? '';
        result.push({ role: 'system', content: text });
        continue;
      }

      if (msg.role === 'tool_result') {
        // OpenAI: tool 结果是独立的 role=tool 消息
        for (const c of msg.content) {
          if (c.type === 'tool_result') {
            result.push({
              role: 'tool',
              tool_call_id: c.toolCallId,
              content: c.content,
            });
          }
        }
        continue;
      }

      const toolCalls = msg.content
        .filter(c => c.type === 'tool_call')
        .map(c => {
          if (c.type !== 'tool_call') return null!;
          return {
            id: c.id,
            type: 'function' as const,
            function: { name: c.name, arguments: JSON.stringify(c.input) },
          };
        });

      const textContent = msg.content
        .filter(c => c.type === 'text')
        .map(c => c.type === 'text' ? c.text : '')
        .join('');

      if (msg.role === 'assistant') {
        result.push({
          role: 'assistant',
          content: textContent || null,
          tool_calls: toolCalls.length > 0 ? toolCalls : undefined,
        });
      } else {
        result.push({ role: 'user', content: textContent });
      }
    }

    return result;
  }

  toProviderTools(tools: NormalizedTool[]): OpenAI.ChatCompletionTool[] {
    return tools.map(t => ({
      type: 'function' as const,
      function: {
        name: t.name,
        description: t.description,
        parameters: t.inputSchema,
      },
    }));
  }

  fromProviderResponse(raw: OpenAI.ChatCompletion): NormalizedResponse {
    const choice = raw.choices[0];
    const msg = choice.message;

    const contents = [];

    if (msg.content) {
      contents.push({ type: 'text' as const, text: msg.content });
    }

    for (const tc of msg.tool_calls ?? []) {
      contents.push({
        type: 'tool_call' as const,
        id: tc.id,
        name: tc.function.name,
        input: JSON.parse(tc.function.arguments) as Record<string, unknown>,
      });
    }

    const stopReasonMap: Record<string, NormalizedResponse['stopReason']> = {
      stop: 'end_turn',
      tool_calls: 'tool_use',
      length: 'max_tokens',
    };

    return {
      messages: [{ role: 'assistant', content: contents }],
      stopReason: stopReasonMap[choice.finish_reason ?? 'stop'] ?? 'end_turn',
      usage: {
        inputTokens: raw.usage?.prompt_tokens ?? 0,
        outputTokens: raw.usage?.completion_tokens ?? 0,
      },
      model: raw.model,
    };
  }
}
```

---

## 统一 LLM Client

```typescript
// llm-client.ts
import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai';
import type { LLMAdapter, NormalizedMessage, NormalizedTool, NormalizedResponse } from './types/normalized';
import { AnthropicAdapter } from './adapters/anthropic';
import { OpenAIAdapter } from './adapters/openai';

interface ClientConfig {
  provider: 'anthropic' | 'openai';
  model: string;
  systemPrompt?: string;
  maxTokens?: number;
}

export class LLMClient {
  private adapter: LLMAdapter;
  private config: ClientConfig;

  // 底层 SDK 实例
  private anthropic?: Anthropic;
  private openai?: OpenAI;

  constructor(config: ClientConfig) {
    this.config = config;

    if (config.provider === 'anthropic') {
      this.adapter = new AnthropicAdapter();
      this.anthropic = new Anthropic();
    } else if (config.provider === 'openai') {
      this.adapter = new OpenAIAdapter();
      this.openai = new OpenAI();
    } else {
      throw new Error(`Unknown provider: ${config.provider}`);
    }
  }

  async call(
    messages: NormalizedMessage[],
    tools: NormalizedTool[] = [],
  ): Promise<NormalizedResponse> {
    const providerMessages = this.adapter.toProviderMessages(messages);
    const providerTools = this.adapter.toProviderTools(tools);

    if (this.config.provider === 'anthropic') {
      const raw = await this.anthropic!.messages.create({
        model: this.config.model,
        max_tokens: this.config.maxTokens ?? 4096,
        system: this.config.systemPrompt,
        messages: providerMessages as any,
        tools: providerTools.length > 0 ? providerTools as any : undefined,
      });
      return this.adapter.fromProviderResponse(raw);
    }

    if (this.config.provider === 'openai') {
      const msgs = this.config.systemPrompt
        ? [{ role: 'system' as const, content: this.config.systemPrompt }, ...providerMessages as any]
        : providerMessages;

      const raw = await this.openai!.chat.completions.create({
        model: this.config.model,
        max_tokens: this.config.maxTokens ?? 4096,
        messages: msgs,
        tools: providerTools.length > 0 ? providerTools as any : undefined,
      });
      return this.adapter.fromProviderResponse(raw);
    }

    throw new Error('Unreachable');
  }
}
```

---

## Agent Loop：完全 Provider 无关

```typescript
// agent.ts
import { LLMClient } from './llm-client';
import type { NormalizedMessage, NormalizedTool, NormalizedContent } from './types/normalized';

// ✅ 这段 Agent 代码完全不知道底层是 Anthropic 还是 OpenAI
export async function runAgent(
  userMessage: string,
  client: LLMClient,
  tools: NormalizedTool[],
  toolHandlers: Record<string, (input: Record<string, unknown>) => Promise<string>>,
): Promise<string> {
  const messages: NormalizedMessage[] = [
    { role: 'user', content: [{ type: 'text', text: userMessage }] },
  ];

  while (true) {
    const response = await client.call(messages, tools);

    // 把 assistant 消息追加到历史
    messages.push(...response.messages);

    if (response.stopReason === 'end_turn') {
      // 提取最后一条文本
      const lastMsg = response.messages.at(-1);
      const text = lastMsg?.content.find(c => c.type === 'text');
      return text?.type === 'text' ? text.text : '';
    }

    if (response.stopReason === 'tool_use') {
      // 处理所有工具调用
      const toolResults: NormalizedContent[] = [];

      for (const msg of response.messages) {
        for (const block of msg.content) {
          if (block.type !== 'tool_call') continue;

          const handler = toolHandlers[block.name];
          if (!handler) {
            toolResults.push({
              type: 'tool_result',
              toolCallId: block.id,
              content: `Error: unknown tool ${block.name}`,
              isError: true,
            });
            continue;
          }

          try {
            const result = await handler(block.input);
            toolResults.push({
              type: 'tool_result',
              toolCallId: block.id,
              content: result,
            });
          } catch (err) {
            toolResults.push({
              type: 'tool_result',
              toolCallId: block.id,
              content: String(err),
              isError: true,
            });
          }
        }
      }

      // 工具结果作为 user 消息（Anthropic 约定）/ tool 消息（OpenAI）
      // Adapter 负责转换，Agent 统一用 tool_result role
      messages.push({ role: 'tool_result', content: toolResults });
      continue;
    }

    break;
  }

  return '';
}
```

---

## 使用示例：一键切换 Provider

```typescript
// main.ts
import { LLMClient } from './llm-client';
import { runAgent } from './agent';
import type { NormalizedTool } from './types/normalized';

const tools: NormalizedTool[] = [{
  name: 'get_weather',
  description: '获取指定城市的天气',
  inputSchema: {
    type: 'object',
    properties: {
      city: { type: 'string', description: '城市名' },
    },
    required: ['city'],
  },
}];

const handlers = {
  get_weather: async ({ city }: Record<string, unknown>) => {
    return `${city}: 晴，25°C`;
  },
};

// 用 Anthropic
const anthropicClient = new LLMClient({
  provider: 'anthropic',
  model: 'claude-opus-4-5',
  systemPrompt: '你是一个天气助手。',
});

// 用 OpenAI —— Agent 代码零改动
const openaiClient = new LLMClient({
  provider: 'openai',
  model: 'gpt-4o',
  systemPrompt: '你是一个天气助手。',
});

const answer = await runAgent('北京今天天气怎么样？', anthropicClient, tools, handlers);
console.log(answer);
```

---

## OpenClaw 的实现：multi-model fallback 中的适配

在 OpenClaw 的 `multi-model fallback chain`（第91课）中，切换 Provider 时就依赖类似的格式规范化：

```typescript
// OpenClaw 内部：每个 Provider 注册自己的适配器
const adapterRegistry = new Map<string, LLMAdapter>([
  ['anthropic', new AnthropicAdapter()],
  ['openai', new OpenAIAdapter()],
]);

class ModelRouter {
  async route(request: NormalizedRequest): Promise<NormalizedResponse> {
    const provider = this.selectProvider(request);
    const adapter = adapterRegistry.get(provider)!;
    // ... 调用 + 适配
  }
}
```

pi-mono 的 `ModelRouter` 同理：所有 Provider 实现统一接口，业务层只用一套 API。

---

## 关键设计原则

1. **单向数据流**：外部格式 → 内部格式 → 外部格式，绝不跨层混用
2. **适配器无状态**：Adapter 只做格式转换，不保存任何状态
3. **内部格式最大公约数**：不要为了某个 Provider 的特性污染 `NormalizedMessage`
4. **Provider 特性用扩展点**：`LLMClient.call()` 可传 `providerOptions` 透传原始参数
5. **测试**：每个 Adapter 独立单测，Agent Loop 用 mock Adapter

---

## 总结

| 没有适配器 | 有适配器 |
|-----------|---------|
| Agent 代码散落 `if openai ... else if anthropic ...` | Agent 只操作 `NormalizedMessage` |
| 切换 Provider = 重写 | 切换 Provider = 改一行配置 |
| 测试困难 | Mock Adapter 轻松隔离 |
| 新加 Provider 改 Agent 核心 | 新加 Provider 只加一个 Adapter 类 |

协议适配器是 Agent 工程化的基础设施——写一次，Provider 随意换。
