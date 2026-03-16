# 65 - Agent Workflow DSL：声明式工作流管道

> 与其让 Agent 在运行时"想到哪做到哪"，不如把执行逻辑写成可读、可测试、可版本化的工作流定义。

---

## 为什么需要 Workflow DSL？

当你的 Agent 只做一件事，逻辑写在 loop 里没问题。
但当你要：

- **串行多个子任务**（先搜索 → 再总结 → 再发邮件）
- **条件分支**（如果找到结果走 A，否则走 B）
- **并行 fan-out**（同时查询 3 个数据源）
- **重试某些步骤**，跳过另一些

…这时候"在 prompt 里写逻辑"就失控了。

**Workflow DSL 的核心思想**：把"做什么"（业务逻辑）和"怎么做"（执行引擎）分离。

---

## 最简 DSL 设计

一个 Workflow = 有向无环图（DAG），节点是步骤，边是依赖关系。

```yaml
# workflow.yaml
name: research_and_report
steps:
  - id: search
    tool: web_search
    input:
      query: "{{ params.topic }}"

  - id: summarize
    tool: llm_summarize
    depends_on: [search]
    input:
      content: "{{ steps.search.result }}"

  - id: send_report
    tool: send_email
    depends_on: [summarize]
    input:
      to: "{{ params.email }}"
      body: "{{ steps.summarize.result }}"
```

三个关键概念：
1. **`depends_on`** — 声明依赖，引擎自动并行没有依赖关系的步骤
2. **`{{ template }}`** — 运行时插值，把上一步结果注入下一步
3. **`tool`** — 解耦：DSL 只知道工具名，不知道实现

---

## Python 实现：miniflow

```python
# miniflow.py - 一个 60 行的 Workflow 执行引擎
import asyncio
import re
from typing import Any, Callable
import yaml

class WorkflowEngine:
    def __init__(self):
        self.tools: dict[str, Callable] = {}
    
    def register(self, name: str):
        """注册工具"""
        def decorator(fn):
            self.tools[name] = fn
            return fn
        return decorator
    
    def _render(self, template: Any, context: dict) -> Any:
        """简单模板引擎：替换 {{ expression }}"""
        if isinstance(template, str):
            def replace(m):
                expr = m.group(1).strip()
                # 支持 steps.xxx.result 和 params.xxx
                parts = expr.split(".")
                val = context
                for p in parts:
                    val = val[p]
                return str(val)
            return re.sub(r'\{\{(.+?)\}\}', replace, template)
        elif isinstance(template, dict):
            return {k: self._render(v, context) for k, v in template.items()}
        return template
    
    async def run(self, workflow_def: dict, params: dict = {}) -> dict:
        steps = {s["id"]: s for s in workflow_def["steps"]}
        results = {}
        context = {"params": params, "steps": {}}
        completed = set()
        
        async def execute_step(step_id: str):
            step = steps[step_id]
            # 等待所有依赖完成
            deps = step.get("depends_on", [])
            await asyncio.gather(*[
                execute_step(d) for d in deps if d not in completed
            ])
            
            if step_id in completed:
                return
            
            # 渲染输入
            rendered_input = self._render(step.get("input", {}), context)
            
            # 执行工具
            tool_fn = self.tools[step["tool"]]
            result = await tool_fn(**rendered_input)
            
            context["steps"][step_id] = {"result": result}
            results[step_id] = result
            completed.add(step_id)
        
        # 并发执行所有根节点
        await asyncio.gather(*[
            execute_step(sid) for sid in steps
        ])
        
        return results


# ---- 使用示例 ----
engine = WorkflowEngine()

@engine.register("web_search")
async def web_search(query: str):
    # 实际调用搜索 API
    return f"Search results for: {query}"

@engine.register("llm_summarize")
async def llm_summarize(content: str):
    return f"Summary: {content[:100]}..."

@engine.register("send_email")
async def send_email(to: str, body: str):
    print(f"Sending to {to}: {body}")
    return "sent"

async def main():
    with open("workflow.yaml") as f:
        wf = yaml.safe_load(f)
    
    results = await engine.run(wf, params={
        "topic": "AI agents 2025",
        "email": "boss@example.com"
    })
    print(results)

asyncio.run(main())
```

**关键点**：`execute_step` 递归地等待依赖，`asyncio.gather` 自动并行独立步骤。

---

## TypeScript 实现：pi-mono 风格

在 TypeScript 中，可以用 **zod** 做 schema 验证，类型更安全：

```typescript
// workflow-engine.ts
import { z } from "zod"

const StepSchema = z.object({
  id: z.string(),
  tool: z.string(),
  input: z.record(z.unknown()).default({}),
  depends_on: z.array(z.string()).default([]),
  retry: z.number().default(0),
})

const WorkflowSchema = z.object({
  name: z.string(),
  steps: z.array(StepSchema),
})

type Step = z.infer<typeof StepSchema>
type ToolFn = (input: Record<string, unknown>) => Promise<unknown>

export class WorkflowEngine {
  private tools = new Map<string, ToolFn>()

  register(name: string, fn: ToolFn) {
    this.tools.set(name, fn)
    return this
  }

  private renderTemplate(
    template: unknown, 
    ctx: Record<string, unknown>
  ): unknown {
    if (typeof template === "string") {
      return template.replace(/\{\{(.+?)\}\}/g, (_, expr) => {
        const val = expr.trim().split(".").reduce(
          (obj: Record<string, unknown>, key: string) => obj?.[key] as Record<string, unknown>,
          ctx
        )
        return String(val ?? "")
      })
    }
    if (typeof template === "object" && template !== null) {
      return Object.fromEntries(
        Object.entries(template as Record<string, unknown>).map(
          ([k, v]) => [k, this.renderTemplate(v, ctx)]
        )
      )
    }
    return template
  }

  async run(
    workflowDef: unknown, 
    params: Record<string, unknown> = {}
  ): Promise<Record<string, unknown>> {
    const workflow = WorkflowSchema.parse(workflowDef)
    const steps = new Map(workflow.steps.map(s => [s.id, s]))
    const results: Record<string, unknown> = {}
    const completed = new Set<string>()
    const ctx = { params, steps: {} as Record<string, unknown> }

    const executeStep = async (stepId: string): Promise<void> => {
      if (completed.has(stepId)) return
      
      const step = steps.get(stepId)!
      
      // 并行等待所有依赖
      await Promise.all(
        step.depends_on
          .filter(d => !completed.has(d))
          .map(d => executeStep(d))
      )
      
      if (completed.has(stepId)) return // double-check（防止并发重入）

      const rendered = this.renderTemplate(step.input, ctx)
      const tool = this.tools.get(step.tool)
      if (!tool) throw new Error(`Unknown tool: ${step.tool}`)

      // 带重试
      let result: unknown
      for (let attempt = 0; attempt <= step.retry; attempt++) {
        try {
          result = await tool(rendered as Record<string, unknown>)
          break
        } catch (e) {
          if (attempt === step.retry) throw e
          await new Promise(r => setTimeout(r, 1000 * (attempt + 1)))
        }
      }

      ;(ctx.steps as Record<string, unknown>)[stepId] = { result }
      results[stepId] = result
      completed.add(stepId)
    }

    // 触发所有步骤（引擎自动处理依赖顺序）
    await Promise.all(workflow.steps.map(s => executeStep(s.id)))
    return results
  }
}
```

---

## 条件分支：DSL 扩展

真实工作流需要条件逻辑：

```yaml
steps:
  - id: check_cache
    tool: cache_lookup
    input:
      key: "{{ params.query }}"

  - id: fetch_fresh
    tool: web_search
    condition: "{{ steps.check_cache.result == null }}"  # ← 条件步骤
    input:
      query: "{{ params.query }}"

  - id: respond
    tool: llm_generate
    depends_on: [check_cache, fetch_fresh]
    input:
      # 用 ?? 处理可能为 null 的步骤结果
      context: "{{ steps.check_cache.result ?? steps.fetch_fresh.result }}"
```

在引擎里处理 `condition`：

```python
async def execute_step(step_id: str):
    step = steps[step_id]
    
    # 检查条件
    condition = step.get("condition")
    if condition:
        rendered_condition = self._render(condition, context)
        # 简单求值：支持 == null, != null, true/false
        if rendered_condition.strip().lower() in ("false", "null", ""):
            context["steps"][step_id] = {"result": None}
            completed.add(step_id)
            return  # 跳过此步骤
    
    # ... 正常执行
```

---

## OpenClaw 的 Skills 就是一种 Workflow DSL

看 OpenClaw 的技能系统，`SKILL.md` 本质上就是一个**自然语言 DSL**：

```markdown
# SKILL.md
1. 读取配置文件
2. 调用 API 获取数据
3. 如果返回 404，触发告警
4. 将结果写入数据库
```

Agent 读取 SKILL.md 后，把它转换成内部执行计划——这跟 YAML Workflow DSL 的本质是一样的，只是表达形式从结构化 → 非结构化。

**为什么 OpenClaw 用自然语言而不是 YAML？**

| | 自然语言 DSL (SKILL.md) | 结构化 DSL (YAML) |
|---|---|---|
| 人类可读性 | ✅ 极高 | 中等 |
| 机器解析稳定性 | ❌ 依赖 LLM | ✅ 确定性 |
| 表达复杂逻辑 | ✅ 灵活 | ❌ 需要扩展 |
| 版本控制/diff | 友好 | 友好 |
| 适合场景 | 复杂、模糊的任务 | 稳定、重复的流程 |

---

## 实战：为 MysteryBox 写一个监控工作流

```yaml
# monitor-profit.yaml
name: daily_profit_check
steps:
  - id: query_openbox
    tool: grafana_query
    input:
      sql: |
        SELECT SUM(b.price) - SUM(i.price) as profit
        FROM open_box_results obr
        JOIN boxes b ON obr.box_id = b.id
        JOIN items i ON obr.item_id = i.id
        WHERE obr.created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
        AND obr.user_id NOT IN (SELECT user_id FROM bots)

  - id: query_battles
    tool: grafana_query  # 与 query_openbox 并行执行！
    input:
      sql: |
        SELECT SUM(total_box_price * capacity) as total_bet,
               COUNT(*) as room_count
        FROM battle_rooms
        WHERE status = 400
        AND created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)

  - id: generate_report
    tool: llm_generate
    depends_on: [query_openbox, query_battles]
    input:
      prompt: |
        今日数据：
        开箱利润: {{ steps.query_openbox.result.profit }}
        对战总投注: {{ steps.query_battles.result.total_bet }}
        请生成简洁的日报。

  - id: send_alert
    tool: telegram_send
    depends_on: [generate_report]
    condition: "{{ steps.query_openbox.result.profit < 0 }}"
    input:
      chat_id: "-5115329245"
      text: "⚠️ 亏损警告！\n{{ steps.generate_report.result }}"
```

`query_openbox` 和 `query_battles` 没有依赖关系，引擎自动并行查询，省一半时间。

---

## 最佳实践

1. **步骤要幂等** — 工作流可能因错误重跑，每个步骤重复执行要无副作用
2. **结果要可序列化** — 步骤结果存到数据库，支持断点续跑
3. **超时要显式设置** — 每个步骤都加 `timeout`，防止僵尸步骤
4. **版本化工作流定义** — YAML 文件放 Git，用 commit hash 追踪每次执行用的哪个版本
5. **把 DSL 和业务逻辑分开** — 工具实现不关心工作流，工作流不关心工具实现

---

## 总结

| 概念 | 要点 |
|------|------|
| Workflow DSL | 声明"做什么"，引擎负责"怎么做" |
| DAG 执行 | 自动并行没有依赖的步骤 |
| 模板插值 | 上一步结果注入下一步输入 |
| 条件步骤 | 运行时决定是否执行某步骤 |
| 重试机制 | 在步骤级别声明，引擎统一处理 |

下次遇到"Agent 要按顺序做好几件事"的需求，别把逻辑写死在 prompt 里——**写一个 YAML，让引擎跑它**。

---

*Lesson 65 / Agent 开发课程*
