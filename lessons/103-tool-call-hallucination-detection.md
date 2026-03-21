# 103 - Agent 工具调用幻觉检测与修复（Tool Call Hallucination Detection & Recovery）

> **核心问题：** LLM 会自信地调用一个不存在的工具，或者传一个类型完全错误的参数。你的 Agent 有没有在崩溃前先检查一下？

---

## 幻觉不只是"胡说答案"

大家通常说的 LLM 幻觉是"回答了一个错误的事实"。但在 Agent 系统里，还有另一类幻觉更危险：

**工具调用幻觉（Tool Call Hallucination）**

```
LLM 输出:
{
  "tool": "send_email_with_attachment",   ← 这个工具不存在
  "params": {
    "to": "boss@company.com",
    "attachment": "/tmp/report.pdf",
    "priority": "urgent"                  ← 这个参数也不存在
  }
}
```

常见的幻觉类型：

| 类型 | 描述 | 后果 |
|------|------|------|
| **幻觉工具名** | 调用未注册的工具 | KeyError / 崩溃 |
| **缺失必填参数** | 忘记传 required 字段 | 验证失败 |
| **参数类型错误** | 传字符串给 integer 字段 | 类型异常 |
| **参数逻辑错误** | start_date > end_date | 静默错误，结果错 |
| **参数幻觉** | 传了 schema 里没有的参数 | 被忽略或报错 |
| **枚举越界** | enum 字段传了合法值之外的内容 | 下游服务 400 |

---

## 核心架构：拦截层 + 诊断 + 恢复

```
[LLM] ──tool_call──▶ [Hallucination Detector]
                            │
                    ┌───────┴───────┐
                    ▼               ▼
               [Valid Call]   [Hallucination?]
                    │               │
               [Execute]    [Diagnose & Repair]
                                    │
                            ┌───────┴───────┐
                            ▼               ▼
                       [Auto-Fix]    [Re-prompt LLM]
                            │               │
                            └───────┬───────┘
                                    ▼
                              [Retry Execute]
```

---

## 实现：TypeScript 版（pi-mono 风格）

```typescript
// tool-hallucination-guard.ts

interface ToolSchema {
  name: string;
  description: string;
  parameters: {
    type: 'object';
    properties: Record<string, {
      type: string;
      description?: string;
      enum?: string[];
      minimum?: number;
      maximum?: number;
    }>;
    required: string[];
  };
}

interface ToolCall {
  tool: string;
  params: Record<string, unknown>;
}

interface HallucinationDiagnosis {
  isHallucination: boolean;
  errors: HallucinationError[];
  suggestedFix?: Partial<ToolCall>;
}

interface HallucinationError {
  type: 'UNKNOWN_TOOL' | 'MISSING_PARAM' | 'WRONG_TYPE' | 'INVALID_ENUM' | 'UNKNOWN_PARAM' | 'LOGIC_ERROR';
  field?: string;
  message: string;
  autoFixable: boolean;
}

class ToolHallucinationGuard {
  private registry: Map<string, ToolSchema>;
  private fuzzyMatcher: FuzzyToolMatcher;

  constructor(schemas: ToolSchema[]) {
    this.registry = new Map(schemas.map(s => [s.name, s]));
    this.fuzzyMatcher = new FuzzyToolMatcher(schemas);
  }

  diagnose(call: ToolCall): HallucinationDiagnosis {
    const errors: HallucinationError[] = [];
    let suggestedFix: Partial<ToolCall> = { tool: call.tool, params: { ...call.params } };

    // ① 检查工具是否存在
    if (!this.registry.has(call.tool)) {
      const closestMatch = this.fuzzyMatcher.findClosest(call.tool);
      const autoFixable = closestMatch !== null && closestMatch.similarity > 0.85;
      
      errors.push({
        type: 'UNKNOWN_TOOL',
        message: `Tool "${call.tool}" not found.${closestMatch ? ` Did you mean "${closestMatch.name}"?` : ''}`,
        autoFixable,
      });

      if (autoFixable && closestMatch) {
        suggestedFix.tool = closestMatch.name;
      } else {
        // 工具都找不到，后续参数检查没意义
        return { isHallucination: true, errors };
      }
    }

    const schema = this.registry.get(suggestedFix.tool!)!;
    const props = schema.parameters.properties;
    const required = schema.parameters.required;

    // ② 检查必填参数
    for (const req of required) {
      if (!(req in call.params) || call.params[req] === null || call.params[req] === undefined) {
        errors.push({
          type: 'MISSING_PARAM',
          field: req,
          message: `Required parameter "${req}" is missing.`,
          autoFixable: false,
        });
      }
    }

    // ③ 检查未知参数（幻觉参数）
    for (const key of Object.keys(call.params)) {
      if (!(key in props)) {
        errors.push({
          type: 'UNKNOWN_PARAM',
          field: key,
          message: `Parameter "${key}" is not in schema. Removing it.`,
          autoFixable: true,
        });
        // 自动移除幻觉参数
        delete (suggestedFix.params as Record<string, unknown>)[key];
      }
    }

    // ④ 检查类型
    for (const [key, value] of Object.entries(call.params)) {
      if (!(key in props)) continue;
      const expectedType = props[key].type;
      const actualType = typeof value;

      if (expectedType === 'integer' || expectedType === 'number') {
        if (actualType === 'string' && !isNaN(Number(value))) {
          // 可自动修复：字符串 "42" → 数字 42
          errors.push({
            type: 'WRONG_TYPE',
            field: key,
            message: `"${key}" should be ${expectedType}, got string "${value}". Auto-coercing.`,
            autoFixable: true,
          });
          (suggestedFix.params as Record<string, unknown>)[key] = Number(value);
        } else if (actualType !== 'number') {
          errors.push({
            type: 'WRONG_TYPE',
            field: key,
            message: `"${key}" should be ${expectedType}, got ${actualType}.`,
            autoFixable: false,
          });
        }
      }

      if (expectedType === 'boolean' && actualType === 'string') {
        if (value === 'true' || value === 'false') {
          errors.push({
            type: 'WRONG_TYPE',
            field: key,
            message: `"${key}" should be boolean, got string "${value}". Auto-coercing.`,
            autoFixable: true,
          });
          (suggestedFix.params as Record<string, unknown>)[key] = value === 'true';
        }
      }

      // ⑤ 检查 enum
      if (props[key].enum && !props[key].enum!.includes(value as string)) {
        errors.push({
          type: 'INVALID_ENUM',
          field: key,
          message: `"${key}" value "${value}" not in enum [${props[key].enum!.join(', ')}].`,
          autoFixable: false,
        });
      }
    }

    // ⑥ 业务逻辑校验（可扩展）
    const logicErrors = this.checkLogic(suggestedFix.tool!, suggestedFix.params as Record<string, unknown>, props);
    errors.push(...logicErrors);

    const isHallucination = errors.length > 0;
    return { isHallucination, errors, suggestedFix: isHallucination ? suggestedFix : undefined };
  }

  private checkLogic(
    tool: string,
    params: Record<string, unknown>,
    props: Record<string, { type: string }>
  ): HallucinationError[] {
    const errors: HallucinationError[] = [];

    // 日期范围逻辑检查
    const dateFields = Object.keys(props).filter(k => k.toLowerCase().includes('date'));
    const startField = dateFields.find(k => k.includes('start') || k.includes('from') || k.includes('begin'));
    const endField = dateFields.find(k => k.includes('end') || k.includes('to') || k.includes('until'));

    if (startField && endField && params[startField] && params[endField]) {
      const start = new Date(params[startField] as string);
      const end = new Date(params[endField] as string);
      if (!isNaN(start.getTime()) && !isNaN(end.getTime()) && start > end) {
        errors.push({
          type: 'LOGIC_ERROR',
          field: `${startField}/${endField}`,
          message: `start date "${params[startField]}" is after end date "${params[endField]}".`,
          autoFixable: false,
        });
      }
    }

    return errors;
  }

  canAutoFix(diagnosis: HallucinationDiagnosis): boolean {
    if (!diagnosis.isHallucination) return true;
    // 所有错误都可以自动修复
    return diagnosis.errors.every(e => e.autoFixable);
  }
}

// ————————————————————
// 模糊匹配工具名
// ————————————————————
class FuzzyToolMatcher {
  private names: string[];
  private schemas: ToolSchema[];

  constructor(schemas: ToolSchema[]) {
    this.schemas = schemas;
    this.names = schemas.map(s => s.name);
  }

  findClosest(name: string): { name: string; similarity: number } | null {
    let best: { name: string; similarity: number } | null = null;

    for (const candidate of this.names) {
      const sim = this.similarity(name.toLowerCase(), candidate.toLowerCase());
      if (!best || sim > best.similarity) {
        best = { name: candidate, similarity: sim };
      }
    }

    return best && best.similarity > 0.5 ? best : null;
  }

  private similarity(a: string, b: string): number {
    // Dice coefficient on bigrams
    const bigrams = (s: string) => {
      const result = new Set<string>();
      for (let i = 0; i < s.length - 1; i++) result.add(s.slice(i, i + 2));
      return result;
    };
    const aB = bigrams(a);
    const bB = bigrams(b);
    let intersection = 0;
    for (const bg of aB) if (bB.has(bg)) intersection++;
    return (2 * intersection) / (aB.size + bB.size);
  }
}
```

---

## 与 Agent Loop 集成：拦截 + 修复 + 重试

```typescript
// agent-loop-with-hallucination-guard.ts

class AgentLoopWithGuard {
  private guard: ToolHallucinationGuard;
  private tools: Map<string, (params: Record<string, unknown>) => Promise<unknown>>;
  private maxHallucinationRetries = 2;

  async executeToolCall(
    call: ToolCall,
    conversationHistory: Message[]
  ): Promise<{ result?: unknown; error?: string; retried: boolean }> {
    const diagnosis = this.guard.diagnose(call);

    if (!diagnosis.isHallucination) {
      // 完全正常，直接执行
      const fn = this.tools.get(call.tool)!;
      const result = await fn(call.params);
      return { result, retried: false };
    }

    // 记录幻觉日志
    console.warn('[HallucinationGuard] Detected:', {
      originalCall: call,
      errors: diagnosis.errors,
    });

    // 尝试自动修复
    if (this.guard.canAutoFix(diagnosis) && diagnosis.suggestedFix) {
      const fixedCall = diagnosis.suggestedFix as ToolCall;
      console.log('[HallucinationGuard] Auto-fixed:', fixedCall);
      
      const fn = this.tools.get(fixedCall.tool);
      if (fn) {
        const result = await fn(fixedCall.params);
        return { result, retried: false }; // 自动修复，对 LLM 透明
      }
    }

    // 无法自动修复 → 把诊断结果注入对话，让 LLM 重试
    const errorFeedback = this.buildErrorFeedback(call, diagnosis);
    return await this.repromptLLM(errorFeedback, conversationHistory);
  }

  private buildErrorFeedback(call: ToolCall, diagnosis: HallucinationDiagnosis): string {
    const errorList = diagnosis.errors
      .map(e => `- [${e.type}] ${e.message}`)
      .join('\n');

    return `Tool call failed validation:
Tool: "${call.tool}"
Errors:
${errorList}

Available tools: ${[...this.tools.keys()].join(', ')}
Please correct the tool call and try again.`;
  }

  private async repromptLLM(
    feedback: string,
    history: Message[],
    attempt = 0
  ): Promise<{ result?: unknown; error?: string; retried: boolean }> {
    if (attempt >= this.maxHallucinationRetries) {
      return { error: 'Max hallucination retry exceeded', retried: true };
    }

    // 将错误反馈作为 tool_result 注入上下文，让 LLM 重新决策
    const newHistory: Message[] = [
      ...history,
      {
        role: 'tool',
        content: feedback,
        isHallucinationFeedback: true,
      },
    ];

    const newCall = await this.callLLM(newHistory);
    if (!newCall) return { error: 'LLM returned no tool call', retried: true };

    const newDiagnosis = this.guard.diagnose(newCall);
    if (newDiagnosis.isHallucination) {
      return this.repromptLLM(
        this.buildErrorFeedback(newCall, newDiagnosis),
        newHistory,
        attempt + 1
      );
    }

    const fn = this.tools.get(newCall.tool)!;
    const result = await fn(newCall.params);
    return { result, retried: true };
  }
}
```

---

## Python 版（learn-claude-code 风格）

```python
# tool_hallucination_guard.py

from dataclasses import dataclass, field
from typing import Any, Optional
from enum import Enum
import json

class ErrorType(Enum):
    UNKNOWN_TOOL = "UNKNOWN_TOOL"
    MISSING_PARAM = "MISSING_PARAM"
    WRONG_TYPE = "WRONG_TYPE"
    INVALID_ENUM = "INVALID_ENUM"
    UNKNOWN_PARAM = "UNKNOWN_PARAM"
    LOGIC_ERROR = "LOGIC_ERROR"

@dataclass
class HallucinationError:
    type: ErrorType
    message: str
    auto_fixable: bool
    field: Optional[str] = None

@dataclass
class Diagnosis:
    is_hallucination: bool
    errors: list[HallucinationError] = field(default_factory=list)
    suggested_fix: Optional[dict] = None

class ToolHallucinationGuard:
    def __init__(self, tools: list[dict]):
        """tools: list of tool schema dicts (Anthropic format)"""
        self.registry = {t["name"]: t for t in tools}

    def diagnose(self, tool_name: str, params: dict) -> Diagnosis:
        errors: list[HallucinationError] = []
        fixed_params = dict(params)
        fixed_tool = tool_name

        # 1. 工具是否存在
        if tool_name not in self.registry:
            closest = self._fuzzy_match(tool_name)
            if closest and self._similarity(tool_name, closest) > 0.8:
                errors.append(HallucinationError(
                    type=ErrorType.UNKNOWN_TOOL,
                    message=f'Unknown tool "{tool_name}", did you mean "{closest}"?',
                    auto_fixable=True,
                ))
                fixed_tool = closest
            else:
                errors.append(HallucinationError(
                    type=ErrorType.UNKNOWN_TOOL,
                    message=f'Unknown tool "{tool_name}". Available: {list(self.registry.keys())}',
                    auto_fixable=False,
                ))
                return Diagnosis(is_hallucination=True, errors=errors)

        schema = self.registry[fixed_tool]
        input_schema = schema.get("input_schema", {})
        props = input_schema.get("properties", {})
        required = input_schema.get("required", [])

        # 2. 必填参数
        for req in required:
            if req not in params or params[req] is None:
                errors.append(HallucinationError(
                    type=ErrorType.MISSING_PARAM,
                    message=f'Missing required parameter "{req}"',
                    auto_fixable=False,
                    field=req,
                ))

        # 3. 未知参数（自动移除）
        for key in list(fixed_params.keys()):
            if key not in props:
                errors.append(HallucinationError(
                    type=ErrorType.UNKNOWN_PARAM,
                    message=f'Unknown parameter "{key}" removed',
                    auto_fixable=True,
                    field=key,
                ))
                del fixed_params[key]

        # 4. 类型 & enum 检查
        for key, val in list(params.items()):
            if key not in props:
                continue
            prop = props[key]
            expected = prop.get("type")

            # 数字类型强转
            if expected in ("integer", "number") and isinstance(val, str):
                try:
                    fixed_params[key] = int(val) if expected == "integer" else float(val)
                    errors.append(HallucinationError(
                        type=ErrorType.WRONG_TYPE,
                        message=f'"{key}" coerced from string "{val}" to {expected}',
                        auto_fixable=True,
                        field=key,
                    ))
                except ValueError:
                    errors.append(HallucinationError(
                        type=ErrorType.WRONG_TYPE,
                        message=f'"{key}" should be {expected}, got "{val}"',
                        auto_fixable=False,
                        field=key,
                    ))

            # enum 检查
            if "enum" in prop and val not in prop["enum"]:
                errors.append(HallucinationError(
                    type=ErrorType.INVALID_ENUM,
                    message=f'"{key}"="{val}" not in enum {prop["enum"]}',
                    auto_fixable=False,
                    field=key,
                ))

        is_hall = len(errors) > 0
        return Diagnosis(
            is_hallucination=is_hall,
            errors=errors,
            suggested_fix={"tool": fixed_tool, "params": fixed_params} if is_hall else None,
        )

    def can_auto_fix(self, diag: Diagnosis) -> bool:
        return all(e.auto_fixable for e in diag.errors)

    def _fuzzy_match(self, name: str) -> Optional[str]:
        best, best_sim = None, 0.0
        for candidate in self.registry:
            sim = self._similarity(name.lower(), candidate.lower())
            if sim > best_sim:
                best, best_sim = candidate, sim
        return best if best_sim > 0.5 else None

    def _similarity(self, a: str, b: str) -> float:
        # Jaccard on character trigrams
        tri = lambda s: {s[i:i+3] for i in range(len(s)-2)}
        ta, tb = tri(a), tri(b)
        if not ta and not tb:
            return 1.0
        return len(ta & tb) / len(ta | tb) if ta | tb else 0.0


# ————————————————————
# 与 Claude API 集成
# ————————————————————
def run_agent_with_guard(
    user_message: str,
    tools: list[dict],
    tool_handlers: dict,
) -> str:
    guard = ToolHallucinationGuard(tools)
    messages = [{"role": "user", "content": user_message}]
    max_iters = 10

    for _ in range(max_iters):
        response = anthropic_client.messages.create(
            model="claude-opus-4-5",
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            return response.content[0].text

        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type != "tool_use":
                    continue

                diag = guard.diagnose(block.name, block.input)

                if not diag.is_hallucination:
                    # 正常执行
                    result = tool_handlers[block.name](**block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(result),
                    })

                elif guard.can_auto_fix(diag):
                    # 自动修复
                    fixed = diag.suggested_fix
                    result = tool_handlers[fixed["tool"]](**fixed["params"])
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(result),
                    })

                else:
                    # 把错误反馈给 LLM
                    error_msg = "Tool call failed:\n" + "\n".join(
                        f"- [{e.type.value}] {e.message}" for e in diag.errors
                    )
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": error_msg,
                        "is_error": True,
                    })

            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})

    return "Max iterations reached"
```

---

## OpenClaw 实战：见过的真实幻觉

OpenClaw 代码里有这样的防护逻辑（简化版）：

```typescript
// openclaw/src/tool-dispatcher.ts（概念示意）

async dispatch(call: { name: string; input: unknown }) {
  const tool = this.registry.get(call.name);

  if (!tool) {
    // 不抛异常，而是返回结构化错误让 LLM 重试
    return {
      type: 'tool_result',
      is_error: true,
      content: `Unknown tool: "${call.name}". ` +
        `Available tools: ${[...this.registry.keys()].join(', ')}`,
    };
  }

  // Zod schema 验证
  const parsed = tool.schema.safeParse(call.input);
  if (!parsed.success) {
    return {
      type: 'tool_result',
      is_error: true,
      content: `Parameter validation failed: ${parsed.error.message}`,
    };
  }

  return tool.handler(parsed.data);
}
```

关键设计：**把 schema 验证错误变成 tool_result 返回，而不是 throw**。这让 LLM 有机会看到自己的错误并自我修正。

---

## 幻觉发生率统计（真实数据参考）

不同模型在工具调用上的幻觉率（Berkeley Function Calling Leaderboard 数据）：

| 模型 | 工具名幻觉率 | 参数错误率 |
|------|-------------|------------|
| GPT-4o | ~2% | ~8% |
| Claude Opus | ~1% | ~5% |
| Gemini Pro | ~3% | ~10% |
| 较弱模型 | ~15-30% | ~30-50% |

**结论：即使是最好的模型，在复杂工具集下也会出错。guard 层是必须的。**

---

## 最佳实践 Checklist

```
✅ Schema 验证放在执行之前，不是之后
✅ 未知工具 → 返回可用工具列表，不要 crash
✅ 参数错误 → 结构化错误信息，指出哪个字段出了什么问题
✅ 能自动修复的（类型强转、移除多余参数）→ 静默修复
✅ 无法修复的 → 作为 tool_result is_error:true 反馈给 LLM
✅ 最多重试 2-3 次，超过则走降级（返回默认值 or 人工干预）
✅ 记录所有幻觉事件，用于模型选择和 prompt 优化
✅ 工具数量 > 20 时，考虑工具分组 + 按意图动态加载（降低幻觉率）
```

---

## 小结

```
工具调用幻觉是 Agent 生产环境最常见的故障来源之一。

防御三层：
  1. 拦截层  → Schema 验证，发现问题
  2. 修复层  → 能自动改的静默改（类型强转、移除幻觉参数）
  3. 反馈层  → 改不了的，把诊断结果喂回 LLM，让它自我修正

不要 throw → 要 return error as tool_result
这是 Agent 和普通程序最大的区别之一：
  普通程序失败就 crash，Agent 要把失败作为信息传递给下一步。
```
