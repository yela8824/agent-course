# 05 - 模型路由：智能选择最优 LLM

## 为什么需要模型路由？

在生产环境中，一个 Agent 往往不只用一个模型。不同任务有不同需求：

| 任务类型 | 最佳模型选择 | 原因 |
|---------|-------------|------|
| 复杂推理 | Claude Opus / GPT-4o | 需要强逻辑能力 |
| 简单问答 | Claude Haiku / GPT-4o-mini | 快速、便宜 |
| 代码生成 | Claude Sonnet / Codex | 代码专精 |
| 图像理解 | GPT-4o / Claude 3.5 | 视觉能力强 |

**核心权衡**：质量 vs 速度 vs 成本

```
Claude Opus:  质量 ★★★★★  速度 ★★☆☆☆  成本 $$$$$
Claude Sonnet: 质量 ★★★★☆  速度 ★★★☆☆  成本 $$$
Claude Haiku: 质量 ★★★☆☆  速度 ★★★★★  成本 $
```

## 路由策略

### 策略 1: 任务分类路由

根据任务类型自动选择模型：

```python
class TaskRouter:
    """根据任务类型选择模型"""
    
    MODEL_MAP = {
        "reasoning": "anthropic/claude-opus-4-5",
        "coding": "anthropic/claude-sonnet-4",
        "simple_qa": "anthropic/claude-haiku-3",
        "vision": "openai/gpt-4o",
    }
    
    def classify_task(self, prompt: str) -> str:
        """分类任务（可以用小模型来分类）"""
        # 简单关键词匹配
        if any(kw in prompt.lower() for kw in ["分析", "推理", "为什么"]):
            return "reasoning"
        if any(kw in prompt.lower() for kw in ["代码", "函数", "class", "def"]):
            return "coding"
        if any(kw in prompt.lower() for kw in ["图片", "image", "看看"]):
            return "vision"
        return "simple_qa"
    
    def route(self, prompt: str) -> str:
        task_type = self.classify_task(prompt)
        return self.MODEL_MAP[task_type]
```

### 策略 2: 成本感知路由

设置预算，智能降级：

```python
class CostAwareRouter:
    """成本感知路由"""
    
    def __init__(self, daily_budget_usd: float = 10.0):
        self.daily_budget = daily_budget_usd
        self.today_spend = 0.0
    
    # 每百万 token 成本 (估算)
    COSTS = {
        "anthropic/claude-opus-4-5": 75.0,    # $15/M input + $75/M output
        "anthropic/claude-sonnet-4": 15.0,    # $3/M input + $15/M output  
        "anthropic/claude-haiku-3": 1.25,     # $0.25/M input + $1.25/M output
    }
    
    def route(self, estimated_tokens: int, task_importance: str) -> str:
        remaining = self.daily_budget - self.today_spend
        
        # 重要任务：用最好的模型
        if task_importance == "critical":
            return "anthropic/claude-opus-4-5"
        
        # 预算充足：用 Sonnet
        if remaining > self.daily_budget * 0.5:
            return "anthropic/claude-sonnet-4"
        
        # 预算紧张：用 Haiku
        return "anthropic/claude-haiku-3"
```

### 策略 3: 延迟敏感路由

实时场景选快的模型：

```python
class LatencyRouter:
    """延迟敏感路由"""
    
    # 平均响应时间 (首 token)
    LATENCY_MS = {
        "anthropic/claude-opus-4-5": 2000,
        "anthropic/claude-sonnet-4": 800,
        "anthropic/claude-haiku-3": 200,
    }
    
    def route(self, max_latency_ms: int) -> str:
        for model, latency in sorted(self.LATENCY_MS.items(), key=lambda x: -x[1]):
            if latency <= max_latency_ms:
                return model
        # 兜底返回最快的
        return "anthropic/claude-haiku-3"
```

## OpenClaw 的模型配置

OpenClaw 支持多级模型配置：

```yaml
# config.yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-sonnet-4"  # 默认模型
    
# 会话级别覆盖
session:
  model_override: "anthropic/claude-opus-4-5"  # 特定会话用 Opus
```

### 模型别名

简化配置：

```yaml
# 使用别名
model: opus          # → anthropic/claude-opus-4-5
model: sonnet        # → anthropic/claude-sonnet-4
model: haiku         # → anthropic/claude-haiku-3
```

## 实战：实现智能路由器

完整实现：

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import time

class TaskType(Enum):
    REASONING = "reasoning"
    CODING = "coding"
    CHAT = "chat"
    VISION = "vision"

@dataclass
class ModelConfig:
    name: str
    cost_per_1m_tokens: float
    avg_latency_ms: int
    quality_score: int  # 1-10

class SmartRouter:
    """
    智能模型路由器
    综合考虑：任务类型、成本、延迟、质量
    """
    
    MODELS = {
        "opus": ModelConfig(
            name="anthropic/claude-opus-4-5",
            cost_per_1m_tokens=75.0,
            avg_latency_ms=2000,
            quality_score=10
        ),
        "sonnet": ModelConfig(
            name="anthropic/claude-sonnet-4", 
            cost_per_1m_tokens=15.0,
            avg_latency_ms=800,
            quality_score=8
        ),
        "haiku": ModelConfig(
            name="anthropic/claude-haiku-3",
            cost_per_1m_tokens=1.25,
            avg_latency_ms=200,
            quality_score=6
        ),
    }
    
    # 任务类型 → 推荐模型
    TASK_PREFERENCES = {
        TaskType.REASONING: ["opus", "sonnet"],
        TaskType.CODING: ["sonnet", "opus"],
        TaskType.CHAT: ["haiku", "sonnet"],
        TaskType.VISION: ["sonnet", "opus"],
    }
    
    def __init__(
        self,
        daily_budget: float = 10.0,
        max_latency_ms: int = 5000,
        min_quality: int = 5
    ):
        self.daily_budget = daily_budget
        self.max_latency_ms = max_latency_ms
        self.min_quality = min_quality
        self.today_spend = 0.0
        self.request_count = 0
    
    def route(
        self,
        task_type: TaskType,
        estimated_tokens: int = 1000,
        force_model: Optional[str] = None,
        realtime: bool = False
    ) -> str:
        """
        智能路由决策
        
        Args:
            task_type: 任务类型
            estimated_tokens: 预估 token 数
            force_model: 强制使用某模型
            realtime: 是否实时场景（低延迟优先）
        
        Returns:
            模型名称
        """
        # 强制指定
        if force_model and force_model in self.MODELS:
            return self.MODELS[force_model].name
        
        # 获取该任务类型的候选模型
        candidates = self.TASK_PREFERENCES.get(task_type, ["sonnet"])
        
        # 过滤：质量门槛
        candidates = [
            c for c in candidates 
            if self.MODELS[c].quality_score >= self.min_quality
        ]
        
        # 实时场景：延迟优先
        if realtime:
            candidates = [
                c for c in candidates
                if self.MODELS[c].avg_latency_ms <= self.max_latency_ms
            ]
            # 选最快的
            candidates.sort(key=lambda c: self.MODELS[c].avg_latency_ms)
        
        # 成本检查
        remaining_budget = self.daily_budget - self.today_spend
        affordable = []
        for c in candidates:
            cost = self.MODELS[c].cost_per_1m_tokens * estimated_tokens / 1_000_000
            if cost <= remaining_budget * 0.1:  # 单次不超过剩余的 10%
                affordable.append(c)
        
        if affordable:
            candidates = affordable
        
        # 返回第一个候选（最优）
        if candidates:
            return self.MODELS[candidates[0]].name
        
        # 兜底：最便宜的
        return self.MODELS["haiku"].name
    
    def record_usage(self, model: str, tokens: int):
        """记录使用量"""
        for key, config in self.MODELS.items():
            if config.name == model:
                cost = config.cost_per_1m_tokens * tokens / 1_000_000
                self.today_spend += cost
                self.request_count += 1
                break


# 使用示例
router = SmartRouter(daily_budget=5.0)

# 复杂推理 → Opus
model = router.route(TaskType.REASONING, estimated_tokens=2000)
print(f"推理任务: {model}")  # anthropic/claude-opus-4-5

# 实时聊天 → Haiku（延迟优先）
model = router.route(TaskType.CHAT, realtime=True)
print(f"实时聊天: {model}")  # anthropic/claude-haiku-3

# 代码生成 → Sonnet
model = router.route(TaskType.CODING, estimated_tokens=5000)
print(f"代码生成: {model}")  # anthropic/claude-sonnet-4
```

## 进阶：分层路由架构

生产系统常用分层设计：

```
┌─────────────────────────────────────────┐
│              用户请求                    │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│         L1: 任务分类器                   │
│    (用 Haiku 快速分类任务类型)           │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│         L2: 路由决策器                   │
│    (根据分类 + 上下文选择模型)           │
└─────────────┬───────────────────────────┘
              │
     ┌────────┼────────┬────────┐
     ▼        ▼        ▼        ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│ Opus   ││ Sonnet ││ Haiku  ││ Vision │
│ 复杂   ││ 通用   ││ 简单   ││ 图像   │
└────────┘└────────┘└────────┘└────────┘
```

```python
class TieredRouter:
    """分层路由器"""
    
    async def route(self, messages: list) -> str:
        # L1: 快速分类（用小模型）
        classification = await self.classify_with_haiku(messages)
        
        # L2: 根据分类决策
        if classification["complexity"] == "high":
            if classification["requires_vision"]:
                return "openai/gpt-4o"
            return "anthropic/claude-opus-4-5"
        
        if classification["is_code"]:
            return "anthropic/claude-sonnet-4"
        
        return "anthropic/claude-haiku-3"
    
    async def classify_with_haiku(self, messages: list) -> dict:
        """用 Haiku 做快速分类"""
        prompt = f"""
        分析这个对话，返回 JSON:
        {{
            "complexity": "low" | "medium" | "high",
            "is_code": true | false,
            "requires_vision": true | false
        }}
        
        对话: {messages[-1]["content"][:500]}
        """
        # 调用 Haiku 分类...
        return {"complexity": "medium", "is_code": False, "requires_vision": False}
```

## 关键设计原则

1. **默认保守**：默认用便宜的模型，按需升级
2. **可观测**：记录每次路由决策，便于调优
3. **可配置**：允许用户 override 默认路由
4. **降级机制**：API 故障时自动切换到备用模型

## 总结

| 策略 | 适用场景 | 优点 | 缺点 |
|-----|---------|------|------|
| 任务分类 | 任务类型明确 | 简单直接 | 分类可能出错 |
| 成本感知 | 预算有限 | 控制成本 | 可能牺牲质量 |
| 延迟敏感 | 实时交互 | 用户体验好 | 可能质量下降 |
| 分层路由 | 生产环境 | 灵活全面 | 复杂度高 |

**最佳实践**：组合多种策略，根据业务场景权衡。

---

下一课：Context Compaction 进阶 - 长对话的上下文管理
