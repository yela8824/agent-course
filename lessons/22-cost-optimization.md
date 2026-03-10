# Cost Optimization：Agent 成本优化策略

> 第 22 课：如何让你的 Agent 既聪明又省钱

## 为什么成本优化很重要？

AI Agent 的成本主要来自 LLM API 调用。以 Claude 为例：
- Opus: $15/M input, $75/M output
- Sonnet: $3/M input, $15/M output
- Haiku: $0.25/M input, $1.25/M output

一个活跃的 Agent 每天可能消耗数万 tokens，如果不优化，成本会快速失控。

## 核心优化策略

### 1. Token 预算控制

**learn-claude-code 实现：**

```python
class TokenBudget:
    """Token 预算管理器"""
    
    def __init__(self, daily_limit: int = 1_000_000, alert_threshold: float = 0.8):
        self.daily_limit = daily_limit
        self.alert_threshold = alert_threshold
        self.usage_today = 0
        self.reset_date = datetime.now().date()
    
    def check_and_update(self, input_tokens: int, output_tokens: int) -> bool:
        """检查并更新使用量，返回是否允许继续"""
        # 日期变化时重置
        if datetime.now().date() > self.reset_date:
            self.usage_today = 0
            self.reset_date = datetime.now().date()
        
        total = input_tokens + output_tokens * 3  # Output 更贵，加权计算
        
        if self.usage_today + total > self.daily_limit:
            return False  # 超预算
        
        self.usage_today += total
        
        # 达到阈值时告警
        if self.usage_today / self.daily_limit > self.alert_threshold:
            self._send_alert()
        
        return True
    
    def get_remaining(self) -> int:
        return max(0, self.daily_limit - self.usage_today)
```

**实际应用：**

```python
class CostAwareAgent:
    def __init__(self):
        self.budget = TokenBudget(daily_limit=500_000)
        self.model_tiers = ["opus", "sonnet", "haiku"]
        self.current_tier = 0
    
    async def run(self, task: str) -> str:
        # 预算紧张时自动降级模型
        if self.budget.get_remaining() < 100_000:
            self.current_tier = min(self.current_tier + 1, 2)
        
        model = self.model_tiers[self.current_tier]
        
        response = await self.call_llm(task, model=model)
        
        # 更新预算
        if not self.budget.check_and_update(
            response.input_tokens, 
            response.output_tokens
        ):
            raise BudgetExceededError("Daily budget exceeded")
        
        return response.content
```

### 2. 智能模型降级

不是所有任务都需要最强模型。根据任务复杂度选择合适的模型：

**pi-mono 的实现思路：**

```typescript
interface TaskClassification {
  complexity: 'simple' | 'medium' | 'complex';
  requiresReasoning: boolean;
  requiresCodeGen: boolean;
}

function selectModel(task: TaskClassification): ModelConfig {
  // 简单任务：Haiku 足够
  if (task.complexity === 'simple' && !task.requiresReasoning) {
    return { model: 'claude-3-haiku', thinking: 'none' };
  }
  
  // 需要推理但不复杂：Sonnet
  if (task.complexity === 'medium' || task.requiresCodeGen) {
    return { model: 'claude-sonnet-4', thinking: 'low' };
  }
  
  // 复杂任务：Opus
  return { model: 'claude-opus-4', thinking: 'high' };
}

// 任务分类器（用小模型来分类）
async function classifyTask(task: string): Promise<TaskClassification> {
  const response = await callLLM({
    model: 'claude-3-haiku',  // 用最便宜的模型做分类
    messages: [{
      role: 'user',
      content: `Classify this task's complexity (simple/medium/complex):
      Task: ${task}
      
      Output JSON: {"complexity": "...", "requiresReasoning": bool, "requiresCodeGen": bool}`
    }]
  });
  
  return JSON.parse(response.content);
}
```

### 3. Prompt 压缩

长 prompt 意味着更多 tokens。压缩技巧：

```python
class PromptCompressor:
    """智能 Prompt 压缩"""
    
    def __init__(self, max_context_tokens: int = 50000):
        self.max_tokens = max_context_tokens
    
    def compress_conversation(self, messages: list[Message]) -> list[Message]:
        """压缩对话历史"""
        total_tokens = sum(self.count_tokens(m.content) for m in messages)
        
        if total_tokens <= self.max_tokens:
            return messages
        
        compressed = []
        
        # 保留系统消息和最近的交互
        system_msgs = [m for m in messages if m.role == 'system']
        recent_msgs = messages[-6:]  # 最近 3 轮对话
        
        compressed.extend(system_msgs)
        
        # 中间部分做摘要
        middle_msgs = messages[len(system_msgs):-6]
        if middle_msgs:
            summary = self._summarize(middle_msgs)
            compressed.append(Message(
                role='system',
                content=f"[Previous conversation summary: {summary}]"
            ))
        
        compressed.extend(recent_msgs)
        return compressed
    
    def compress_tool_results(self, result: str, max_chars: int = 5000) -> str:
        """压缩工具执行结果"""
        if len(result) <= max_chars:
            return result
        
        # 保留开头和结尾，中间截断
        head = result[:max_chars // 2]
        tail = result[-max_chars // 2:]
        
        return f"{head}\n\n... [truncated {len(result) - max_chars} chars] ...\n\n{tail}"
```

### 4. 缓存策略

重复的查询不需要重复调用 LLM：

```python
import hashlib
from functools import lru_cache

class ResponseCache:
    """LLM 响应缓存"""
    
    def __init__(self, redis_client=None):
        self.redis = redis_client
        self.local_cache = {}  # 内存缓存
    
    def _make_key(self, messages: list, model: str) -> str:
        """生成缓存 key"""
        content = f"{model}:{json.dumps(messages, sort_keys=True)}"
        return hashlib.sha256(content.encode()).hexdigest()
    
    async def get_or_call(
        self, 
        messages: list, 
        model: str,
        call_fn: Callable
    ) -> str:
        key = self._make_key(messages, model)
        
        # 先查缓存
        cached = self.local_cache.get(key) or (
            await self.redis.get(key) if self.redis else None
        )
        
        if cached:
            return cached
        
        # 缓存未命中，调用 LLM
        response = await call_fn()
        
        # 存入缓存（设置 TTL）
        self.local_cache[key] = response
        if self.redis:
            await self.redis.setex(key, 3600, response)  # 1 小时过期
        
        return response


# 使用示例
cache = ResponseCache()

async def smart_call(messages, model):
    return await cache.get_or_call(
        messages=messages,
        model=model,
        call_fn=lambda: anthropic.messages.create(model=model, messages=messages)
    )
```

### 5. 批处理优化

多个独立请求可以并行处理，减少总时间成本：

```python
import asyncio
from typing import List

async def batch_process(tasks: List[str], max_concurrent: int = 5) -> List[str]:
    """批处理多个任务"""
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def process_one(task: str) -> str:
        async with semaphore:
            return await call_llm(task)
    
    results = await asyncio.gather(*[process_one(t) for t in tasks])
    return results


# 批量工具调用
async def batch_file_analysis(file_paths: List[str]) -> dict:
    """批量分析文件，而不是一个个分析"""
    # 合并成一个请求
    combined_prompt = "Analyze these files:\n"
    for path in file_paths[:10]:  # 限制数量
        content = read_file(path)[:1000]  # 限制每个文件长度
        combined_prompt += f"\n### {path}\n{content}\n"
    
    combined_prompt += "\nProvide analysis for each file."
    
    # 一次调用搞定
    return await call_llm(combined_prompt)
```

### 6. OpenClaw 的成本控制

**配置层面的控制：**

```yaml
# config.yaml
model:
  default: "anthropic/claude-sonnet-4"  # 默认用性价比高的
  
  # 按会话类型配置
  sessions:
    main:
      model: "anthropic/claude-opus-4"  # 主会话用最好的
    heartbeat:
      model: "anthropic/claude-haiku-3"  # 心跳检查用最便宜的
    subagent:
      model: "anthropic/claude-sonnet-4"  # 子 agent 用中档

# 预算控制
limits:
  daily_cost_usd: 50
  per_session_tokens: 100000
  alert_email: "admin@example.com"
```

**运行时切换：**

```bash
# 临时切换到便宜模型
/model haiku

# 查看当前用量
/status
```

## 实战：成本感知的 Agent Loop

```python
class CostOptimizedAgent:
    """成本优化的 Agent 实现"""
    
    def __init__(self):
        self.budget = TokenBudget()
        self.cache = ResponseCache()
        self.compressor = PromptCompressor()
        
        # 成本追踪
        self.session_cost = 0.0
        self.model_pricing = {
            'opus': {'input': 0.015, 'output': 0.075},
            'sonnet': {'input': 0.003, 'output': 0.015},
            'haiku': {'input': 0.00025, 'output': 0.00125},
        }
    
    def calculate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        pricing = self.model_pricing.get(model, self.model_pricing['sonnet'])
        return (input_tokens * pricing['input'] + output_tokens * pricing['output']) / 1000
    
    async def run_turn(self, messages: list, task_type: str = 'general') -> str:
        # 1. 压缩上下文
        compressed = self.compressor.compress_conversation(messages)
        
        # 2. 选择模型
        remaining_budget = self.budget.get_remaining()
        if remaining_budget < 50000:
            model = 'haiku'
        elif task_type in ['code_review', 'architecture']:
            model = 'opus'
        else:
            model = 'sonnet'
        
        # 3. 检查缓存
        response = await self.cache.get_or_call(
            messages=compressed,
            model=model,
            call_fn=lambda: self._call_llm(compressed, model)
        )
        
        # 4. 更新成本追踪
        cost = self.calculate_cost(model, response.input_tokens, response.output_tokens)
        self.session_cost += cost
        
        # 5. 记录到可观测系统
        self._emit_metric('llm_cost', cost, {'model': model, 'task_type': task_type})
        
        return response.content
    
    def get_session_summary(self) -> dict:
        return {
            'total_cost_usd': round(self.session_cost, 4),
            'remaining_budget': self.budget.get_remaining(),
            'cache_hit_rate': self.cache.get_hit_rate(),
        }
```

## 成本优化 Checklist

| 策略 | 节省幅度 | 实现难度 | 优先级 |
|------|---------|---------|--------|
| 响应缓存 | 30-70% | 低 | ⭐⭐⭐ |
| Prompt 压缩 | 20-40% | 中 | ⭐⭐⭐ |
| 模型降级 | 50-90% | 中 | ⭐⭐ |
| 批处理 | 10-30% | 低 | ⭐⭐ |
| Token 预算 | - | 低 | ⭐⭐⭐ |

## 关键要点

1. **分层模型策略**：不是所有任务都需要 Opus，学会分类任务
2. **缓存是王道**：相同输入 = 相同输出，不要重复花钱
3. **压缩上下文**：长对话历史可以摘要，工具结果可以截断
4. **预算告警**：设置阈值，避免意外账单
5. **监控成本**：不能优化你不测量的东西

## 下节预告

下节课我们将学习 **Agent Orchestration Patterns（Agent 编排模式）**，探讨如何组织多个 Agent 协同工作的不同模式。

---

*本课程每 3 小时更新，代码示例来自 learn-claude-code、pi-mono 和 OpenClaw*
