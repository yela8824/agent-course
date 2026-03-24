# 127 - Agent 工具参数自动补全与上下文推断（Tool Parameter Auto-Completion & Contextual Inference）

> **一句话总结：** 用户说"帮我发邮件给上次那个人"——Agent 不该傻问"请问是谁？"，而该从对话历史、工具调用记录、用户偏好中**自动推断**缺失参数，做到"用户说半句，Agent 干完整"。

---

## 为什么需要参数自动补全？

每次工具调用都要用户填完整参数 = 差体验：

```
用户: 帮我查一下订单
Agent: 请提供订单ID
用户: 就上一个啊
Agent: 请提供具体的订单ID
用户: 你能不能聪明点...
```

好的 Agent 应该：
1. 从上下文自动推断缺失参数
2. 只在真的无法推断时才问
3. 推断时说明来源，保持透明

---

## 核心推断策略

### 策略 1：对话历史提取

```typescript
// pi-mono 风格：从历史消息中提取实体
interface ContextExtractor {
  extract(history: Message[], paramName: string): InferredValue | null;
}

class ConversationHistoryExtractor implements ContextExtractor {
  extract(history: Message[], paramName: string): InferredValue | null {
    // 倒序扫描历史，找最近提到的相关实体
    for (let i = history.length - 1; i >= 0; i--) {
      const msg = history[i];
      const value = this.extractFromMessage(msg, paramName);
      if (value) {
        return {
          value,
          source: `conversation[${i}]`,
          confidence: this.calcConfidence(i, history.length),
        };
      }
    }
    return null;
  }

  private calcConfidence(msgIndex: number, total: number): number {
    // 越近的消息置信度越高
    const recency = (msgIndex + 1) / total;
    return 0.5 + recency * 0.5; // 0.5 ~ 1.0
  }
}
```

### 策略 2：工具调用记录复用

```typescript
// 从之前的工具调用结果中提取参数
class ToolResultExtractor implements ContextExtractor {
  constructor(private toolCallLog: ToolCallRecord[]) {}

  extract(history: Message[], paramName: string): InferredValue | null {
    // 查找最近产生相关数据的工具调用
    const relevantCalls = this.toolCallLog
      .filter(r => this.isRelevantParam(r, paramName))
      .sort((a, b) => b.timestamp - a.timestamp);

    if (relevantCalls.length === 0) return null;

    const latest = relevantCalls[0];
    const value = this.extractFromResult(latest.result, paramName);
    
    return value ? {
      value,
      source: `tool:${latest.toolName}`,
      confidence: 0.9, // 工具结果置信度高
    } : null;
  }
}

// 示例：用户说"帮我取消刚才查到的订单"
// 系统自动从 get_order 的结果里取 orderId
const toolLog: ToolCallRecord[] = [
  {
    toolName: 'get_order',
    args: { userId: '123' },
    result: { orderId: 'ORD-456', status: 'pending' },
    timestamp: Date.now() - 5000,
  }
];
```

### 策略 3：用户偏好 Profile

```typescript
// 持久化用户偏好，自动填充常用参数
class UserPreferenceExtractor implements ContextExtractor {
  constructor(
    private userId: string,
    private prefs: UserPreferences
  ) {}

  extract(_history: Message[], paramName: string): InferredValue | null {
    const prefKey = this.mapParamToPreference(paramName);
    const value = this.prefs.get(this.userId, prefKey);
    
    if (!value) return null;
    
    return {
      value,
      source: 'user_preference',
      confidence: 0.7, // 偏好置信度中等，用户习惯可能变
    };
  }

  // 学习：工具成功调用后，更新用户偏好
  async learn(paramName: string, value: unknown, success: boolean) {
    if (success) {
      const prefKey = this.mapParamToPreference(paramName);
      await this.prefs.set(this.userId, prefKey, value);
    }
  }
}
```

---

## 完整推断引擎

```typescript
interface InferredValue {
  value: unknown;
  source: string;      // 来源说明（透明度）
  confidence: number;  // 0.0 ~ 1.0
}

interface ParameterSpec {
  name: string;
  type: string;
  required: boolean;
  description: string;
}

class ToolParameterInferenceEngine {
  private extractors: ContextExtractor[];

  constructor(
    private history: Message[],
    private toolCallLog: ToolCallRecord[],
    private userId: string,
    private prefs: UserPreferences
  ) {
    this.extractors = [
      new ToolResultExtractor(toolCallLog),     // 优先：工具结果最准
      new ConversationHistoryExtractor(),        // 次之：对话历史
      new UserPreferenceExtractor(userId, prefs), // 兜底：用户偏好
    ];
  }

  async inferParameters(
    toolName: string,
    spec: ParameterSpec[],
    providedArgs: Record<string, unknown>
  ): Promise<InferenceResult> {
    const inferred: Record<string, InferredValue> = {};
    const needClarification: ParameterSpec[] = [];

    for (const param of spec) {
      // 已提供的参数直接跳过
      if (param.name in providedArgs) continue;
      // 非必需参数不推断
      if (!param.required) continue;

      // 依次尝试各提取器
      let found: InferredValue | null = null;
      for (const extractor of this.extractors) {
        found = extractor.extract(this.history, param.name);
        if (found && found.confidence >= CONFIDENCE_THRESHOLD) break;
      }

      if (found && found.confidence >= CONFIDENCE_THRESHOLD) {
        inferred[param.name] = found;
      } else {
        needClarification.push(param);
      }
    }

    return {
      resolvedArgs: {
        ...providedArgs,
        ...Object.fromEntries(
          Object.entries(inferred).map(([k, v]) => [k, v.value])
        ),
      },
      inferred,
      needClarification,
    };
  }
}

const CONFIDENCE_THRESHOLD = 0.6;

interface InferenceResult {
  resolvedArgs: Record<string, unknown>;
  inferred: Record<string, InferredValue>;
  needClarification: ParameterSpec[];
}
```

---

## 与 Agent Loop 集成

```typescript
// learn-claude-code 风格：在工具分发层注入推断
class SmartToolDispatcher {
  constructor(
    private tools: Map<string, ToolDefinition>,
    private engine: ToolParameterInferenceEngine
  ) {}

  async dispatch(
    toolName: string,
    rawArgs: Record<string, unknown>,
    history: Message[]
  ): Promise<ToolResult> {
    const tool = this.tools.get(toolName);
    if (!tool) throw new Error(`Unknown tool: ${toolName}`);

    // 1. 推断缺失参数
    const result = await this.engine.inferParameters(
      toolName,
      tool.parameterSpec,
      rawArgs
    );

    // 2. 还有缺失参数，需要澄清
    if (result.needClarification.length > 0) {
      return {
        type: 'clarification_needed',
        question: this.buildClarificationQuestion(
          toolName,
          result.needClarification
        ),
        partialArgs: result.resolvedArgs,
      };
    }

    // 3. 参数完整，执行工具
    // 如果有推断参数，向用户说明（透明度）
    if (Object.keys(result.inferred).length > 0) {
      const notes = Object.entries(result.inferred)
        .map(([k, v]) => `${k}="${v.value}" (来自 ${v.source})`)
        .join(', ');
      console.log(`🔍 自动推断参数：${notes}`);
    }

    return await tool.execute(result.resolvedArgs);
  }

  private buildClarificationQuestion(
    toolName: string,
    missingParams: ParameterSpec[]
  ): string {
    if (missingParams.length === 1) {
      return `执行 ${toolName} 需要 ${missingParams[0].description}，请提供？`;
    }
    const list = missingParams.map(p => `• ${p.description}`).join('\n');
    return `执行 ${toolName} 还需要以下信息：\n${list}`;
  }
}
```

---

## OpenClaw 实战：少问多做

```typescript
// OpenClaw skill 示例：send_message 自动补全
// 用户: "发消息给他说明天推迟"
// Agent 自动推断: recipient = 上次对话中提到的联系人

const sendMessageTool: ToolDefinition = {
  name: 'send_message',
  parameterSpec: [
    { name: 'recipient', type: 'string', required: true, description: '收件人' },
    { name: 'message', type: 'string', required: true, description: '消息内容' },
    { name: 'channel', type: 'string', required: false, description: '发送渠道' },
  ],
  execute: async (args) => {
    // 实际发送逻辑...
  }
};

// 测试推断链
async function demo() {
  const history: Message[] = [
    { role: 'user', content: '帮我找一下张三的联系方式' },
    { role: 'assistant', content: '张三的 Telegram 是 @zhangsan' },
    { role: 'user', content: '发消息给他说明天会议推迟到3点' },
  ];

  const dispatcher = new SmartToolDispatcher(tools, engine);
  const result = await dispatcher.dispatch(
    'send_message',
    { message: '明天会议推迟到3点' }, // 只提供了 message
    history
  );
  // 自动推断：recipient = "@zhangsan" (来自 conversation[1])
  // 输出：🔍 自动推断参数：recipient="@zhangsan" (来自 conversation[1])
}
```

---

## 推断质量的关键细节

### 1. 置信度校准

```typescript
// 不同来源的基础置信度
const BASE_CONFIDENCE = {
  tool_result: 0.90,        // 工具结果最可靠
  recent_message: 0.80,     // 最近消息
  older_message: 0.55,      // 较旧消息置信度低
  user_preference: 0.70,    // 用户偏好
  default_value: 0.40,      // 默认值兜底
};

// 类型匹配加分
function typeBonus(inferred: unknown, expectedType: string): number {
  if (typeof inferred === expectedType) return 0.1;
  return -0.2; // 类型不匹配降分
}
```

### 2. 推断透明度（避免"幽灵操作"）

```typescript
// ✅ 好的做法：告诉用户在推断什么
"正在给 @zhangsan 发消息（来自您刚才提到的联系人），继续吗？"

// ❌ 坏的做法：悄悄推断，用户不知道
// 直接发了，用户说"我没让你发给这个人！"
```

### 3. 推断失败的优雅降级

```typescript
// 只问最关键的一个参数，不要一次问很多
if (result.needClarification.length > 1) {
  // 按重要性排序，只问第一个
  const mostImportant = result.needClarification
    .sort((a, b) => a.importance - b.importance)[0];
  return askForOne(mostImportant);
}
```

---

## 总结

| 策略 | 来源 | 置信度 | 适用场景 |
|------|------|--------|----------|
| 工具结果复用 | 历史 tool call | 0.9 | "取消刚才查到的那个" |
| 对话历史提取 | 聊天记录 | 0.55~0.8 | "发给上次那个人" |
| 用户偏好 | 持久化 Profile | 0.7 | 常用参数（语言、时区） |
| 默认值兜底 | 配置 | 0.4 | 可选参数的合理默认 |

**核心原则：推断要透明，失败要优雅，只问最重要的那一个。**

---

## 参考

- [pi-mono tool dispatcher](https://github.com/badlogic/pi-mono) - TypeScript 生产实现
- [learn-claude-code agent loop](https://github.com/shareAI-lab/learn-claude-code) - Python 教学实现  
- Anthropic Docs: [Tool use with Claude](https://docs.anthropic.com/claude/docs/tool-use)
