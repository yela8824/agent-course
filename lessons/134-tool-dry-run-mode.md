# 134 - Agent 工具调用 Dry-Run 模式（Tool Call Dry-Run Mode）

> 核心问题：Agent 准备删库跑路，你怎么让它先"演练一遍"再真正执行？

---

## 为什么需要 Dry-Run？

生产环境有一类操作是"执行就无法撤销"的：

- 删除文件/数据库记录
- 发送邮件/消息
- 部署代码到生产环境
- 转账/支付
- 调用第三方付费 API

如果直接让 Agent 执行，出了问题后悔莫及。  
**Dry-Run 模式**让 Agent 先"规划"所有工具调用，展示执行计划，确认后再真正执行。

---

## 三种 Dry-Run 策略

```
策略1: 全局 Dry-Run ── 所有工具只模拟，不执行
策略2: 标签过滤 ──── 危险工具模拟，安全工具真执行  
策略3: 两阶段执行 ── 先规划全部调用，审批后批量执行
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
// dry-run-registry.ts

type ToolRisk = 'safe' | 'moderate' | 'destructive';

interface ToolMeta {
  risk: ToolRisk;
  dryRunResponse?: (params: unknown) => unknown; // 模拟返回值
}

interface DryRunResult {
  toolName: string;
  params: unknown;
  risk: ToolRisk;
  simulatedResponse: unknown;
  wouldExecute: boolean;
}

class DryRunToolRegistry {
  private tools = new Map<string, { fn: Function; meta: ToolMeta }>();
  private dryRunLog: DryRunResult[] = [];

  register(name: string, fn: Function, meta: ToolMeta) {
    this.tools.set(name, { fn, meta });
  }

  // 创建工具调用包装器
  createExecutor(dryRun: boolean, approvedRisks: ToolRisk[] = ['safe']) {
    return async (toolName: string, params: unknown): Promise<unknown> => {
      const tool = this.tools.get(toolName);
      if (!tool) throw new Error(`Unknown tool: ${toolName}`);

      const { fn, meta } = tool;
      const shouldExecute = !dryRun || approvedRisks.includes(meta.risk);

      if (!shouldExecute) {
        // Dry-run: 记录意图，返回模拟结果
        const simulated = meta.dryRunResponse
          ? meta.dryRunResponse(params)
          : { dryRun: true, message: `[DRY-RUN] Would call ${toolName}`, params };

        this.dryRunLog.push({
          toolName,
          params,
          risk: meta.risk,
          simulatedResponse: simulated,
          wouldExecute: true,
        });

        return simulated;
      }

      // 真实执行
      return fn(params);
    };
  }

  getDryRunPlan(): DryRunResult[] {
    return [...this.dryRunLog];
  }

  clearLog() {
    this.dryRunLog = [];
  }
}

// 使用示例
const registry = new DryRunToolRegistry();

registry.register('send_email', sendEmail, {
  risk: 'destructive',
  dryRunResponse: (params: any) => ({
    dryRun: true,
    wouldSend: { to: params.to, subject: params.subject },
    estimatedDelivery: '~2 seconds',
  }),
});

registry.register('read_file', readFile, {
  risk: 'safe',
  // 无 dryRunResponse → 直接真实执行（读操作安全）
});

registry.register('delete_records', deleteRecords, {
  risk: 'destructive',
  dryRunResponse: (params: any) => ({
    dryRun: true,
    wouldDelete: `${params.ids.length} records`,
    ids: params.ids,
  }),
});
```

---

## 两阶段执行：先规划，后审批

```typescript
// two-phase-executor.ts

interface ExecutionPlan {
  steps: Array<{
    toolName: string;
    params: unknown;
    risk: ToolRisk;
    estimatedEffect: string;
  }>;
  requiresApproval: boolean;
  highRiskCount: number;
}

async function planAndExecute(
  userRequest: string,
  registry: DryRunToolRegistry,
): Promise<void> {
  // 阶段1: Dry-run 模式运行，收集调用计划
  console.log('📋 Planning phase...');
  const dryRunExecutor = registry.createExecutor(true); // dryRun = true
  
  await runAgentLoop(userRequest, dryRunExecutor); // Agent 正常跑，但所有危险工具只记录
  
  const plan = registry.getDryRunPlan();
  
  if (plan.length === 0) {
    console.log('✅ No tool calls planned, nothing to approve.');
    return;
  }

  // 展示计划
  const highRisk = plan.filter(p => p.risk === 'destructive');
  console.log('\n🔍 Execution Plan:');
  plan.forEach((step, i) => {
    const icon = step.risk === 'destructive' ? '⚠️' : step.risk === 'moderate' ? '⚡' : '✅';
    console.log(`  ${i + 1}. ${icon} ${step.toolName}(${JSON.stringify(step.params)})`);
  });

  if (highRisk.length > 0) {
    console.log(`\n⚠️  ${highRisk.length} destructive operations require approval.`);
    const approved = await promptUserApproval(plan);
    if (!approved) {
      console.log('❌ Execution cancelled by user.');
      return;
    }
  }

  // 阶段2: 真实执行
  console.log('\n🚀 Executing approved plan...');
  registry.clearLog();
  const realExecutor = registry.createExecutor(false); // dryRun = false
  await runAgentLoop(userRequest, realExecutor); // 重新执行，这次是真的
  console.log('✅ Done.');
}
```

---

## Python 实现（learn-claude-code 风格）

```python
# dry_run_decorator.py

from functools import wraps
from typing import Any, Callable, Optional
from dataclasses import dataclass, field
from enum import Enum
import contextvars

class ToolRisk(Enum):
    SAFE = "safe"
    MODERATE = "moderate"
    DESTRUCTIVE = "destructive"

# 用 contextvar 控制当前是否 dry-run（线程安全）
_dry_run_ctx: contextvars.ContextVar[bool] = contextvars.ContextVar(
    'dry_run', default=False
)

@dataclass
class DryRunRecord:
    tool_name: str
    params: dict
    risk: ToolRisk
    simulated_response: Any

_dry_run_log: list[DryRunRecord] = []

def tool(
    risk: ToolRisk = ToolRisk.SAFE,
    dry_run_response: Optional[Callable] = None
):
    """装饰器：声明工具的风险等级和 dry-run 行为"""
    def decorator(fn: Callable) -> Callable:
        @wraps(fn)
        def wrapper(*args, **kwargs) -> Any:
            if _dry_run_ctx.get() and risk != ToolRisk.SAFE:
                # Dry-run 模式：记录意图，不执行
                simulated = (
                    dry_run_response(*args, **kwargs)
                    if dry_run_response
                    else {"dry_run": True, "tool": fn.__name__, "params": kwargs}
                )
                _dry_run_log.append(DryRunRecord(
                    tool_name=fn.__name__,
                    params=kwargs,
                    risk=risk,
                    simulated_response=simulated,
                ))
                return simulated
            return fn(*args, **kwargs)
        
        wrapper._risk = risk  # 便于工具注册时读取
        return wrapper
    return decorator


# 声明工具
@tool(
    risk=ToolRisk.DESTRUCTIVE,
    dry_run_response=lambda to, subject, body: {
        "dry_run": True,
        "would_send": {"to": to, "subject": subject},
        "preview": body[:100] + "..."
    }
)
def send_email(to: str, subject: str, body: str) -> dict:
    """真实发送邮件"""
    import smtplib
    # ... 实际发送逻辑
    return {"sent": True, "message_id": "xxx"}


@tool(risk=ToolRisk.SAFE)
def get_user_info(user_id: str) -> dict:
    """读操作，safe 级别，dry-run 模式也真实执行"""
    return db.query(f"SELECT * FROM users WHERE id = {user_id}")


# 两阶段执行上下文管理器
from contextlib import contextmanager

@contextmanager
def dry_run_context():
    """with dry_run_context(): 块内所有危险工具调用都是模拟"""
    token = _dry_run_ctx.set(True)
    _dry_run_log.clear()
    try:
        yield _dry_run_log
    finally:
        _dry_run_ctx.reset(token)


# Agent 使用示例
async def safe_agent_execute(task: str, agent_fn: Callable):
    # 第一阶段：dry-run 收集计划
    with dry_run_context() as plan:
        await agent_fn(task)
    
    if not plan:
        # 没有危险操作，直接执行
        return await agent_fn(task)
    
    # 展示计划
    destructive = [r for r in plan if r.risk == ToolRisk.DESTRUCTIVE]
    print(f"\n计划执行 {len(plan)} 个工具调用，其中 {len(destructive)} 个危险操作：")
    for r in destructive:
        print(f"  ⚠️  {r.tool_name}({r.params})")
        print(f"     预计效果: {r.simulated_response}")
    
    confirm = input("\n确认执行？(y/N) ")
    if confirm.lower() != 'y':
        print("已取消。")
        return None
    
    # 第二阶段：真实执行
    return await agent_fn(task)
```

---

## OpenClaw 集成：Telegram inline 审批

```typescript
// openclaw-dry-run-approval.ts
// 利用 OpenClaw 的 inline buttons 发送审批请求

async function requestApprovalViaTelegram(
  plan: DryRunResult[],
  chatId: string,
): Promise<boolean> {
  const summary = plan
    .filter(p => p.risk === 'destructive')
    .map(p => `• ${p.toolName}: ${JSON.stringify(p.params)}`)
    .join('\n');

  // 发送审批消息（通过 message tool 的 inline buttons）
  const msg = await sendTelegramMessage({
    chat_id: chatId,
    text: `⚠️ **Agent 执行计划需要审批**\n\n危险操作:\n${summary}`,
    reply_markup: {
      inline_keyboard: [[
        { text: '✅ 批准执行', callback_data: 'approve' },
        { text: '❌ 取消', callback_data: 'cancel' },
      ]],
    },
  });

  // 等待用户点击（30秒超时）
  const approval = await waitForCallback(msg.message_id, 30_000);
  return approval === 'approve';
}
```

---

## 实战：Agent 部署流水线的 Dry-Run

```typescript
// deployment-agent-dry-run.ts

const deploymentRegistry = new DryRunToolRegistry();

deploymentRegistry.register('run_tests', runTests, { risk: 'safe' });
deploymentRegistry.register('build_docker', buildDocker, { risk: 'moderate' });
deploymentRegistry.register('push_to_registry', pushImage, {
  risk: 'moderate',
  dryRunResponse: (p) => ({ dryRun: true, wouldPush: `${p.image}:${p.tag}` }),
});
deploymentRegistry.register('deploy_to_production', deployProd, {
  risk: 'destructive',
  dryRunResponse: (p) => ({
    dryRun: true,
    wouldDeploy: p.service,
    estimatedDowntime: '~30s rolling update',
    affectedInstances: p.instances ?? 3,
  }),
});
deploymentRegistry.register('notify_slack', notifySlack, {
  risk: 'moderate',
  dryRunResponse: (p) => ({ dryRun: true, wouldNotify: `#${p.channel}` }),
});

// 用户触发
await planAndExecute(
  'Deploy v2.1.0 of the payment service to production',
  deploymentRegistry,
);

// 输出：
// 📋 Planning phase...
// 
// 🔍 Execution Plan:
//   1. ✅ run_tests({ service: "payment", version: "v2.1.0" })
//   2. ⚡ build_docker({ tag: "v2.1.0" })
//   3. ⚡ push_to_registry({ image: "payment", tag: "v2.1.0" })
//   4. ⚠️  deploy_to_production({ service: "payment", instances: 3 })
//   5. ⚡ notify_slack({ channel: "deployments" })
// 
// ⚠️  1 destructive operations require approval.
// 确认部署到生产？(y/N)
```

---

## 关键设计原则

| 原则 | 说明 |
|------|------|
| **幂等 Dry-Run** | 同样的输入，dry-run 永远返回相同的模拟结果 |
| **安全操作不拦截** | 读操作、查询类工具跳过 dry-run，避免信息缺失导致计划不准 |
| **两阶段重跑** | Phase 2 完整重跑（不是"回放" Phase 1 的记录），保证结果一致性 |
| **超时保护** | 审批请求设超时，超时自动取消，不挂起 Agent |
| **日志清晰** | Dry-run 的返回值要明确标注 `dryRun: true`，避免混淆 |

---

## 与其他模式的配合

```
Dry-Run ──→ Human-in-the-Loop (16)  审批流程
         ──→ Command Pattern (125)   记录可撤销的命令队列
         ──→ Audit Logging (115)     记录"计划了但未执行"的意图
         ──→ Tool Budget (133)       Dry-run 阶段预估成本，超预算直接拒绝
```

---

## 小结

Dry-Run 模式的本质是**把 Agent 的"想法"和"行动"分开**：

1. **规划阶段**：让 Agent 自由思考、调用工具、收集意图（模拟执行）
2. **审批阶段**：展示计划，人工或自动审批
3. **执行阶段**：确认后真正执行

生产级 Agent 碰到危险操作时，"先演练后执行"是比事后撤销更可靠的安全网。
