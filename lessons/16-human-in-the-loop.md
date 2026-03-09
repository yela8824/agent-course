# Human-in-the-Loop：人机协作模式

Agent 不是要取代人类，而是和人类协作。本课讲解如何设计优雅的人机协作模式，让 Agent 知道什么时候该问、什么时候该等、什么时候该确认。

## 为什么需要 Human-in-the-Loop？

1. **安全性** - 高风险操作需要人类确认
2. **准确性** - Agent 不确定时主动询问比瞎猜好
3. **合规性** - 某些操作法律/业务上必须人类批准
4. **信任建立** - 让用户保持控制感

```
┌──────────────────────────────────────────────────────────────┐
│  Agent Autonomy Spectrum                                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Full Human    ←─────────────────────────→    Full Agent     │
│  Control                                      Autonomy       │
│                                                              │
│    Ask           Suggest         Act           Act           │
│    Every         & Wait         & Tell        Silently       │
│    Step                                                      │
│                                                              │
│    🔒              🤝             📢             🚀          │
│   安全模式       协作模式        通知模式       自主模式      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 核心设计模式

### 1. Confirmation Pattern（确认模式）

最常见的 HITL 模式：执行危险操作前请求确认。

**OpenClaw 实现 - 消息确认：**

```typescript
// pi-mono: src/tool-handler.ts
interface ConfirmationRequest {
  action: string;
  description: string;
  risk: 'low' | 'medium' | 'high' | 'critical';
  timeout?: number; // 超时自动取消
  defaultAction?: 'proceed' | 'cancel';
}

async function requestConfirmation(
  ctx: Context,
  request: ConfirmationRequest
): Promise<boolean> {
  // 发送确认请求
  const message = await ctx.channel.send({
    text: formatConfirmationMessage(request),
    buttons: [
      [
        { text: '✅ 确认执行', callback_data: `confirm:${request.id}:yes` },
        { text: '❌ 取消', callback_data: `confirm:${request.id}:no` }
      ]
    ]
  });
  
  // 等待用户响应
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      resolve(request.defaultAction === 'proceed');
    }, request.timeout ?? 300000); // 默认 5 分钟超时
    
    ctx.once(`confirmation:${request.id}`, (approved: boolean) => {
      clearTimeout(timeoutId);
      resolve(approved);
    });
  });
}
```

**使用示例：**

```typescript
// 执行危险操作前请求确认
async function handleDeleteFile(ctx: Context, filePath: string) {
  const confirmed = await requestConfirmation(ctx, {
    action: 'delete_file',
    description: `删除文件: ${filePath}`,
    risk: 'high',
    timeout: 60000,
    defaultAction: 'cancel'
  });
  
  if (!confirmed) {
    return { status: 'cancelled', message: '用户取消了删除操作' };
  }
  
  await fs.unlink(filePath);
  return { status: 'success', message: `已删除 ${filePath}` };
}
```

### 2. Approval Workflow（审批流程）

复杂操作需要多步审批。

```typescript
// OpenClaw 风格的审批流程
interface ApprovalStep {
  name: string;
  approver: 'user' | 'admin' | 'auto';
  condition?: (context: any) => boolean;
}

interface ApprovalWorkflow {
  steps: ApprovalStep[];
  onApproved: () => Promise<void>;
  onRejected: (step: string) => Promise<void>;
}

// 示例：发布代码的审批流程
const deployWorkflow: ApprovalWorkflow = {
  steps: [
    { name: 'code_review', approver: 'auto', condition: (ctx) => ctx.testsPass },
    { name: 'security_check', approver: 'auto', condition: (ctx) => !ctx.hasVulnerabilities },
    { name: 'final_approval', approver: 'user' }  // 人类最终确认
  ],
  onApproved: async () => { /* 执行部署 */ },
  onRejected: async (step) => { /* 通知失败原因 */ }
};
```

### 3. Escalation Pattern（升级模式）

Agent 遇到不确定情况时升级给人类。

```typescript
// learn-claude-code 风格
interface EscalationTrigger {
  condition: string;
  threshold: number;
  escalateTo: 'user' | 'admin' | 'oncall';
}

const escalationRules: EscalationTrigger[] = [
  // 置信度太低时升级
  { condition: 'confidence < 0.3', threshold: 0.3, escalateTo: 'user' },
  // 连续失败时升级
  { condition: 'consecutive_failures > 3', threshold: 3, escalateTo: 'admin' },
  // 花费超过阈值时升级
  { condition: 'estimated_cost > 10', threshold: 10, escalateTo: 'user' },
];

async function shouldEscalate(context: AgentContext): Promise<string | null> {
  if (context.confidence < 0.3) {
    return `我不太确定这个问题的答案，想请您确认一下...`;
  }
  if (context.consecutiveFailures > 3) {
    return `已经尝试了 ${context.consecutiveFailures} 次都失败了，需要您的帮助...`;
  }
  return null;
}
```

## OpenClaw 的 HITL 实现

OpenClaw 在多个层面实现了人机协作：

### 1. 工具级别控制

```yaml
# openclaw.yaml
tools:
  exec:
    ask: on-miss      # miss 时才问，hit 直接执行
    security: allowlist
    allowlist:
      - 'git *'
      - 'npm *'
  
  browser:
    ask: always       # 每次都问
  
  message:
    ask: off          # 不用问，直接执行
```

**ask 的三种模式：**

- `off` - 直接执行，不询问
- `on-miss` - 命令不在 allowlist 时询问
- `always` - 每次都询问

### 2. 安全分级

```typescript
// 工具风险分级
enum SecurityLevel {
  SAFE = 'safe',           // 读取操作，随便执行
  MODERATE = 'moderate',   // 写入操作，记录日志
  DANGEROUS = 'dangerous', // 删除/发送，需要确认
  CRITICAL = 'critical'    // 系统级，必须确认
}

const toolSecurityMap: Record<string, SecurityLevel> = {
  'read': SecurityLevel.SAFE,
  'web_search': SecurityLevel.SAFE,
  'write': SecurityLevel.MODERATE,
  'edit': SecurityLevel.MODERATE,
  'exec': SecurityLevel.DANGEROUS,
  'message:send': SecurityLevel.DANGEROUS,
  'gateway:restart': SecurityLevel.CRITICAL,
};
```

### 3. 内联按钮确认

```typescript
// Telegram 内联按钮实现
async function sendConfirmationWithButtons(
  ctx: TelegramContext,
  message: string,
  actionId: string
) {
  await ctx.reply(message, {
    reply_markup: {
      inline_keyboard: [[
        { text: '✅ Yes', callback_data: `confirm:${actionId}:yes` },
        { text: '❌ No', callback_data: `confirm:${actionId}:no` },
        { text: '📝 More Info', callback_data: `confirm:${actionId}:info` }
      ]]
    }
  });
}

// 处理回调
bot.on('callback_query', async (ctx) => {
  const [type, actionId, response] = ctx.callbackQuery.data.split(':');
  
  if (type === 'confirm') {
    switch (response) {
      case 'yes':
        await executeAction(actionId);
        await ctx.answerCbQuery('已执行');
        break;
      case 'no':
        await cancelAction(actionId);
        await ctx.answerCbQuery('已取消');
        break;
      case 'info':
        await showActionDetails(ctx, actionId);
        break;
    }
  }
});
```

## 实战：构建智能确认系统

### 风险评估器

```typescript
interface RiskAssessment {
  score: number;        // 0-100
  factors: string[];
  recommendation: 'auto' | 'confirm' | 'deny';
}

function assessRisk(action: ToolCall): RiskAssessment {
  let score = 0;
  const factors: string[] = [];
  
  // 1. 工具类型风险
  const toolRisk = getToolRiskScore(action.tool);
  score += toolRisk;
  if (toolRisk > 20) factors.push(`${action.tool} 是高风险工具`);
  
  // 2. 参数风险
  if (action.tool === 'exec') {
    const cmd = action.params.command;
    if (cmd.includes('rm ') || cmd.includes('delete')) {
      score += 30;
      factors.push('命令包含删除操作');
    }
    if (cmd.includes('sudo')) {
      score += 40;
      factors.push('需要管理员权限');
    }
  }
  
  // 3. 上下文风险
  if (isOutsideWorkingHours()) {
    score += 10;
    factors.push('非工作时间');
  }
  
  // 4. 历史风险
  if (hasRecentFailures(action.tool)) {
    score += 15;
    factors.push('该工具最近有失败记录');
  }
  
  return {
    score,
    factors,
    recommendation: score > 70 ? 'deny' : score > 30 ? 'confirm' : 'auto'
  };
}
```

### 智能确认消息生成

```typescript
function formatSmartConfirmation(
  action: ToolCall,
  risk: RiskAssessment
): string {
  const emoji = risk.score > 70 ? '🚨' : risk.score > 30 ? '⚠️' : '📋';
  
  let message = `${emoji} **请确认操作**\n\n`;
  message += `**操作**: ${action.tool}\n`;
  message += `**参数**: \`${JSON.stringify(action.params)}\`\n`;
  message += `**风险评分**: ${risk.score}/100\n`;
  
  if (risk.factors.length > 0) {
    message += `\n**风险因素**:\n`;
    risk.factors.forEach(f => message += `  • ${f}\n`);
  }
  
  if (risk.recommendation === 'deny') {
    message += `\n⛔ 建议: 不执行此操作`;
  }
  
  return message;
}
```

## 异步审批模式

有时候用户不能立即响应，需要异步处理。

```typescript
interface PendingApproval {
  id: string;
  action: ToolCall;
  createdAt: Date;
  expiresAt: Date;
  status: 'pending' | 'approved' | 'rejected' | 'expired';
}

class ApprovalQueue {
  private pending: Map<string, PendingApproval> = new Map();
  
  async requestApproval(action: ToolCall, ttl: number = 3600000): Promise<string> {
    const approval: PendingApproval = {
      id: crypto.randomUUID(),
      action,
      createdAt: new Date(),
      expiresAt: new Date(Date.now() + ttl),
      status: 'pending'
    };
    
    this.pending.set(approval.id, approval);
    
    // 通知用户
    await this.notifyUser(approval);
    
    // 设置过期定时器
    setTimeout(() => this.expire(approval.id), ttl);
    
    return approval.id;
  }
  
  async waitForApproval(id: string, timeout?: number): Promise<boolean> {
    const approval = this.pending.get(id);
    if (!approval) throw new Error('Approval not found');
    
    return new Promise((resolve) => {
      const checkInterval = setInterval(() => {
        const current = this.pending.get(id);
        if (!current || current.status !== 'pending') {
          clearInterval(checkInterval);
          resolve(current?.status === 'approved');
        }
      }, 1000);
      
      if (timeout) {
        setTimeout(() => {
          clearInterval(checkInterval);
          resolve(false);
        }, timeout);
      }
    });
  }
  
  approve(id: string) {
    const approval = this.pending.get(id);
    if (approval) approval.status = 'approved';
  }
  
  reject(id: string) {
    const approval = this.pending.get(id);
    if (approval) approval.status = 'rejected';
  }
}
```

## 最佳实践

### 1. 明确边界

```typescript
// 定义什么需要确认，什么不需要
const ALWAYS_AUTO = [
  'read file',
  'search web',
  'get time',
  'list directory'
];

const ALWAYS_CONFIRM = [
  'send email',
  'send message to external',
  'delete production data',
  'modify system config',
  'financial transactions'
];

const CONTEXT_DEPENDENT = [
  'write file',     // 工作目录内自动，外部确认
  'exec command',   // allowlist 内自动，外部确认
  'api call'        // 只读自动，写入确认
];
```

### 2. 渐进信任

```typescript
// 用户可以调整信任级别
interface TrustSettings {
  level: 'paranoid' | 'cautious' | 'balanced' | 'trusting' | 'full_auto';
  toolOverrides: Record<string, 'auto' | 'confirm' | 'deny'>;
  learnFromHistory: boolean;
}

function shouldConfirm(action: ToolCall, trust: TrustSettings): boolean {
  // 检查是否有明确的 override
  if (trust.toolOverrides[action.tool]) {
    return trust.toolOverrides[action.tool] === 'confirm';
  }
  
  // 根据信任级别决定
  switch (trust.level) {
    case 'paranoid': return true;  // 一切都要确认
    case 'full_auto': return false; // 一切都自动
    default:
      return assessRisk(action).recommendation === 'confirm';
  }
}
```

### 3. 优雅降级

```typescript
// 用户不响应时的处理
async function executeWithFallback(
  action: ToolCall,
  ctx: Context
): Promise<ToolResult> {
  const risk = assessRisk(action);
  
  if (risk.recommendation === 'auto') {
    return await executeTool(action);
  }
  
  try {
    const approved = await requestConfirmation(ctx, action, {
      timeout: 60000  // 1 分钟超时
    });
    
    if (approved) {
      return await executeTool(action);
    } else {
      return { status: 'cancelled', reason: 'user_rejected' };
    }
  } catch (e) {
    if (e instanceof TimeoutError) {
      // 超时处理
      if (risk.score < 20) {
        // 低风险，自动执行
        ctx.notify('确认超时，由于风险较低已自动执行');
        return await executeTool(action);
      } else {
        // 高风险，取消
        return { status: 'cancelled', reason: 'confirmation_timeout' };
      }
    }
    throw e;
  }
}
```

### 4. 审计日志

```typescript
// 记录所有人机交互
interface HITLLog {
  timestamp: Date;
  action: ToolCall;
  riskScore: number;
  confirmationRequested: boolean;
  userResponse: 'approved' | 'rejected' | 'timeout' | 'auto';
  responseTime?: number;  // 用户响应时间
  finalOutcome: 'executed' | 'cancelled';
}

// 用于后续分析和改进
async function logHITLInteraction(log: HITLLog) {
  await db.insert('hitl_logs', log);
  
  // 分析模式，优化默认行为
  if (log.userResponse === 'approved' && log.confirmationRequested) {
    // 用户每次都批准的操作，可以考虑降低确认频率
    await updateAutoApprovalScore(log.action.tool, +1);
  }
}
```

## 常见陷阱

### ❌ 确认疲劳

```typescript
// 错误：什么都要确认
async function handleAnyAction(action) {
  const confirmed = await confirm(action);  // 用户会烦死
  if (confirmed) execute(action);
}

// 正确：智能判断
async function handleAnyAction(action) {
  if (needsConfirmation(action)) {
    const confirmed = await confirm(action);
    if (!confirmed) return;
  }
  execute(action);
}
```

### ❌ 模糊的确认信息

```typescript
// 错误：用户不知道在确认什么
await confirm("确认执行操作？");

// 正确：清晰说明
await confirm(`确认删除 ${filePath}？此操作不可撤销。`);
```

### ❌ 没有超时处理

```typescript
// 错误：永远等待
const result = await waitForConfirmation();  // 可能永远阻塞

// 正确：设置合理超时
const result = await waitForConfirmation({ timeout: 300000 });
```

## 总结

Human-in-the-Loop 的核心原则：

1. **风险驱动** - 风险越高，人类参与越多
2. **上下文感知** - 同一操作在不同场景风险不同
3. **渐进信任** - 让用户控制自动化程度
4. **优雅降级** - 总有 fallback 方案
5. **透明记录** - 所有决策可追溯

Agent 的目标不是取代人类，而是成为人类的得力助手。知道什么时候该问、什么时候该等、什么时候该自己做决定，这才是真正智能的体现。

---

下节预告：**Multi-Model Orchestration** - 如何在一个 Agent 中协调使用多个模型
