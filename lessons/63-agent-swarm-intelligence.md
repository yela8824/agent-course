# 63 - Agent Swarm Intelligence（蜂群智能）

> 多个专业化小 Agent 组成的群体，如何通过协作涌现出比单一大 Agent 更强的能力

---

## 🐝 什么是蜂群智能？

蜂巢里没有一只蜜蜂"懂得"整个蜂巢的运作，但蜂群整体却能完成极其复杂的任务。Agent Swarm 的思路完全一样：

- **单个 Agent**：能力有限、上下文有限、容易犯错
- **Swarm**：多个专业化 Agent 并发协作，互相校验，涌现出整体智能

这和 Subagents（任务分发）的区别是：

| 模式 | 特征 |
|------|------|
| Subagents | 主 Agent 分配明确子任务，单向控制 |
| Swarm | 多 Agent 平等协作，互相影响，无中心控制器 |

---

## 🏗️ 核心模式：三种蜂群拓扑

### 1. Parallel Swarm（并行蜂群）

多个 Agent 独立执行，最后聚合结果：

```
Query → [Agent A] → ┐
       [Agent B] → ┼→ Aggregator → Final Answer
       [Agent C] → ┘
```

**适用场景**：多视角分析、投票决策、并发搜索

### 2. Competitive Swarm（竞争蜂群）

多个 Agent 互相竞争，最优解胜出：

```
Task → [Agent 1: Solution A]
     → [Agent 2: Solution B]  → Judge → Best Solution
     → [Agent 3: Solution C]
```

**适用场景**：代码生成（多方案比选）、创意任务

### 3. Collaborative Swarm（协作蜂群）

Agent 互相传递、批评、优化彼此的输出：

```
Agent 1 → draft → Agent 2 → critique → Agent 3 → refine → Final
                      ↑__________________________|
```

**适用场景**：长文写作、复杂推理、代码审查

---

## 💻 实现：基于 pi-mono 的 Swarm

```typescript
// pi-mono 风格的 Swarm 实现
// packages/pi/src/swarm.ts

import { Agent, AgentConfig, Message } from "./agent";

interface SwarmConfig {
  agents: AgentConfig[];
  topology: "parallel" | "competitive" | "collaborative";
  aggregationStrategy?: "majority" | "best-of" | "synthesis";
  maxRounds?: number;
}

interface SwarmResult {
  answer: string;
  confidence: number;
  contributions: Array<{ agentId: string; output: string; score?: number }>;
}

export class AgentSwarm {
  private agents: Map<string, Agent> = new Map();
  private config: SwarmConfig;

  constructor(config: SwarmConfig) {
    this.config = config;
    // 初始化所有 Agent
    for (const agentConfig of config.agents) {
      this.agents.set(agentConfig.id, new Agent(agentConfig));
    }
  }

  async run(query: string): Promise<SwarmResult> {
    switch (this.config.topology) {
      case "parallel":
        return this.runParallel(query);
      case "competitive":
        return this.runCompetitive(query);
      case "collaborative":
        return this.runCollaborative(query);
    }
  }

  // 并行执行，然后聚合
  private async runParallel(query: string): Promise<SwarmResult> {
    const agentList = Array.from(this.agents.values());

    // 所有 Agent 并发处理同一问题
    const results = await Promise.all(
      agentList.map(async (agent) => ({
        agentId: agent.id,
        output: await agent.run(query),
      }))
    );

    // 聚合策略
    const answer = await this.aggregate(query, results);

    return {
      answer,
      confidence: this.calculateConsensus(results),
      contributions: results,
    };
  }

  // 竞争模式：Judge Agent 评选最优答案
  private async runCompetitive(query: string): Promise<SwarmResult> {
    const agentList = Array.from(this.agents.values());

    // 所有 Agent 并发生成候选答案
    const candidates = await Promise.all(
      agentList.map(async (agent) => ({
        agentId: agent.id,
        output: await agent.run(query),
      }))
    );

    // Judge Agent 评分（用一个独立 Agent 或 LLM 直接评判）
    const judgePrompt = `
你是一个严格的评判者。以下是针对同一问题的多个答案，请选出最优的：

问题: ${query}

${candidates.map((c, i) => `答案 ${i + 1} (${c.agentId}):\n${c.output}`).join("\n\n---\n\n")}

请输出最优答案的编号（1-${candidates.length}）和理由，格式：
{"winner": <编号>, "reason": "<理由>"}
`;

    const judgeResult = await this.callLLM(judgePrompt);
    const { winner, reason } = JSON.parse(judgeResult);
    const best = candidates[winner - 1];

    return {
      answer: best.output,
      confidence: 0.85,
      contributions: candidates.map((c, i) => ({
        ...c,
        score: i + 1 === winner ? 1.0 : 0.3,
      })),
    };
  }

  // 协作模式：Agent 链式优化
  private async runCollaborative(query: string): Promise<SwarmResult> {
    const agentList = Array.from(this.agents.values());
    const maxRounds = this.config.maxRounds ?? 2;
    
    let currentDraft = "";
    const contributions: SwarmResult["contributions"] = [];

    for (let round = 0; round < maxRounds; round++) {
      for (const agent of agentList) {
        const prompt =
          round === 0 && !currentDraft
            ? query
            : `原始问题: ${query}\n\n当前草稿:\n${currentDraft}\n\n请作为专业评审，改进这个答案：`;

        const output = await agent.run(prompt);
        currentDraft = output;
        contributions.push({ agentId: `${agent.id}:round${round}`, output });
      }
    }

    return {
      answer: currentDraft,
      confidence: 0.9,
      contributions,
    };
  }

  // 共识度计算（基于输出相似度）
  private calculateConsensus(
    results: Array<{ agentId: string; output: string }>
  ): number {
    if (results.length <= 1) return 1.0;
    // 简化版：实际可用向量相似度计算
    const uniqueOutputs = new Set(results.map((r) => r.output.slice(0, 100)));
    return 1 - (uniqueOutputs.size - 1) / results.length;
  }

  private async aggregate(
    query: string,
    results: Array<{ agentId: string; output: string }>
  ): Promise<string> {
    // synthesis 策略：让 LLM 综合多个视角
    const prompt = `
综合以下 ${results.length} 个 Agent 的分析，给出最终答案：

原始问题: ${query}

${results.map((r) => `[${r.agentId}]: ${r.output}`).join("\n\n")}

请综合所有视角，给出一个全面、准确的回答：
`;
    return this.callLLM(prompt);
  }

  private async callLLM(prompt: string): Promise<string> {
    // 调用底层 LLM（pi-mono 的 model 层）
    const { generateText } = await import("ai");
    const { result } = await generateText({
      model: this.getModel(),
      prompt,
    });
    return result.text;
  }
}
```

---

## 🎯 OpenClaw 中的 Swarm 实践

OpenClaw 的 `sessions_spawn` + `sessions_send` 天然支持 Swarm 模式：

```typescript
// OpenClaw Swarm：3个专家 Agent 并发分析同一问题
async function swarmAnalysis(question: string) {
  // 启动3个专家 Agent（不同 system prompt）
  const [techAgent, businessAgent, riskAgent] = await Promise.all([
    sessions_spawn({
      task: `你是技术专家。分析：${question}`,
      mode: "run",
      runtime: "subagent",
    }),
    sessions_spawn({
      task: `你是商业分析师。分析：${question}`,
      mode: "run",
      runtime: "subagent",
    }),
    sessions_spawn({
      task: `你是风险评估专家。分析：${question}`,
      mode: "run",
      runtime: "subagent",
    }),
  ]);

  // 等待所有完成，汇总到主 Agent 综合判断
  const results = await Promise.all([
    waitForResult(techAgent),
    waitForResult(businessAgent),
    waitForResult(riskAgent),
  ]);

  return synthesize(results);
}
```

---

## 🧠 learn-claude-code 中的实现

Python 版本用 `asyncio` 实现并发 Swarm：

```python
# learn-claude-code/swarm_agent.py
import asyncio
from anthropic import AsyncAnthropic
from typing import List, Dict, Any

client = AsyncAnthropic()

EXPERT_PERSONAS = {
    "researcher": "你是研究专家，擅长收集和分析信息，注重准确性和引用来源。",
    "critic": "你是批判性思维专家，擅长发现漏洞和反驳论点，不留情面。",
    "synthesizer": "你是综合专家，擅长整合多方观点，找到共识与最优解。",
}

async def run_expert(
    persona: str, system_prompt: str, query: str, context: str = ""
) -> Dict[str, Any]:
    """单个专家 Agent 运行"""
    messages = [{"role": "user", "content": query}]
    if context:
        messages.insert(
            0,
            {
                "role": "user",
                "content": f"背景信息（其他专家的观点）：\n{context}\n\n请在此基础上回答：",
            },
        )

    response = await client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=system_prompt,
        messages=messages,
    )
    return {"persona": persona, "output": response.content[0].text}


async def parallel_swarm(query: str) -> List[Dict]:
    """并行蜂群：所有专家同时分析"""
    tasks = [
        run_expert(persona, system, query)
        for persona, system in EXPERT_PERSONAS.items()
    ]
    results = await asyncio.gather(*tasks)
    return list(results)


async def collaborative_swarm(query: str, rounds: int = 2) -> str:
    """协作蜂群：专家轮流优化答案"""
    experts = list(EXPERT_PERSONAS.items())
    current_draft = ""

    for round_num in range(rounds):
        for persona, system in experts:
            context = current_draft if current_draft else ""
            result = await run_expert(persona, system, query, context)
            current_draft = result["output"]
            print(f"Round {round_num+1}, {persona}: {current_draft[:100]}...")

    return current_draft


async def swarm_with_voting(query: str) -> str:
    """投票蜂群：多数决定最终答案"""
    results = await parallel_swarm(query)

    # 让 Synthesizer 进行最终投票综合
    all_outputs = "\n\n".join(
        [f"[{r['persona']}]: {r['output']}" for r in results]
    )
    synthesis_prompt = f"""
以下是多个专家对同一问题的分析：
{all_outputs}

请综合以上观点，给出最终答案。明确指出哪些观点达成共识，哪些存在分歧。
"""
    final = await run_expert(
        "synthesizer",
        "你是综合协调员，负责整合多方专家意见。",
        synthesis_prompt,
    )
    return final["output"]


# 使用示例
async def main():
    query = "AI Agent 在 2025 年最大的安全风险是什么？"

    print("=== 并行蜂群 ===")
    results = await parallel_swarm(query)
    for r in results:
        print(f"\n[{r['persona']}]:\n{r['output']}\n")

    print("\n=== 投票综合 ===")
    final = await swarm_with_voting(query)
    print(final)


asyncio.run(main())
```

---

## ⚡ 关键设计决策

### 1. 什么时候用 Swarm？

✅ **适合 Swarm**：
- 需要多视角（技术/商业/风险）
- 单 Agent 答案不可靠（需要验证）
- 创意任务（多样性优于单一最优）
- 高风险决策（需要交叉验证）

❌ **不适合 Swarm**：
- 简单事实查询（浪费 token）
- 需要状态连续性的长任务
- 延迟敏感场景（Swarm 天生慢）

### 2. 专业化 vs 通用化

```typescript
// ❌ 错误：所有 Agent 用相同 prompt，白白浪费
const agents = Array(5).fill(new Agent({ system: "你是一个助手" }));

// ✅ 正确：每个 Agent 有明确专业方向
const agents = [
  new Agent({ system: "你是代码安全专家，专注发现安全漏洞" }),
  new Agent({ system: "你是性能优化专家，专注发现性能瓶颈" }),
  new Agent({ system: "你是架构师，专注评估系统设计" }),
];
```

### 3. 避免 Swarm 失控

```typescript
// 关键：设置超时和最大轮数，防止无限协作循环
const swarm = new AgentSwarm({
  agents: expertConfigs,
  topology: "collaborative",
  maxRounds: 3,          // ← 硬性限制轮数
  timeoutMs: 30_000,     // ← 总超时
});
```

---

## 📊 Swarm vs 单一 Agent 性能对比

| 指标 | 单 Agent | Parallel Swarm | Collaborative Swarm |
|------|---------|----------------|---------------------|
| 准确率 | 基准 | +15~25% | +20~35% |
| 延迟 | 1x | ~1x（并发） | 3~5x |
| Token 成本 | 1x | Nx（N个Agent） | N*rounds |
| 适合场景 | 简单任务 | 分析/验证 | 复杂推理/创作 |

---

## 🔑 核心要点

1. **Swarm ≠ 更多 Agent**：关键是**专业化分工**，不是堆数量
2. **并行优先**：能并行就不要串行，延迟是 Swarm 最大痛点
3. **聚合是灵魂**：Aggregator 的质量决定 Swarm 输出质量
4. **设置上限**：maxRounds、timeoutMs、maxAgents 缺一不可
5. **OpenClaw 天然适合**：`sessions_spawn` 的并发 subagent 就是 Swarm 的基础设施

---

## 🚀 课后实践

用 OpenClaw 实现一个 3-Agent Swarm 来分析一段代码：
1. **安全 Agent**：找安全漏洞
2. **性能 Agent**：找性能问题  
3. **可读性 Agent**：评估代码质量

三个 Agent 并发跑，最后一个 Synthesizer Agent 综合输出报告。

> 下一课预告：**Agent Knowledge Graph（知识图谱）** — 让 Agent 在结构化知识网络中推理，超越简单的向量检索
