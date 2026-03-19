# 93 - Agent PII 脱敏与隐私护栏（PII Masking & Privacy Guards）

> **核心思想**：Agent 每天处理用户数据——邮箱、手机号、API Key、信用卡号。一旦这些敏感信息被写入日志、传给 LLM、甚至回传给其他 Agent，就是一次隐私事故。隐私护栏的核心是：**在数据进入 LLM 前脱敏，在日志写出前屏蔽，在 Agent 间传递时只传必要信息**。

---

## 一、为什么 Agent 特别容易泄漏隐私

普通应用的数据流是确定的，Agent 的数据流是动态的：

```
用户消息 → 工具调用 → 工具结果 → 注入 Context → LLM 处理 → 输出
         ↑ 可能含 PII        ↑ 可能含 PII       ↑ 全进了 prompt
```

每一个箭头都是泄漏点：
- 工具拉回的数据库记录里有真实手机号
- LLM 回复里顺手带出了用户的 Email
- 日志记录了完整的 tool result（含支付信息）
- Sub-agent 通过消息传递了原始用户数据

---

## 二、PII 类型与检测策略

### 2.1 常见 PII 模式

```typescript
// pi-mono 风格：PII 类型注册表
interface PiiPattern {
  name: string;
  regex: RegExp;
  confidence: 'high' | 'medium' | 'low';
  mask: (match: string) => string;
}

const PII_PATTERNS: PiiPattern[] = [
  {
    name: 'email',
    regex: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
    confidence: 'high',
    mask: (m) => m.replace(/(?<=.{2}).(?=.*@)/g, '*'),
    // user@example.com → us**@example.com
  },
  {
    name: 'phone_cn',
    regex: /\b1[3-9]\d{9}\b/g,
    confidence: 'high',
    mask: (m) => m.slice(0, 3) + '****' + m.slice(7),
    // 13812345678 → 138****5678
  },
  {
    name: 'credit_card',
    regex: /\b(?:\d[ -]?){13,16}\b/g,
    confidence: 'medium',
    mask: (m) => m.replace(/\d(?=\d{4})/g, '*'),
    // 4111 1111 1111 1111 → **** **** **** 1111
  },
  {
    name: 'api_key',
    // 常见格式：sk-xxx, Bearer xxx, glsa_xxx
    regex: /\b(sk-[A-Za-z0-9]{20,}|glsa_[A-Za-z0-9_]+|Bearer\s+[A-Za-z0-9\-._~+/]+=*)\b/g,
    confidence: 'high',
    mask: (m) => m.slice(0, 6) + '...[REDACTED]',
  },
  {
    name: 'id_card_cn',
    regex: /\b\d{17}[\dX]\b/g,
    confidence: 'high',
    mask: (m) => m.slice(0, 6) + '********' + m.slice(14),
  },
];
```

### 2.2 检测 + 脱敏引擎

```typescript
interface MaskResult {
  masked: string;
  detections: Array<{
    type: string;
    original: string;
    position: [number, number];
  }>;
  hasPii: boolean;
}

class PiiGuard {
  private patterns: PiiPattern[];
  
  constructor(patterns = PII_PATTERNS) {
    this.patterns = patterns;
  }

  mask(text: string): MaskResult {
    let masked = text;
    const detections: MaskResult['detections'] = [];

    for (const pattern of this.patterns) {
      // 重置 regex 状态（全局 regex 有 lastIndex 问题）
      const regex = new RegExp(pattern.regex.source, pattern.regex.flags);
      let match: RegExpExecArray | null;
      
      while ((match = regex.exec(text)) !== null) {
        detections.push({
          type: pattern.name,
          original: match[0],
          position: [match.index, match.index + match[0].length],
        });
      }
      
      // 替换
      masked = masked.replace(new RegExp(pattern.regex.source, pattern.regex.flags), 
        (m) => pattern.mask(m));
    }

    return {
      masked,
      detections,
      hasPii: detections.length > 0,
    };
  }

  // 只检测，不脱敏（用于告警）
  detect(text: string): boolean {
    return this.patterns.some(p => 
      new RegExp(p.regex.source, p.regex.flags).test(text)
    );
  }
}
```

---

## 三、在 Agent 工具管道中挂载护栏

### 3.1 工具结果过滤（Tool Result Interceptor）

```typescript
// OpenClaw/pi-mono 风格的工具结果拦截器
class PrivacyAwareToolExecutor {
  private guard = new PiiGuard();
  private logger: Logger;

  async executeTool(
    tool: Tool,
    params: unknown,
    options: { maskInput?: boolean; maskOutput?: boolean } = {}
  ): Promise<ToolResult> {
    
    // 1. 入参检测（防止把用户原始数据直接传给外部 API）
    const inputStr = JSON.stringify(params);
    if (options.maskInput !== false && this.guard.detect(inputStr)) {
      this.logger.warn('PII detected in tool input', {
        tool: tool.name,
        // 日志里不记录原始值，只记录类型
        piiTypes: this.guard.mask(inputStr).detections.map(d => d.type),
      });
    }

    // 2. 执行工具
    const rawResult = await tool.execute(params);

    // 3. 输出脱敏
    if (options.maskOutput !== false) {
      const resultStr = JSON.stringify(rawResult);
      const { masked, detections } = this.guard.mask(resultStr);
      
      if (detections.length > 0) {
        this.logger.info('PII masked from tool result', {
          tool: tool.name,
          count: detections.length,
          types: [...new Set(detections.map(d => d.type))],
        });
        
        return JSON.parse(masked);
      }
    }

    return rawResult;
  }
}
```

### 3.2 Context 注入前的清洗

```typescript
// 在把工具结果注入 LLM Context 之前过滤
class ContextSanitizer {
  private guard = new PiiGuard();
  
  // 选择性暴露：只给 LLM 它需要的字段
  sanitizeForLlm<T extends Record<string, unknown>>(
    data: T,
    allowList: (keyof T)[]
  ): Partial<T> {
    const filtered = Object.fromEntries(
      allowList
        .filter(k => k in data)
        .map(k => [k, data[k]])
    ) as Partial<T>;
    
    // 二次扫描剩余字段中的 PII
    const { masked } = this.guard.mask(JSON.stringify(filtered));
    return JSON.parse(masked);
  }

  // 用于 system prompt 注入
  sanitizeSystemContext(context: AgentContext): string {
    // 用户信息只传 ID，不传手机/邮箱
    return `User ID: ${context.userId}\nSession: ${context.sessionId}`;
    // ❌ 不要: `User: ${context.phone} / ${context.email}`
  }
}
```

---

## 四、日志层防护

```typescript
// 结构化日志中间件，自动过滤敏感字段
const SENSITIVE_KEYS = new Set([
  'password', 'passwd', 'secret', 'token', 'apiKey', 'api_key',
  'authorization', 'cookie', 'creditCard', 'ssn', 'idCard',
  'phone', 'email', 'address',
]);

function sanitizeLogObject(obj: unknown, depth = 0): unknown {
  if (depth > 10) return '[MAX_DEPTH]';
  if (typeof obj !== 'object' || obj === null) return obj;
  if (Array.isArray(obj)) return obj.map(i => sanitizeLogObject(i, depth + 1));
  
  return Object.fromEntries(
    Object.entries(obj as Record<string, unknown>).map(([k, v]) => {
      if (SENSITIVE_KEYS.has(k.toLowerCase())) {
        return [k, '[REDACTED]'];
      }
      return [k, sanitizeLogObject(v, depth + 1)];
    })
  );
}

// 在 OpenClaw 的工具执行日志里使用
logger.info('Tool executed', sanitizeLogObject({
  tool: 'getUserProfile',
  params: { userId: '123' },
  result: {
    id: '123',
    email: 'user@example.com',  // → [REDACTED]
    phone: '13812345678',        // → [REDACTED]
    username: 'alice',           // 保留
  },
}));
```

---

## 五、Sub-agent 间数据传递的最小化原则

```typescript
// ❌ 错误：把完整用户对象传给 Sub-agent
await subagent.run({
  task: '给用户发优惠券',
  context: {
    user: fullUserRecord,  // 含 PII
  },
});

// ✅ 正确：只传必要的最小数据集
await subagent.run({
  task: '给用户发优惠券',
  context: {
    userId: user.id,           // 只传 ID
    preferredChannel: 'email', // 只传行为相关的元数据
    couponCode: 'SAVE20',
    // Sub-agent 自己去查需要的信息，减少 PII 在系统中的流动
  },
});
```

---

## 六、OpenClaw 实战：给工具执行器加 Privacy 中间件

OpenClaw 的工具执行经过 `ToolPolicyPipeline`，可以在这里挂载：

```typescript
// ~/.openclaw/workspace/tools/privacy-middleware.ts
import { ToolMiddleware } from '@openclaw/core';

export const privacyMiddleware: ToolMiddleware = {
  name: 'privacy-guard',
  priority: 100, // 最高优先级，最先执行
  
  async before(ctx) {
    // 扫描工具参数
    const scan = piiGuard.mask(JSON.stringify(ctx.params));
    if (scan.hasPii) {
      ctx.metadata.piiDetectedInInput = true;
      ctx.metadata.piiTypes = scan.detections.map(d => d.type);
    }
  },
  
  async after(ctx, result) {
    // 脱敏工具结果
    if (shouldMask(ctx.tool.name)) {
      const { masked, detections } = piiGuard.mask(JSON.stringify(result));
      if (detections.length > 0) {
        ctx.metadata.piiMaskedFromOutput = detections.length;
        return JSON.parse(masked);
      }
    }
    return result;
  },
};

// 可配置的白名单（某些工具需要处理原始 PII）
function shouldMask(toolName: string): boolean {
  const exemptions = ['send_verification_sms', 'user_kyc_check'];
  return !exemptions.includes(toolName);
}
```

---

## 七、learn-claude-code 对比：Claude 的内置隐私行为

Claude Code 本身有一些隐私行为值得借鉴：

1. **不主动记忆 PII**：Claude 不会把用户的手机号存进记忆系统
2. **工具调用透明**：每次工具调用都对用户可见（可审计）
3. **上下文隔离**：不同会话的上下文不互相污染

**我们在 Agent 里复刻这些原则：**
- ✅ 脱敏后写 memory，不写原始 PII
- ✅ 工具调用日志可审计（结构化 + 索引）
- ✅ 多租户场景中 userId 做 namespace 隔离

---

## 八、总结

| 防护层 | 策略 | 实现位置 |
|--------|------|----------|
| 工具入参 | 检测 + 告警，必要时拦截 | ToolMiddleware.before |
| 工具结果 | 字段级脱敏，注入 LLM 前清洗 | ToolMiddleware.after |
| 日志输出 | 敏感字段替换为 [REDACTED] | Logger 中间件 |
| Context | 最小化原则，只传 ID | ContextSanitizer |
| Sub-agent | 数据最小化传递 | Agent 任务定义规范 |
| Memory | 存脱敏版本 | Memory 写入前过滤 |

**一句话总结**：**隐私护栏不是加密，是隔离——让 PII 只在需要它的地方出现，其他地方一律脱敏或替换为 ID。**
