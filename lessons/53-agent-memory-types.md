# 第 53 课：Agent 记忆类型 — Episodic / Semantic / Procedural

> 人类有三种记忆，Agent 也该有。别把所有东西都塞进一个 JSON 文件了。

---

## 为什么要区分记忆类型？

很多开发者做 Agent 记忆的方式是：把所有东西塞进 `MEMORY.md`，或者全往向量库里扔。

这就像你把"昨天吃了啥"、"巴黎的首都是哪"、"怎么骑自行车"全写在同一张便利贴上。

认知科学早就告诉我们：记忆有类型，不同类型用不同机制存储和检索。Agent 也一样。

## 三种记忆类型

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent Memory Types                        │
├────────────────┬──────────────────┬──────────────────────────┤
│  Episodic      │  Semantic        │  Procedural              │
│  情景记忆       │  语义记忆         │  程序记忆                 │
├────────────────┼──────────────────┼──────────────────────────┤
│ "发生了什么"    │ "世界是什么样的"  │ "怎么做事"                │
│ 具体事件、对话  │ 知识、概念、事实  │ 技能、工具调用模式         │
│ 有时间戳       │ 无时间维度        │ 隐性知识                  │
├────────────────┼──────────────────┼──────────────────────────┤
│ 向量DB +       │ 结构化KV +       │ Few-shot examples +       │
│ 时间索引       │ 知识图谱         │ Tool usage patterns       │
└────────────────┴──────────────────┴──────────────────────────┘
```

---

## 1. Episodic Memory（情景记忆）

### 是什么

记住具体发生的事件：

- "2024-01-15 用户问过比特币价格"
- "上次部署失败是因为环境变量没设"
- "老板不喜欢超过 3 段的回答"

### 实现方式

**核心：带时间戳的向量存储 + 时间衰减**

```typescript
// pi-mono 风格实现
interface EpisodicMemory {
  id: string;
  timestamp: number;
  content: string;
  embedding: number[];
  metadata: {
    userId?: string;
    sessionId?: string;
    importance: number; // 0-1
  };
}

class EpisodicMemoryStore {
  private db: VectorDB;
  
  async store(event: string, metadata?: Record<string, any>) {
    const embedding = await this.embed(event);
    const memory: EpisodicMemory = {
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      content: event,
      embedding,
      metadata: {
        importance: await this.scoreImportance(event),
        ...metadata,
      },
    };
    await this.db.upsert(memory);
  }

  async recall(query: string, opts?: { 
    maxAge?: number;      // 只查最近 N 毫秒
    limit?: number;
    recencyWeight?: number; // 0=只看相似度, 1=只看时间
  }) {
    const { maxAge = Infinity, limit = 5, recencyWeight = 0.3 } = opts ?? {};
    
    const queryEmbed = await this.embed(query);
    const cutoff = Date.now() - maxAge;
    
    // 混合分数 = 相似度 * (1-recencyWeight) + 时间分 * recencyWeight
    const results = await this.db.search(queryEmbed, {
      filter: { timestamp: { $gte: cutoff } },
      limit: limit * 3, // 多取一些再重排
    });

    return results
      .map(r => ({
        ...r,
        score: r.similarity * (1 - recencyWeight) + 
               this.recencyScore(r.timestamp) * recencyWeight,
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }

  private recencyScore(timestamp: number): number {
    // 1小时内=1.0, 1天=0.5, 1周=0.1
    const ageHours = (Date.now() - timestamp) / 3_600_000;
    return Math.exp(-ageHours * 0.1);
  }

  private async scoreImportance(event: string): Promise<number> {
    // 用小模型快速打分，避免用大模型浪费钱
    const response = await callLLM({
      model: "claude-haiku-3",
      prompt: `Rate the importance of this event for future reference (0.0-1.0, one decimal):
Event: ${event}
Score:`,
      maxTokens: 5,
    });
    return parseFloat(response) || 0.5;
  }
}
```

**OpenClaw 的实际做法**：

OpenClaw 用 `memory/YYYY-MM-DD.md` 做情景记忆，每天一个文件，自然形成时间序列。`memory_search` 工具做语义检索，时间维度靠文件名过滤。简单粗暴但够用。

```bash
# OpenClaw memory 目录结构
memory/
  2026-03-14.md  # 昨天发生的事
  2026-03-15.md  # 今天的事（实时追加）
  heartbeat-state.json  # 心跳检查状态
```

---

## 2. Semantic Memory（语义记忆）

### 是什么

关于世界的通用知识，不绑定具体时间：

- "用户偏好：回答要简洁，不超过3段"
- "公司产品：MysteryBox 是 CS:GO 皮肤盲盒平台"
- "技术栈：后端 Laravel，前端 Vue3，部署 AWS"

### 实现方式

**核心：结构化 KV + 概念图谱**

```typescript
// 分层语义记忆
interface SemanticMemory {
  concept: string;      // 概念名
  category: string;     // 分类（user_pref / domain_knowledge / system_info）
  value: string;        // 内容
  confidence: number;   // 置信度（可能会被更新/推翻）
  lastUpdated: number;
  source?: string;      // 从哪学来的
}

class SemanticMemoryStore {
  private store = new Map<string, SemanticMemory>();

  // 存储知识
  learn(concept: string, value: string, opts?: Partial<SemanticMemory>) {
    const existing = this.store.get(concept);
    
    if (existing && existing.confidence > (opts?.confidence ?? 0.8)) {
      // 高置信度的知识不轻易覆盖，但要记录冲突
      console.log(`[Memory] Conflict for "${concept}": 
        old="${existing.value}", new="${value}"`);
    }
    
    this.store.set(concept, {
      concept,
      value,
      category: opts?.category ?? 'general',
      confidence: opts?.confidence ?? 0.8,
      lastUpdated: Date.now(),
      source: opts?.source,
    });
  }

  // 按分类批量查询
  recall(category?: string): SemanticMemory[] {
    const all = Array.from(this.store.values());
    return category ? all.filter(m => m.category === category) : all;
  }

  // 构建 context 注入到 system prompt
  toContextString(category?: string): string {
    const memories = this.recall(category);
    if (memories.length === 0) return '';
    
    return `## Known Facts\n` + 
      memories.map(m => `- ${m.concept}: ${m.value}`).join('\n');
  }
}

// 使用示例
const semantic = new SemanticMemoryStore();
semantic.learn('user_name', '老板', { category: 'user_pref', confidence: 1.0 });
semantic.learn('user_language', '中文', { category: 'user_pref', confidence: 1.0 });
semantic.learn('product_name', 'MysteryBox', { category: 'domain' });

// 注入 system prompt
const context = semantic.toContextString('user_pref');
// => "## Known Facts\n- user_name: 老板\n- user_language: 中文"
```

**OpenClaw 的实际做法**：

`MEMORY.md` 和 `USER.md` 就是语义记忆的具体实现——结构化的、持久化的、关于用户和世界的知识库。每次 session 开始时注入到 context。

```markdown
# MEMORY.md (语义记忆片段)
## 用户偏好
- 不喜欢废话，直接给结论
- 技术问题不需要解释基础概念
- 喜欢代码示例多于文字描述
```

---

## 3. Procedural Memory（程序记忆）

### 是什么

"怎么做事"的隐性知识：

- "查询 Grafana 数据的 SQL 模板"
- "部署新功能的步骤：PR → CI → staging → prod"
- "遇到 rate limit 就用指数退避，不要直接重试"

### 实现方式

**核心：Few-shot examples + Tool usage patterns**

```typescript
// 程序记忆 = 可复用的"做法模板"
interface ProceduralMemory {
  skill: string;          // 技能名
  trigger: string;        // 什么情况下触发
  steps: string[];        // 步骤
  examples: Array<{       // 成功案例（few-shot）
    input: string;
    output: string;
    toolCalls?: any[];
  }>;
  successRate: number;    // 历史成功率（用于自动更新）
}

class ProceduralMemoryStore {
  private skills = new Map<string, ProceduralMemory>();

  registerSkill(skill: ProceduralMemory) {
    this.skills.set(skill.skill, skill);
  }

  // 找最相关的技能（用于 few-shot injection）
  async findRelevant(task: string): Promise<ProceduralMemory[]> {
    // 简单版：关键词匹配
    // 生产版：embedding 相似度
    return Array.from(this.skills.values())
      .filter(s => task.toLowerCase().includes(s.trigger.toLowerCase()))
      .sort((a, b) => b.successRate - a.successRate)
      .slice(0, 3);
  }

  // 更新成功率（强化学习 lite）
  recordOutcome(skill: string, success: boolean) {
    const s = this.skills.get(skill);
    if (!s) return;
    // 指数移动平均
    s.successRate = s.successRate * 0.9 + (success ? 1 : 0) * 0.1;
  }

  // 生成 few-shot prompt 片段
  toFewShotPrompt(skills: ProceduralMemory[]): string {
    return skills.flatMap(s => 
      s.examples.map(ex => 
        `Task: ${ex.input}\nSolution: ${ex.output}`
      )
    ).join('\n\n');
  }
}

// OpenClaw Skills 就是程序记忆的一种实现
// SKILL.md 告诉 Agent "怎么用这个工具"
```

**OpenClaw 的实际做法**：

Skills 系统（`/opt/homebrew/lib/node_modules/openclaw/skills/*/SKILL.md`）就是程序记忆：

```
skills/
  github/SKILL.md      # "怎么用 gh CLI 操作 GitHub"
  weather/SKILL.md     # "怎么查天气"  
  gog/SKILL.md         # "怎么操作 Google Workspace"
```

每个 SKILL.md 是一个结构化的"做法模板"，包含：触发条件、步骤、注意事项、示例。Agent 在需要时加载，而不是一直塞在 context 里（节省 tokens）。

---

## 三种记忆的协同

```typescript
class AgentMemorySystem {
  episodic: EpisodicMemoryStore;
  semantic: SemanticMemoryStore;
  procedural: ProceduralMemoryStore;

  // 构建完整的 context
  async buildContext(userMessage: string): Promise<string> {
    const parts: string[] = [];

    // 1. 语义记忆：用户偏好和领域知识（始终注入）
    const semanticCtx = this.semantic.toContextString();
    if (semanticCtx) parts.push(semanticCtx);

    // 2. 情景记忆：相关历史事件（按需检索）
    const recentEvents = await this.episodic.recall(userMessage, {
      maxAge: 7 * 24 * 3600_000, // 最近7天
      limit: 3,
      recencyWeight: 0.4,
    });
    if (recentEvents.length > 0) {
      parts.push('## Relevant History\n' + 
        recentEvents.map(e => `- ${e.content}`).join('\n'));
    }

    // 3. 程序记忆：相关技能模板（任务匹配时注入）
    const skills = await this.procedural.findRelevant(userMessage);
    if (skills.length > 0) {
      parts.push('## How To\n' + this.procedural.toFewShotPrompt(skills));
    }

    return parts.join('\n\n');
  }

  // 会话结束后，自动归档情景记忆
  async consolidate(session: Session) {
    const summary = await summarizeSession(session);
    
    // 新事件 → 情景记忆
    await this.episodic.store(summary.events.join('; '));
    
    // 学到的知识 → 语义记忆
    for (const fact of summary.facts) {
      this.semantic.learn(fact.concept, fact.value);
    }
    
    // 成功/失败的模式 → 更新程序记忆成功率
    for (const outcome of summary.skillOutcomes) {
      this.procedural.recordOutcome(outcome.skill, outcome.success);
    }
  }
}
```

---

## 选哪种？决策树

```
你想记住什么？
│
├── 具体事件（"用户上次说了XXX"）
│   → Episodic Memory（向量DB + 时间索引）
│
├── 通用知识（"用户喜欢简洁"）  
│   → Semantic Memory（结构化KV）
│
└── 操作步骤（"怎么部署/查询/发消息"）
    → Procedural Memory（Skills / Few-shot examples）
```

## 常见错误

| 错误做法 | 问题 | 正确做法 |
|---------|------|---------|
| 把所有记忆塞进一个文件 | 检索慢，噪声多 | 按类型分开存储 |
| 每次都全量注入 | Token 浪费 | 按需检索，只注入相关的 |
| 情景记忆不加时间权重 | 旧事件干扰当前判断 | 加时间衰减 |
| 程序记忆写死不更新 | 无法从失败中学习 | 记录成功率，动态调整 |

---

## 实战作业

1. 看 OpenClaw 的 `memory/` 目录结构，识别哪些是情景记忆，哪些是语义记忆
2. 在你的 Agent 里加一个 `SemanticMemoryStore`，把用户偏好从 system prompt 里提取出来单独管理
3. 把你最常用的工具调用模式写成 `ProceduralMemory`，用 few-shot 方式注入，看看准确率有没有提升

---

*下节课预告：Agent 冷启动优化 — 新用户第一次使用时如何快速建立上下文，减少"你是谁、你会什么"的冗余交互*
