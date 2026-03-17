# 71 - Agent Rollback & State Recovery
# Agent 回滚与状态恢复：用 Saga 模式处理分布式副作用

## 核心问题

Agent 执行工具调用时会产生**真实的副作用**：
- 发了一封邮件 → 不能"撤回"
- 写了数据库 → 可以回滚
- 扣了余额 → 需要补偿事务
- 部署了代码 → 可以回退版本

当 Agent 执行到第 3 步失败时，前两步已完成的操作怎么办？

---

## Saga 模式：分布式事务的正确打开方式

传统数据库有 ACID 事务，但 Agent 调用的工具分散在各处（API、DB、文件系统），
无法用单一事务包裹。Saga 模式的答案：**每个步骤都配一个补偿操作**。

```
正向步骤（T）           补偿操作（C）
─────────────────────────────────────
T1: 扣库存          ← C1: 还库存
T2: 创建订单        ← C2: 取消订单
T3: 扣余额          ← C3: 退款
T4: 发邮件          ← C4: (幂等，不需要补偿)
```

失败时：按反序执行 C3 → C2 → C1（C4 不可逆但幂等，接受）

---

## Python 实现（learn-claude-code 风格）

```python
from dataclasses import dataclass, field
from typing import Callable, Any
import logging

logger = logging.getLogger(__name__)

@dataclass
class SagaStep:
    name: str
    action: Callable[..., Any]
    compensate: Callable[..., Any] | None = None
    result: Any = None
    executed: bool = False

class SagaExecutor:
    """
    Saga 执行器：顺序执行步骤，失败时自动补偿已完成的步骤。
    
    用法：
        saga = SagaExecutor()
        saga.add_step("扣库存", deduct_stock, restore_stock)
        saga.add_step("创建订单", create_order, cancel_order)
        saga.add_step("扣余额", charge_balance, refund_balance)
        await saga.execute(payload)
    """
    
    def __init__(self):
        self.steps: list[SagaStep] = []
        self.completed: list[SagaStep] = []
    
    def add_step(
        self,
        name: str,
        action: Callable,
        compensate: Callable | None = None
    ):
        self.steps.append(SagaStep(name, action, compensate))
    
    async def execute(self, ctx: dict) -> dict:
        for step in self.steps:
            try:
                logger.info(f"[Saga] 执行: {step.name}")
                step.result = await step.action(ctx)
                step.executed = True
                self.completed.append(step)
                # 把结果传递给后续步骤
                ctx[step.name] = step.result
                
            except Exception as e:
                logger.error(f"[Saga] 步骤 {step.name} 失败: {e}")
                await self._rollback(ctx)
                raise SagaRollbackError(
                    f"Saga 在 {step.name} 失败，已回滚 {len(self.completed)} 个步骤",
                    failed_step=step.name,
                    original_error=e
                )
        
        return ctx
    
    async def _rollback(self, ctx: dict):
        """反序执行补偿操作"""
        for step in reversed(self.completed):
            if step.compensate is None:
                logger.info(f"[Saga] 跳过补偿: {step.name}（不可逆操作，已接受）")
                continue
            try:
                logger.info(f"[Saga] 补偿: {step.name}")
                await step.compensate(ctx)
            except Exception as e:
                # 补偿失败是严重问题，记录但继续补偿其他步骤
                logger.critical(f"[Saga] 补偿 {step.name} 失败！需人工介入: {e}")


class SagaRollbackError(Exception):
    def __init__(self, message, failed_step, original_error):
        super().__init__(message)
        self.failed_step = failed_step
        self.original_error = original_error


# ── 实际 Agent 工具中使用 ──

async def process_order_tool(payload: dict) -> dict:
    saga = SagaExecutor()
    
    saga.add_step(
        "扣库存",
        action=lambda ctx: inventory_service.deduct(payload["item_id"], payload["qty"]),
        compensate=lambda ctx: inventory_service.restore(payload["item_id"], payload["qty"])
    )
    saga.add_step(
        "创建订单",
        action=lambda ctx: order_service.create(payload, ctx["扣库存"]["reservation_id"]),
        compensate=lambda ctx: order_service.cancel(ctx["创建订单"]["order_id"])
    )
    saga.add_step(
        "扣余额",
        action=lambda ctx: wallet_service.charge(payload["user_id"], payload["amount"]),
        compensate=lambda ctx: wallet_service.refund(payload["user_id"], payload["amount"])
    )
    saga.add_step(
        "发送确认邮件",
        action=lambda ctx: email_service.send_confirmation(ctx["创建订单"]["order_id"]),
        compensate=None  # 邮件不可撤回，接受
    )
    
    try:
        result = await saga.execute({})
        return {"status": "success", "order_id": result["创建订单"]["order_id"]}
    except SagaRollbackError as e:
        return {
            "status": "failed",
            "failed_step": e.failed_step,
            "message": str(e)
        }
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
// src/core/saga.ts

type StepFn<T> = (ctx: T) => Promise<Partial<T>>;

interface SagaStep<T> {
  name: string;
  action: StepFn<T>;
  compensate?: StepFn<T>;
}

export class Saga<T extends Record<string, unknown>> {
  private steps: SagaStep<T>[] = [];
  private executed: SagaStep<T>[] = [];

  step(name: string, action: StepFn<T>, compensate?: StepFn<T>): this {
    this.steps.push({ name, action, compensate });
    return this; // 链式调用
  }

  async run(initialCtx: T): Promise<T> {
    let ctx = { ...initialCtx };

    for (const step of this.steps) {
      try {
        console.log(`[Saga] → ${step.name}`);
        const result = await step.action(ctx);
        ctx = { ...ctx, ...result };
        this.executed.push(step);
      } catch (err) {
        console.error(`[Saga] ✗ ${step.name} 失败，开始回滚...`);
        await this.rollback(ctx);
        throw new Error(
          `Saga failed at "${step.name}": ${(err as Error).message}`
        );
      }
    }

    return ctx;
  }

  private async rollback(ctx: T): Promise<void> {
    for (const step of [...this.executed].reverse()) {
      if (!step.compensate) {
        console.log(`[Saga] ⚠ 跳过补偿: ${step.name}`);
        continue;
      }
      try {
        console.log(`[Saga] ↩ 补偿: ${step.name}`);
        await step.compensate(ctx);
      } catch (err) {
        console.error(`[Saga] 💥 补偿失败 ${step.name}:`, err);
        // 继续补偿其他步骤
      }
    }
  }
}

// ── 在 Agent 工具中使用 ──
import { Saga } from '../core/saga';

export async function deployFeatureTool(params: {
  repoId: string;
  version: string;
  userId: string;
}) {
  const result = await new Saga<DeployCtx>()
    .step(
      'build',
      async (ctx) => {
        const buildId = await ci.triggerBuild(params.repoId, params.version);
        return { buildId };
      },
      async (ctx) => {
        if (ctx.buildId) await ci.cancelBuild(ctx.buildId);
      }
    )
    .step(
      'deploy',
      async (ctx) => {
        const deployId = await k8s.deploy(ctx.buildId!, params.version);
        return { deployId };
      },
      async (ctx) => {
        if (ctx.deployId) await k8s.rollback(ctx.deployId);
      }
    )
    .step(
      'notify',
      async (ctx) => {
        await slack.notify(`部署成功: ${params.version}`);
        return {};
      }
      // 通知不补偿
    )
    .run({ userId: params.userId } as DeployCtx);

  return result;
}
```

---

## OpenClaw 中的实际应用

OpenClaw 的 `sessions_spawn` 本身就有 Saga 语义：
- 子 Agent 失败 → 父 Agent 收到错误，可以触发补偿
- `cleanup: "delete"` 参数就是一种自动补偿（清理临时会话）

```typescript
// OpenClaw sessions_spawn 的 Saga 封装示例
async function spawnWithRollback(task: string, onFailure: () => Promise<void>) {
  try {
    const result = await sessionsSpawn({
      task,
      cleanup: "delete",  // 失败时自动清理
      runTimeoutSeconds: 300,
    });
    return result;
  } catch (err) {
    await onFailure();  // 执行业务层补偿
    throw err;
  }
}
```

---

## 关键设计原则

### 1. 幂等补偿
补偿操作必须可以安全地多次执行：

```python
async def refund_balance(ctx: dict):
    # ❌ 错误：重复执行会多次退款
    await wallet.add(ctx["user_id"], ctx["amount"])
    
    # ✅ 正确：检查幂等 key
    await wallet.refund_if_not_refunded(
        order_id=ctx["order_id"],
        amount=ctx["amount"]
    )
```

### 2. 不可逆操作要最后执行
```python
saga.add_step("发短信", send_sms, compensate=None)  # 放最后
```

### 3. 持久化 Saga 状态（防止进程崩溃）
```python
# 每步执行后写入 DB
await saga_log.record(saga_id, step_name, "completed")

# 启动时检查未完成的 Saga，继续执行或补偿
incomplete = await saga_log.find_incomplete()
for saga in incomplete:
    await resume_or_rollback(saga)
```

---

## 总结

| 场景 | 推荐方案 |
|------|---------|
| 可数据库事务的操作 | 用 DB Transaction，不用 Saga |
| 跨服务的 Agent 工具调用 | Saga + 补偿操作 |
| 不可逆操作（发邮件/短信） | 放最后，设计为幂等，接受偶发重复 |
| 长时间运行的 Saga | 持久化步骤状态，支持断点续传 |
| 补偿失败 | 记录到 DLQ，触发人工告警 |

**核心思想**：与其期望每步都成功，不如为每步失败做好准备。
Saga 模式让 Agent 的副作用可控、可追溯、可恢复。
