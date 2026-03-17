# Token Budget Management（Token 预算管理）

> 给 Agent 的"思考"设预算，像控制 API 成本一样控制推理质量与花费

---

## 为什么需要 Token 预算管理？

LLM 的每一次调用都有两个隐藏成本：

1. **延迟成本**：Token 越多，响应越慢
2. **金钱成本**：Input + Output token 都按量计费

Agent 在处理不同任务时，"思考深度"需求截然不同：

| 任务类型 | 所需思考深度 | 合适 Token 预算 |
|----------|------------|----------------|
| 回答简单问答 | 浅 | 0（关闭 thinking） |
| 生成结构化数据 | 中 | 1,000~3,000 |
| 复杂代码调试 | 深 | 5,000~10,000 |
| 多步骤推理规划 | 极深 | 10,000~20,000 |

**核心思路**：根据任务类型动态分配 `budget_tokens`，而不是一刀切地全开或全关。

---

## Claude Extended Thinking 的 Token 预算

Claude 3.7+ 支持 `thinking` 参数，允许你控制模型内部"思考链"的 Token 上限：

```python
# learn-claude-code 风格 - Python 示例
import anthropic

client = anthropic.Anthropic()

def call_with_budget(prompt: str, budget: int = 0):
    """
    budget=0: 关闭 thinking（最快最便宜）
    budget>0: 开启 extended thinking，上限 budget tokens
    """
    params = {
        "model": "claude-sonnet-4-5",
        "max_tokens": budget + 8192,  # max_tokens 必须 > budget_tokens
        "messages": [{"role": "user", "content": prompt}],
    }
    
    if budget > 0:
        params["thinking"] = {
            "type": "enabled",
            "budget_tokens": budget
        }
    
    return client.messages.create(**params)

# 简单任务 - 不用 thinking
response = call_with_budget("今天星期几？", budget=0)

# 复杂推理 - 给足预算
response = call_with_budget(
    "分析这段代码的时间复杂度并优化...", 
    budget=8000
)
```

---

## 自适应预算分配器

根据任务特征自动选择合适的预算：

```python
# adaptive_budget.py
from enum import Enum
from dataclasses import dataclass

class TaskComplexity(Enum):
    TRIVIAL = "trivial"      # 问候、简单查询
    SIMPLE = "simple"        # 格式化、翻译
    MODERATE = "moderate"    # 代码生成、分析
    COMPLEX = "complex"      # 多步推理、规划
    DEEP = "deep"            # 架构设计、调试复杂问题

@dataclass
class BudgetConfig:
    thinking_tokens: int
    max_output_tokens: int
    
BUDGET_MAP = {
    TaskComplexity.TRIVIAL:   BudgetConfig(0, 512),
    TaskComplexity.SIMPLE:    BudgetConfig(0, 2048),
    TaskComplexity.MODERATE:  BudgetConfig(3000, 4096),
    TaskComplexity.COMPLEX:   BudgetConfig(8000, 8192),
    TaskComplexity.DEEP:      BudgetConfig(16000, 16384),
}

def classify_task(prompt: str) -> TaskComplexity:
    """简单的关键词分类器（生产环境可用 LLM 来分类）"""
    prompt_lower = prompt.lower()
    
    deep_keywords = ["架构", "设计方案", "为什么", "debug", "调试", "优化算法"]
    complex_keywords = ["分析", "规划", "步骤", "实现", "代码"]
    moderate_keywords = ["生成", "写", "解释", "总结"]
    
    if any(k in prompt_lower for k in deep_keywords):
        return TaskComplexity.DEEP
    elif any(k in prompt_lower for k in complex_keywords):
        return TaskComplexity.COMPLEX
    elif any(k in prompt_lower for k in moderate_keywords):
        return TaskComplexity.MODERATE
    elif len(prompt) > 200:
        return TaskComplexity.SIMPLE
    else:
        return TaskComplexity.TRIVIAL

def smart_call(prompt: str) -> dict:
    complexity = classify_task(prompt)
    config = BUDGET_MAP[complexity]
    
    print(f"任务复杂度: {complexity.value}, thinking budget: {config.thinking_tokens}")
    
    params = {
        "model": "claude-sonnet-4-5",
        "max_tokens": config.max_output_tokens + config.thinking_tokens,
        "messages": [{"role": "user", "content": prompt}],
    }
    
    if config.thinking_tokens > 0:
        params["thinking"] = {
            "type": "enabled",
            "budget_tokens": config.thinking_tokens
        }
    
    return client.messages.create(**params)
```

---

## OpenClaw 的 Token 预算实现

OpenClaw 通过配置文件控制 thinking 行为，并且支持在 session 中动态切换：

```yaml
# ~/.openclaw/config.yaml (示意)
model:
  default: anthropic/claude-sonnet-4-6
  thinking:
    mode: adaptive  # off | on | stream | adaptive
    budget: 8000    # 默认 budget（tokens）
```

用户可以通过命令切换：
```
/reasoning on     # 开启 thinking，使用默认 budget
/reasoning off    # 关闭 thinking，节省 token
```

在代码层，OpenClaw 根据 session 类型决定是否注入 thinking 参数：

```typescript
// pi-mono 风格 - TypeScript 实现
interface ThinkingConfig {
  type: "enabled" | "disabled";
  budgetTokens?: number;
}

interface ModelCallOptions {
  model: string;
  maxTokens: number;
  thinking?: ThinkingConfig;
  messages: Message[];
}

class TokenBudgetManager {
  private dailyBudget: number;
  private usedTokens: number = 0;
  
  constructor(dailyBudget: number) {
    this.dailyBudget = dailyBudget;
  }
  
  allocateBudget(taskComplexity: "low" | "medium" | "high"): number {
    const remaining = this.dailyBudget - this.usedTokens;
    
    // 如果剩余预算不足，降级处理
    if (remaining < 1000) {
      console.warn("日预算不足，关闭 thinking");
      return 0;
    }
    
    const allocations = {
      low: 0,
      medium: Math.min(3000, remaining * 0.1),
      high: Math.min(10000, remaining * 0.3),
    };
    
    return Math.floor(allocations[taskComplexity]);
  }
  
  recordUsage(inputTokens: number, outputTokens: number, thinkingTokens: number) {
    // thinking tokens 也计入用量
    this.usedTokens += inputTokens + outputTokens + thinkingTokens;
    
    const remainingPercent = ((this.dailyBudget - this.usedTokens) / this.dailyBudget * 100).toFixed(1);
    console.log(`Token 使用: ${this.usedTokens}/${this.dailyBudget} (剩余 ${remainingPercent}%)`);
  }
  
  async callWithBudget(
    client: Anthropic,
    messages: Message[],
    complexity: "low" | "medium" | "high"
  ): Promise<MessageResponse> {
    const thinkingBudget = this.allocateBudget(complexity);
    
    const options: ModelCallOptions = {
      model: "claude-sonnet-4-5",
      maxTokens: thinkingBudget + 8192,
      messages,
    };
    
    if (thinkingBudget > 0) {
      options.thinking = {
        type: "enabled",
        budgetTokens: thinkingBudget,
      };
    }
    
    const response = await client.messages.create(options);
    
    // 记录实际使用量
    const usage = response.usage;
    this.recordUsage(
      usage.input_tokens,
      usage.output_tokens,
      (usage as any).cache_creation_input_tokens ?? 0
    );
    
    return response;
  }
}
```

---

## 成本对比实验

以 Claude Sonnet 为例（每百万 token 约 $3 input / $15 output）：

```python
import time

def benchmark_budget_impact(prompt: str):
    results = {}
    
    for budget in [0, 1000, 5000, 10000]:
        start = time.time()
        response = call_with_budget(prompt, budget)
        elapsed = time.time() - start
        
        usage = response.usage
        # thinking tokens 按 output 价格计费
        cost = (usage.input_tokens * 3 + usage.output_tokens * 15) / 1_000_000
        
        results[budget] = {
            "latency_s": round(elapsed, 2),
            "input_tokens": usage.input_tokens,
            "output_tokens": usage.output_tokens,
            "est_cost_usd": round(cost, 5),
        }
        
        print(f"Budget {budget:6d}: {elapsed:.2f}s, cost=${cost:.5f}")
    
    return results

# 示例输出（复杂推理任务）：
# Budget      0: 1.23s, cost=$0.00089   ← 关闭 thinking，快但质量低
# Budget   1000: 2.45s, cost=$0.00134   ← 少量思考
# Budget   5000: 4.67s, cost=$0.00312   ← 中等深度
# Budget  10000: 8.91s, cost=$0.00587   ← 深度推理，质量最高
```

---

## 关键陷阱

### 1. `max_tokens` 必须大于 `budget_tokens`

```python
# ❌ 错误：max_tokens < budget_tokens + 预期输出
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1000,
    thinking={"type": "enabled", "budget_tokens": 5000},  # 会报错！
    messages=[...]
)

# ✅ 正确：留足输出空间
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=5000 + 8192,  # budget + 期望输出
    thinking={"type": "enabled", "budget_tokens": 5000},
    messages=[...]
)
```

### 2. Thinking Token 也计费

很多人以为 thinking 是"免费的内部过程"——**不是的**！

```
实际费用 = input_tokens × $3 + (thinking_tokens + output_tokens) × $15
```

### 3. Streaming 时 Thinking Block 先返回

```python
with client.messages.stream(
    model="claude-sonnet-4-5",
    max_tokens=16384,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[{"role": "user", "content": "..."}]
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            if event.content_block.type == "thinking":
                print("🤔 模型正在思考...")
            elif event.content_block.type == "text":
                print("💬 正在输出结果...")
        elif event.type == "text":
            print(event.text, end="", flush=True)
```

---

## 实践建议

| 场景 | 推荐策略 |
|------|---------|
| 高并发 API 服务 | 默认关闭 thinking，仅对标记为"复杂"的请求开启 |
| 个人助理 Agent | 中等预算（3000~8000），按需升级 |
| 代码调试/分析 | 高预算（8000~16000），质量优先 |
| 日常对话 | 关闭 thinking，延迟优先 |
| 有日预算限制 | 用 TokenBudgetManager 动态降级 |

---

## 总结

Token 预算管理的本质是**资源分配问题**：

1. **分类任务** → 判断所需推理深度
2. **分配预算** → 为不同类型设合理上限
3. **记录用量** → 实时跟踪日/月预算消耗
4. **动态降级** → 预算不足时自动关闭 thinking

用好 `budget_tokens`，可以在**质量与成本之间找到最优平衡点**——而不是只能在"全开"和"全关"之间二选一。

---

*下一课预告：Agent Dependency Injection（依赖注入模式）— 让 Agent 的工具、记忆、配置可插拔替换*
