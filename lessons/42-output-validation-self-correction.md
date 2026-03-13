# 42 - Output Validation & Self-Correction：Agent 输出校验与自我纠错

> **核心问题：** LLM 输出了结果，但结果是对的吗？怎么让 Agent 自己发现并纠正错误？

---

## 为什么需要输出校验？

结构化输出（Lesson 23）解决了"让 LLM 按格式输出"的问题，但还不够：

```
LLM 输出了 JSON ✅
JSON 格式正确 ✅
但内容是错的 ❌  ← 这就是输出校验要解决的
```

**常见问题：**
- 幻觉（Hallucination）：LLM 编造了不存在的数据
- 逻辑错误：格式对但推理链断了
- 边界违反：数值超出业务允许范围
- 自相矛盾：输出内部前后不一致

---

## 三层校验架构

```
┌─────────────────────────────────────────┐
│            Layer 1: Schema Validation   │  结构合法？
├─────────────────────────────────────────┤
│            Layer 2: Business Rules      │  业务逻辑对？
├─────────────────────────────────────────┤
│            Layer 3: Self-Critique       │  语义可信？
└─────────────────────────────────────────┘
         ↓ 任一层失败 → 触发自我纠错循环
```

---

## 实现：三层校验 + 自我纠错

### Python 实现（learn-claude-code 风格）

```python
import anthropic
import json
from typing import Any, Optional
from dataclasses import dataclass, field
from pydantic import BaseModel, validator
import re

client = anthropic.Anthropic()

# ─── Layer 1：Schema 定义（Pydantic）───────────────────────────────────────

class AnalysisResult(BaseModel):
    summary: str
    confidence: float       # 0.0 ~ 1.0
    action_items: list[str]
    estimated_effort: str   # "low" | "medium" | "high"
    risks: list[str]

    @validator("confidence")
    def confidence_range(cls, v):
        if not 0.0 <= v <= 1.0:
            raise ValueError(f"confidence must be 0~1, got {v}")
        return v

    @validator("estimated_effort")
    def effort_valid(cls, v):
        if v not in ("low", "medium", "high"):
            raise ValueError(f"invalid effort: {v}")
        return v

# ─── Layer 2：业务规则校验 ─────────────────────────────────────────────────

def business_rules_check(result: AnalysisResult) -> list[str]:
    """返回违反的规则列表，空列表 = 通过"""
    violations = []

    if result.confidence > 0.9 and len(result.risks) == 0:
        violations.append("高置信度但没有风险项：可能存在过度乐观的幻觉")

    if result.estimated_effort == "low" and len(result.action_items) > 10:
        violations.append("10+ 个行动项但标记为 low effort，不合理")

    if len(result.summary) < 20:
        violations.append("summary 过短，缺乏实质内容")

    return violations

# ─── Layer 3：Self-Critique（让 LLM 审查自己的输出）───────────────────────

def self_critique(
    original_prompt: str,
    result: AnalysisResult,
    violations: list[str]
) -> Optional[str]:
    """
    让另一个 LLM 调用（可以是同模型，温度更低）来审查输出。
    返回 None 表示通过，否则返回改进建议。
    """
    critique_prompt = f"""你是一个严格的输出质量审核员。

原始任务：
{original_prompt}

AI 生成的分析结果：
{result.model_dump_json(indent=2)}

{"⚠️ 已发现业务规则违反：" + chr(10).join(violations) if violations else ""}

请评估这个分析结果的质量。检查：
1. 内容是否与任务相关？
2. 是否有明显的幻觉或编造信息？
3. 逻辑是否自洽？
4. confidence 值是否合理？

如果质量可以接受，回复：APPROVED
如果需要改进，回复：NEEDS_REVISION: [具体问题和改进方向]"""

    response = client.messages.create(
        model="claude-3-haiku-20240307",  # 用便宜的小模型做审查
        max_tokens=500,
        temperature=0,  # 审查要确定性强
        messages=[{"role": "user", "content": critique_prompt}]
    )

    critique_text = response.content[0].text.strip()
    if critique_text.startswith("APPROVED"):
        return None
    return critique_text.replace("NEEDS_REVISION: ", "")

# ─── 主函数：带自我纠错的 Agent ────────────────────────────────────────────

def analyze_with_self_correction(
    task: str,
    max_retries: int = 3
) -> AnalysisResult:
    
    system_prompt = """你是一个专业的分析助手。
分析用户提供的任务，并返回严格的 JSON 格式结果，不要包含任何其他文字。

格式：
{
  "summary": "简洁的摘要（至少20字）",
  "confidence": 0.0到1.0之间的数字,
  "action_items": ["行动项1", "行动项2"],
  "estimated_effort": "low/medium/high 之一",
  "risks": ["风险1", "风险2"]
}"""

    messages = [{"role": "user", "content": task}]
    last_error = None

    for attempt in range(max_retries):
        print(f"\n🔄 第 {attempt + 1} 次尝试...")

        # 如果有上次的错误，把纠错信息注入 prompt
        if last_error:
            messages.append({
                "role": "assistant",
                "content": last_raw_output  # 上次的错误输出
            })
            messages.append({
                "role": "user",
                "content": f"你的输出有问题，请重新生成：\n\n{last_error}\n\n请修正后重新输出完整的 JSON。"
            })

        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1000,
            system=system_prompt,
            messages=messages
        )

        raw_output = response.content[0].text.strip()
        last_raw_output = raw_output

        # ─── Layer 1：Schema 校验 ────────────────────────────────────────
        try:
            # 提取 JSON（防止模型输出了 markdown 代码块）
            json_match = re.search(r'\{.*\}', raw_output, re.DOTALL)
            if not json_match:
                raise ValueError("输出中找不到 JSON")
            
            data = json.loads(json_match.group())
            result = AnalysisResult(**data)
            print("  ✅ Layer 1: Schema 校验通过")
        except Exception as e:
            last_error = f"Schema 校验失败：{e}"
            print(f"  ❌ Layer 1: {last_error}")
            continue

        # ─── Layer 2：业务规则校验 ────────────────────────────────────────
        violations = business_rules_check(result)
        if violations:
            last_error = "业务规则违反：\n" + "\n".join(f"- {v}" for v in violations)
            print(f"  ❌ Layer 2: {last_error}")
            # 业务规则失败也可以继续到 Layer 3 让 self-critique 来修正
        else:
            print("  ✅ Layer 2: 业务规则校验通过")

        # ─── Layer 3：Self-Critique ─────────────────────────────────────
        critique = self_critique(task, result, violations)
        if critique:
            last_error = f"Self-Critique 建议修改：{critique}"
            print(f"  ❌ Layer 3: {last_error}")
            continue
        
        print("  ✅ Layer 3: Self-Critique 通过")
        print(f"\n✨ 校验成功！（第 {attempt + 1} 次尝试）")
        return result

    raise RuntimeError(f"经过 {max_retries} 次重试仍无法生成有效输出。最后错误：{last_error}")


# ─── 使用示例 ──────────────────────────────────────────────────────────────

if __name__ == "__main__":
    task = """
    分析以下项目计划的可行性：
    "在两周内，由1名工程师独立完成一个支持百万用户的实时聊天系统，
    包括前后端开发、数据库设计、负载均衡配置和安全审计。"
    """

    result = analyze_with_self_correction(task)
    print("\n📊 最终结果：")
    print(result.model_dump_json(indent=2))
```

---

### TypeScript 实现（pi-mono 风格）

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { z } from "zod";

const client = new Anthropic();

// ─── Zod Schema（Layer 1）─────────────────────────────────────────────────

const AnalysisSchema = z.object({
  summary: z.string().min(20, "summary 至少20字"),
  confidence: z.number().min(0).max(1),
  action_items: z.array(z.string()),
  estimated_effort: z.enum(["low", "medium", "high"]),
  risks: z.array(z.string()),
});

type Analysis = z.infer<typeof AnalysisSchema>;

// ─── 自我纠错循环 ──────────────────────────────────────────────────────────

interface ValidationResult {
  ok: boolean;
  errors: string[];
}

function validateBusinessRules(result: Analysis): ValidationResult {
  const errors: string[] = [];

  if (result.confidence > 0.9 && result.risks.length === 0) {
    errors.push("高置信度但无风险项 → 可能过度乐观");
  }

  if (result.estimated_effort === "low" && result.action_items.length > 10) {
    errors.push(`行动项${result.action_items.length}个但标记low effort`);
  }

  return { ok: errors.length === 0, errors };
}

async function analyzeWithSelfCorrection(
  task: string,
  maxRetries = 3
): Promise<Analysis> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: task },
  ];

  let lastError: string | null = null;
  let lastRawOutput = "";

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    console.log(`\n🔄 第 ${attempt + 1} 次尝试...`);

    // 注入上次的错误信息
    if (lastError && attempt > 0) {
      messages.push(
        { role: "assistant", content: lastRawOutput },
        {
          role: "user",
          content: `输出有问题，请修正后重新生成完整 JSON：\n\n${lastError}`,
        }
      );
    }

    const response = await client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 1000,
      system: `分析任务并返回 JSON（不含其他文字）：
{"summary":"...","confidence":0.8,"action_items":["..."],"estimated_effort":"medium","risks":["..."]}`,
      messages,
    });

    const rawText =
      response.content[0].type === "text" ? response.content[0].text : "";
    lastRawOutput = rawText;

    // Layer 1: Schema 校验
    try {
      const jsonMatch = rawText.match(/\{[\s\S]*\}/);
      if (!jsonMatch) throw new Error("无法提取 JSON");

      const parsed = JSON.parse(jsonMatch[0]);
      const result = AnalysisSchema.parse(parsed);
      console.log("  ✅ Layer 1: Schema 通过");

      // Layer 2: 业务规则
      const bizCheck = validateBusinessRules(result);
      if (!bizCheck.ok) {
        lastError = `业务规则违反：${bizCheck.errors.join("; ")}`;
        console.log(`  ❌ Layer 2: ${lastError}`);

        // Layer 3: Self-critique（即使业务规则失败也要检查）
        const critiqueResponse = await client.messages.create({
          model: "claude-3-haiku-20240307",
          max_tokens: 300,
          temperature: 0,
          messages: [
            {
              role: "user",
              content: `审查这个分析结果是否合理（已知问题：${bizCheck.errors.join(", ")}）：
              
${JSON.stringify(result, null, 2)}

回复 APPROVED 或 NEEDS_REVISION: [原因]`,
            },
          ],
        });

        const critique =
          critiqueResponse.content[0].type === "text"
            ? critiqueResponse.content[0].text
            : "";

        if (!critique.startsWith("APPROVED")) {
          lastError = critique;
          continue;
        }
      }

      console.log(`✨ 校验成功！（第 ${attempt + 1} 次）`);
      return result;
    } catch (e) {
      lastError = `Schema 校验失败: ${e}`;
      console.log(`  ❌ Layer 1: ${lastError}`);
    }
  }

  throw new Error(`${maxRetries} 次重试均失败。最后错误：${lastError}`);
}
```

---

## OpenClaw 中的实战：工具调用结果校验

OpenClaw 在处理工具结果时也有类似的校验逻辑。当一个工具返回意外格式，系统会把错误注回对话让 Agent 自我修正：

```typescript
// OpenClaw 工具执行后的结果处理（简化）
async function executeToolAndValidate(
  toolName: string,
  input: unknown,
  expectedSchema: z.ZodSchema
) {
  const result = await runTool(toolName, input);

  // 校验工具输出格式
  const parsed = expectedSchema.safeParse(result);

  if (!parsed.success) {
    // 把校验错误注入下一轮 Agent 消息
    return {
      type: "tool_result",
      is_error: true,
      content: `工具输出格式错误：${parsed.error.message}\n原始输出：${JSON.stringify(result)}`,
    };
  }

  return {
    type: "tool_result",
    content: JSON.stringify(parsed.data),
  };
}
```

---

## Self-Correction 的关键设计原则

### 1. 错误信息要具体

```python
# ❌ 模糊的错误信息（LLM 不知道怎么修）
last_error = "输出不合法"

# ✅ 具体的错误信息
last_error = """Schema 校验失败：
- confidence 字段值 1.5 超出 [0, 1] 范围
- estimated_effort 值 "very_high" 不在允许列表 [low/medium/high] 中
请修正这两个字段后重新输出完整 JSON。"""
```

### 2. 保留对话历史（Multi-turn 纠错）

```python
# 把错误输出和纠错指令追加到 messages
# LLM 能看到"我之前输出了什么 → 错在哪 → 应该怎么改"
# 比每次重置 context 效果好得多
messages.append({"role": "assistant", "content": bad_output})
messages.append({"role": "user", "content": f"请修正：{error_details}"})
```

### 3. 用便宜的模型做 Self-Critique

```python
# 生成用贵的模型（高质量）
generation_model = "claude-opus-4-5"

# 审查用便宜的模型（节省成本）
critique_model = "claude-3-haiku-20240307"

# 成本对比：每次校验只需要生成成本的 1/10
```

### 4. 设置重试上限，准备降级

```python
MAX_RETRIES = 3

try:
    result = analyze_with_self_correction(task, max_retries=MAX_RETRIES)
except RuntimeError as e:
    # 降级：返回部分结果，或通知人工介入
    result = fallback_analysis(task)
    alert_human(f"Agent 自我纠错失败：{e}")
```

---

## 常见陷阱

| 陷阱 | 问题 | 解决方案 |
|------|------|----------|
| Infinite loop | 校验永远不通过 | 设置 max_retries |
| Overcorrection | Self-critique 太严，APPROVED 率极低 | 调整 critique prompt，区分"必须修改"和"建议优化" |
| 错误信息泄露 | 把系统内部错误丢给 LLM 纠错 | 先 sanitize 错误信息，只保留对 LLM 有用的部分 |
| 成本失控 | 每次纠错都用贵模型 | critique 用 haiku，只有最终通过时才算 opus 的钱 |

---

## 小结

```
输出校验三层架构：
  Schema  → Pydantic/Zod 自动校验格式
  业务规则 → 你写的 Python/TS 函数
  Self-Critique → 让 LLM 审查 LLM

自我纠错核心：
  错误信息要具体 → 保留多轮对话历史 → 重试有上限 → 失败有降级

成本控制：
  便宜模型做 critique + haiku 审查 opus 生成
```

**下一课预告：** Multi-Tenant Agent Architecture（多租户 Agent 架构）

---

*Agent 开发课程 第42课 | 2026-03-14*
