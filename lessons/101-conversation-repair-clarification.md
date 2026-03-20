# 101 - Agent 对话修复与澄清（Conversation Repair & Clarification）

> 不懂就问，但要问得聪明。
> Agent 最怕两件事：一是死猜到底、二是问烂了用户。

---

## 🎯 核心问题

用户说："帮我处理那个文件"——什么文件？怎么处理？

烂 Agent 的做法：
- **死猜**：随便选个文件操作，大概率错
- **连环问**：哪个文件？什么格式？怎么处理？输出到哪？……用户直接关 App

好 Agent 的做法：
- **识别歧义**：知道自己不知道什么
- **精准一问**：一次问最关键的那个问题
- **有时不问**：能从上下文推断的，不烦用户

---

## 🧠 Conversation Repair 是什么？

语言学概念：对话中一方发现理解出错或信息缺失，主动"修复"对话的行为。

在 Agent 里有三种场景：

| 场景 | 触发条件 | 修复方式 |
|------|----------|----------|
| **自我修复** | Agent 执行后发现结果不对 | 回滚，重新理解输入 |
| **他者修复** | 用户说"不对，我意思是..." | 接受纠正，更新理解 |
| **预防修复** | 执行前检测到歧义 | 主动澄清 |

我们重点讲第三种：**执行前的主动澄清**。

---

## 🔍 歧义检测：知道自己不知道什么

```python
# learn-claude-code 风格：歧义检测器
from anthropic import Anthropic
import json

client = Anthropic()

AMBIGUITY_DETECTOR_PROMPT = """你是一个 Agent 的歧义检测模块。

给你一个用户请求和可用工具列表，判断是否存在歧义。

输出 JSON：
{
  "is_ambiguous": true/false,
  "ambiguity_type": "missing_target" | "missing_action" | "missing_scope" | "conflicting_intent" | null,
  "confidence": 0.0-1.0,  // 你对自己理解的置信度
  "blocking_question": "最关键的一个问题（如果需要问）",
  "reasonable_default": "如果要猜，最合理的猜测是什么（null 表示无法合理猜测）"
}

规则：
- confidence > 0.85：直接执行，不问
- confidence 0.6-0.85：有 reasonable_default 就用默认，执行后告知用户你的理解
- confidence < 0.6：必须问 blocking_question
"""

def detect_ambiguity(user_request: str, available_tools: list[str], context: str = "") -> dict:
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=300,
        system=AMBIGUITY_DETECTOR_PROMPT,
        messages=[{
            "role": "user",
            "content": f"""用户请求：{user_request}
可用工具：{', '.join(available_tools)}
对话上下文：{context or '无'}"""
        }]
    )
    
    return json.loads(response.content[0].text)

# 测试
result = detect_ambiguity(
    user_request="帮我处理那个文件",
    available_tools=["read_file", "write_file", "delete_file", "convert_file"],
    context="上一条消息：用户上传了 report.xlsx"
)

# 输出：
# {
#   "is_ambiguous": true,
#   "ambiguity_type": "missing_action",
#   "confidence": 0.45,
#   "blocking_question": "您希望对 report.xlsx 做什么？转格式、读取内容，还是其他操作？",
#   "reasonable_default": null
# }
```

---

## 💡 置信度驱动的执行策略

关键洞察：**不是所有歧义都需要问**，用置信度决定行为：

```python
from dataclasses import dataclass
from enum import Enum

class ExecutionStrategy(Enum):
    EXECUTE_DIRECTLY = "direct"       # 置信度高，直接做
    EXECUTE_WITH_ECHO = "echo"        # 中等置信度，执行但回显理解
    ASK_THEN_EXECUTE = "ask"          # 低置信度，先问
    REFUSE_AND_EXPLAIN = "refuse"     # 完全不懂，解释为何无法执行

@dataclass 
class ExecutionDecision:
    strategy: ExecutionStrategy
    assumed_intent: str | None
    clarification_question: str | None

def decide_execution(ambiguity_result: dict) -> ExecutionDecision:
    confidence = ambiguity_result["confidence"]
    default = ambiguity_result.get("reasonable_default")
    question = ambiguity_result.get("blocking_question")
    
    if confidence > 0.85:
        return ExecutionDecision(
            strategy=ExecutionStrategy.EXECUTE_DIRECTLY,
            assumed_intent=None,
            clarification_question=None
        )
    elif confidence > 0.6 and default:
        return ExecutionDecision(
            strategy=ExecutionStrategy.EXECUTE_WITH_ECHO,
            assumed_intent=default,
            clarification_question=None
        )
    elif question:
        return ExecutionDecision(
            strategy=ExecutionStrategy.ASK_THEN_EXECUTE,
            assumed_intent=None,
            clarification_question=question
        )
    else:
        return ExecutionDecision(
            strategy=ExecutionStrategy.REFUSE_AND_EXPLAIN,
            assumed_intent=None,
            clarification_question=None
        )
```

---

## ✍️ 问题生成：一次只问最重要的

最常见的烂体验：Agent 一口气问 5 个问题。

```python
# 好的澄清问题设计原则

BAD_CLARIFICATION = """
我需要了解以下信息才能继续：
1. 你想处理哪个文件？
2. 想要什么格式？
3. 输出到哪里？
4. 需要保留原文件吗？
5. 有特殊字符编码要求吗？
"""

GOOD_CLARIFICATION = """
您是想把 report.xlsx 转成 CSV，还是有其他操作？
"""

# 好问题的特征：
# ✅ 一个问题，最多两个选项
# ✅ 包含你的假设（"您是想把 report.xlsx..."）  
# ✅ 给出最可能的选项（"转成 CSV"）
# ✅ 留一个开放出口（"还是有其他操作？"）
```

用 LLM 生成高质量澄清问题：

```python
CLARIFICATION_GEN_PROMPT = """根据歧义情况，生成一个高质量的澄清问题。

要求：
1. 只问一个最关键的问题
2. 在问题中体现你已知的上下文（显示你在认真听）
3. 提供 1-2 个最可能的选项
4. 保留开放出口
5. 语气自然，不要机器人腔

例子：
好的："您是想把昨天那份销售报告导出成 PDF，还是需要提取里面的数据？"
坏的："请问您要对文件进行什么操作？"
"""

def generate_clarification(
    user_request: str, 
    ambiguity_type: str,
    context: str,
    model: str = "claude-haiku-4-5"  # 用便宜模型，这是小任务
) -> str:
    response = client.messages.create(
        model=model,
        max_tokens=150,
        system=CLARIFICATION_GEN_PROMPT,
        messages=[{
            "role": "user", 
            "content": f"用户请求：{user_request}\n歧义类型：{ambiguity_type}\n上下文：{context}"
        }]
    )
    return response.content[0].text.strip()
```

---

## 🔄 OpenClaw 里的真实实现

OpenClaw 的 SOUL.md 里有一条原则处理这个问题：

```markdown
# SOUL.md 片段
- **遇到问题先汇报，等老板指示再操作，不要擅自"换思路"**
- **任何新方案必须先问老板，得到确认后才能执行**
```

这其实就是 "低置信度 → ASK_THEN_EXECUTE" 策略的自然语言版。

OpenClaw 里 Agent 的澄清循环：

```
用户输入
  ↓
歧义检测（置信度评估）
  ↓
[置信度 > 0.85] → 直接执行 → 结果 + 简短说明
[置信度 0.6-0.85] → 执行 + 回显（"我理解你想要..."）
[置信度 < 0.6] → 生成澄清问题 → 等待用户回复 → 重新执行
```

---

## 🚫 "他者修复"：优雅接受纠正

用户说"不对，我不是这个意思"——Agent 怎么处理？

```python
# pi-mono 风格：TypeScript 纠正处理器
interface CorrectionEvent {
  originalRequest: string;
  agentAction: string;
  userCorrection: string;
}

async function handleCorrection(event: CorrectionEvent, agent: Agent): Promise<void> {
  // 1. 立刻承认，不辩解
  // 2. 提取用户真实意图
  // 3. 更新短期上下文记忆
  // 4. 重新执行
  
  const trueIntent = await agent.llm.complete({
    system: "从用户的纠正中提取真实意图。输出：{ intent, key_diff }",
    messages: [
      { role: "user", content: event.originalRequest },
      { role: "assistant", content: `我执行了：${event.agentAction}` },
      { role: "user", content: event.userCorrection }
    ]
  });

  // 关键：把纠正信息注入 context，防止同一会话再犯
  agent.context.addCorrectionMemory({
    pattern: event.originalRequest,
    wrongInterpretation: event.agentAction,
    correctInterpretation: trueIntent.intent,
    keyDiff: trueIntent.key_diff
  });

  // 重新执行，这次有了纠正上下文
  await agent.execute(trueIntent.intent);
}
```

关键点：**把纠正写入 context 记忆**，同一会话内不犯同样的错。

---

## 📊 澄清限流：防止 Agent 变烦人

```python
class ClarificationRateLimiter:
    """
    限制 Agent 问问题的频率。
    同一会话内，澄清问题不能超过 N 次。
    超过后，强制选择 reasonable_default 或直接执行最可能的意图。
    """
    
    def __init__(self, max_clarifications_per_session: int = 2):
        self.max = max_clarifications_per_session
        self.counts: dict[str, int] = {}  # session_id -> count
    
    def should_ask(self, session_id: str, has_reasonable_default: bool) -> bool:
        count = self.counts.get(session_id, 0)
        
        if count >= self.max:
            # 已经问过太多次了，强制执行
            return False
        
        if count >= 1 and has_reasonable_default:
            # 已经问过一次，这次用默认值
            return False
            
        return True
    
    def increment(self, session_id: str):
        self.counts[session_id] = self.counts.get(session_id, 0) + 1

# 用法
limiter = ClarificationRateLimiter(max_clarifications_per_session=2)

def smart_clarify_or_execute(session_id: str, ambiguity: dict) -> str:
    has_default = bool(ambiguity.get("reasonable_default"))
    
    if ambiguity["confidence"] < 0.6 and limiter.should_ask(session_id, has_default):
        limiter.increment(session_id)
        return f"ASK: {ambiguity['blocking_question']}"
    elif has_default:
        return f"EXECUTE_WITH_DEFAULT: {ambiguity['reasonable_default']}"
    else:
        return "EXECUTE_BEST_GUESS"
```

---

## ✅ 最佳实践总结

```
1. 置信度 > 0.85 → 直接执行，不问
2. 中等置信度 → 执行 + 回显你的理解（"我理解你想要 X，已完成"）
3. 低置信度 → 问一个最关键的问题，给出选项
4. 每个会话问题上限：2 次（之后强制用默认值）
5. 接受纠正时：承认 + 记录纠正模式 + 重新执行
6. 用便宜模型（haiku/flash）做歧义检测，降低成本
```

---

## 🔑 一句话总结

> 好 Agent 不是什么都懂，而是**知道自己不懂什么**，并且能用一个精准的问题解决它。

---

*下一课：Agent 增量数据处理（Incremental Data Processing）*
