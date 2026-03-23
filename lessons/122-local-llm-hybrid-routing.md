# 第122讲 - Agent 本地大模型集成与混合路由（Local LLM Integration & Hybrid Routing）

## 🎯 核心思想

不是所有任务都需要 GPT-4 / Claude。

把任务按**成本 × 隐私 × 复杂度**三个维度分层：
- 简单分类、格式化、摘要 → **本地模型**（Ollama，零成本、零延迟、零数据泄露）
- 复杂推理、代码生成、多步规划 → **云端模型**（Anthropic / OpenAI）
- 含 PII 数据的任务 → **强制走本地**，不上传

这就是混合路由（Hybrid Routing）的核心价值。

---

## 🏗️ 架构图

```
用户请求
   │
   ▼
[路由决策层] ──────────────────────────────────────────
   │                    │                    │
   ▼                    ▼                    ▼
本地模型层         云端模型层           强制本地（PII）
(Ollama)        (Anthropic/OAI)      (隐私护栏触发)
llama3.2:3b     claude-sonnet-4      gemma2:2b
qwen2.5:7b      gpt-4o               phi3:mini
   │                    │                    │
   └────────────────────┴────────────────────┘
                        │
                   统一响应格式
```

---

## 📦 实现代码

### 1. 路由配置表

```typescript
// routing-config.ts
export interface ModelTier {
  id: string;
  provider: 'ollama' | 'anthropic' | 'openai';
  model: string;
  maxTokens: number;
  costPer1kTokens: number; // USD
  avgLatencyMs: number;
  capabilities: string[];
}

export const MODEL_TIERS: ModelTier[] = [
  {
    id: 'local-fast',
    provider: 'ollama',
    model: 'llama3.2:3b',
    maxTokens: 4096,
    costPer1kTokens: 0,
    avgLatencyMs: 200,
    capabilities: ['classify', 'format', 'extract', 'summarize-short'],
  },
  {
    id: 'local-smart',
    provider: 'ollama',
    model: 'qwen2.5:7b',
    maxTokens: 8192,
    costPer1kTokens: 0,
    avgLatencyMs: 800,
    capabilities: ['summarize', 'translate', 'qa-simple', 'code-review-light'],
  },
  {
    id: 'cloud-standard',
    provider: 'anthropic',
    model: 'claude-haiku-3-5',
    maxTokens: 8192,
    costPer1kTokens: 0.001,
    avgLatencyMs: 1200,
    capabilities: ['qa', 'code', 'analyze', 'write'],
  },
  {
    id: 'cloud-premium',
    provider: 'anthropic',
    model: 'claude-sonnet-4',
    maxTokens: 64000,
    costPer1kTokens: 0.003,
    avgLatencyMs: 2500,
    capabilities: ['complex-reasoning', 'multi-step-plan', 'code-architect', 'creative'],
  },
];
```

### 2. 路由决策器

```typescript
// hybrid-router.ts
import Anthropic from '@anthropic-ai/sdk';

interface RoutingContext {
  task: string;
  inputText: string;
  hasPII?: boolean;          // 是否含敏感数据
  maxBudgetUSD?: number;     // 预算上限
  maxLatencyMs?: number;     // 延迟上限
  requiresToolUse?: boolean; // 是否需要工具调用
}

interface RoutingDecision {
  tier: ModelTier;
  reason: string;
  estimatedCost: number;
}

export class HybridRouter {
  private piiPatterns = [
    /\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b/, // 信用卡
    /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}\b/i, // 邮箱
    /\b1[3-9]\d{9}\b/,  // 中国手机号
  ];

  route(ctx: RoutingContext): RoutingDecision {
    const inputLen = ctx.inputText.length;
    const estimatedInputTokens = Math.ceil(inputLen / 3.5);

    // ① 强制本地：含 PII
    if (ctx.hasPII || this.detectPII(ctx.inputText)) {
      const tier = MODEL_TIERS.find(t => t.id === 'local-smart')!;
      return {
        tier,
        reason: 'PII detected - forced local for privacy',
        estimatedCost: 0,
      };
    }

    // ② 强制本地：预算为零
    if (ctx.maxBudgetUSD === 0) {
      const tier = MODEL_TIERS.find(t => t.id === 'local-fast')!;
      return { tier, reason: 'zero budget', estimatedCost: 0 };
    }

    // ③ 工具调用 → 云端（本地模型 function calling 不稳定）
    if (ctx.requiresToolUse) {
      const tier = MODEL_TIERS.find(t => t.id === 'cloud-standard')!;
      const cost = (estimatedInputTokens / 1000) * tier.costPer1kTokens;
      return { tier, reason: 'tool use required', estimatedCost: cost };
    }

    // ④ 简单任务 → 本地快速
    const simpleTaskPatterns = [
      /^classify/i, /^categorize/i, /^extract/i, /^format/i,
      /^翻译成/, /^格式化/, /^分类/,
    ];
    if (simpleTaskPatterns.some(p => p.test(ctx.task)) && inputLen < 2000) {
      const tier = MODEL_TIERS.find(t => t.id === 'local-fast')!;
      return { tier, reason: 'simple task + short input', estimatedCost: 0 };
    }

    // ⑤ 中等任务 → 本地智能
    if (inputLen < 5000 && !ctx.task.includes('complex')) {
      const tier = MODEL_TIERS.find(t => t.id === 'local-smart')!;
      return { tier, reason: 'medium task, local model sufficient', estimatedCost: 0 };
    }

    // ⑥ 默认 → 云端标准
    const tier = MODEL_TIERS.find(t => t.id === 'cloud-standard')!;
    const cost = (estimatedInputTokens / 1000) * tier.costPer1kTokens;
    return { tier, reason: 'default cloud routing', estimatedCost: cost };
  }

  private detectPII(text: string): boolean {
    return this.piiPatterns.some(p => p.test(text));
  }
}
```

### 3. 统一模型调用接口

```typescript
// unified-llm-client.ts
import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai'; // Ollama 兼容 OpenAI API 格式

export class UnifiedLLMClient {
  private anthropic = new Anthropic();
  // Ollama 本地服务，复用 OpenAI SDK
  private ollama = new OpenAI({
    baseURL: 'http://localhost:11434/v1',
    apiKey: 'ollama', // 占位，本地无需真实 key
  });

  async complete(
    tier: ModelTier,
    messages: Array<{ role: 'user' | 'assistant'; content: string }>,
    systemPrompt?: string,
  ): Promise<string> {
    if (tier.provider === 'ollama') {
      return this.callOllama(tier.model, messages, systemPrompt);
    } else if (tier.provider === 'anthropic') {
      return this.callAnthropic(tier.model, messages, systemPrompt);
    }
    throw new Error(`Unknown provider: ${tier.provider}`);
  }

  private async callOllama(
    model: string,
    messages: Array<{ role: 'user' | 'assistant'; content: string }>,
    systemPrompt?: string,
  ): Promise<string> {
    const allMessages = systemPrompt
      ? [{ role: 'system' as const, content: systemPrompt }, ...messages]
      : messages;

    const response = await this.ollama.chat.completions.create({
      model,
      messages: allMessages,
      stream: false,
    });

    return response.choices[0].message.content ?? '';
  }

  private async callAnthropic(
    model: string,
    messages: Array<{ role: 'user' | 'assistant'; content: string }>,
    systemPrompt?: string,
  ): Promise<string> {
    const response = await this.anthropic.messages.create({
      model,
      max_tokens: 4096,
      system: systemPrompt,
      messages,
    });

    const block = response.content[0];
    return block.type === 'text' ? block.text : '';
  }
}
```

### 4. 完整 Agent 集成

```typescript
// hybrid-agent.ts
export class HybridAgent {
  private router = new HybridRouter();
  private client = new UnifiedLLMClient();
  private stats = { localCalls: 0, cloudCalls: 0, totalSaved: 0 };

  async run(task: string, input: string, options: Partial<RoutingContext> = {}) {
    // 路由决策
    const decision = this.router.route({
      task,
      inputText: input,
      ...options,
    });

    console.log(`[Router] → ${decision.tier.id} | ${decision.reason}`);
    console.log(`[Cost] ~$${decision.estimatedCost.toFixed(4)}`);

    // 记录统计
    if (decision.tier.provider === 'ollama') {
      this.stats.localCalls++;
      // 估算节省（假设原本用 cloud-standard）
      const cloudTier = MODEL_TIERS.find(t => t.id === 'cloud-standard')!;
      this.stats.totalSaved += (input.length / 3500) * cloudTier.costPer1kTokens;
    } else {
      this.stats.cloudCalls++;
    }

    // 执行推理
    const result = await this.client.complete(
      decision.tier,
      [{ role: 'user', content: input }],
      `You are a helpful assistant. Task type: ${task}`,
    );

    return { result, tier: decision.tier.id, reason: decision.reason };
  }

  getStats() {
    return {
      ...this.stats,
      localRatio: this.stats.localCalls / (this.stats.localCalls + this.stats.cloudCalls),
      totalSavedUSD: this.stats.totalSaved,
    };
  }
}

// 使用示例
const agent = new HybridAgent();

// 简单分类 → 走本地
await agent.run('classify sentiment', '这个产品真的很棒！');
// → local-fast | simple task + short input

// 含手机号 → 强制本地
await agent.run('analyze', '用户 13812345678 的订单有问题');
// → local-smart | PII detected - forced local for privacy

// 复杂代码任务 → 云端
await agent.run('complex code architect', '设计一个分布式任务调度系统...');
// → cloud-standard | default cloud routing
```

---

## 🔧 OpenClaw 实战：技能中的混合路由

在 OpenClaw Skills 里，可以通过 `session_status` 或任务上下文动态选择模型：

```typescript
// 在 SKILL.md 或工具里调用不同模型
// 轻量任务 → claude-haiku-3-5（通过 session_status model 参数）
// 重度任务 → claude-sonnet-4（默认）

// OpenClaw 支持 per-session model override:
// session_status({ model: "anthropic/claude-haiku-3-5" })
```

---

## 📊 成本对比示例

| 场景 | 请求量/天 | 纯云端成本 | 混合路由成本 | 节省 |
|------|----------|-----------|------------|------|
| 客服分类 | 10,000 | $30/天 | $3/天 | **90%** |
| 内容摘要 | 5,000 | $25/天 | $5/天 | **80%** |
| 代码审查 | 500 | $15/天 | $12/天 | **20%** |

---

## ⚡ 快速启动 Ollama

```bash
# 安装
brew install ollama  # macOS
# 或: curl -fsSL https://ollama.ai/install.sh | sh

# 拉取模型
ollama pull llama3.2:3b    # 2GB，极速
ollama pull qwen2.5:7b     # 5GB，中文更好
ollama pull phi3:mini      # 2.3GB，代码强

# 启动服务（默认 localhost:11434）
ollama serve

# 验证 API
curl http://localhost:11434/v1/models
```

---

## 🎓 核心要点

1. **Ollama 兼容 OpenAI API** → 换个 baseURL 即可复用现有代码
2. **PII 检测强制本地** → 合规的第一道防线，比任何隐私政策都可靠
3. **工具调用走云端** → 本地模型 function calling 质量参差不齐，别踩坑
4. **统计节省成本** → 给老板看数据，ROI 直观
5. **渐进式迁移** → 先把简单任务切本地，复杂任务保持云端，稳定后再扩大

---

## 🔗 参考

- [Ollama 官网](https://ollama.ai)
- [Ollama OpenAI 兼容文档](https://github.com/ollama/ollama/blob/main/docs/openai.md)
- [pi-mono ModelRouter](https://github.com/gfwfail/agent-course) — 云端多模型路由实现
- 第14讲：Model Routing 模型路由（云端路由基础）
- 第93讲：PII Masking & Privacy Guards（隐私护栏）
