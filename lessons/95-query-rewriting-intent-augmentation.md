# 95 - Agent 查询改写与意图增强（Query Rewriting & Intent Augmentation）

> 用户说的不等于用户想的。在把输入交给 LLM/工具之前，先把它"翻译"成最有利于执行的形式。

---

## 为什么需要查询改写？

用户输入往往是**模糊、不完整、依赖上下文**的：

```
用户: "上次那个报表怎么了？"
用户: "帮我查一下"
用户: "cdk的那个错咋修"
```

如果把这些原话直接丢给 RAG 检索、工具调用或 LLM，结果会很差。

**查询改写（Query Rewriting）**解决的是：**在执行前，把模糊意图转化为精确可执行的操作描述**。

---

## 三种核心场景

### 1. 代词解析 —— 消除上下文依赖

```python
# 原始查询（依赖对话历史）
user_turn_1 = "帮我看看 mysterybox 的今日利润"
user_turn_2 = "那昨天呢？"  # "那" 指什么？

# 改写后
rewritten = "查询 mysterybox 平台昨日（2026-03-19）开箱总利润"
```

### 2. 意图扩展 —— 从简短到详尽

```python
# 原始: "部署失败"
# 扩展后的搜索查询:
expanded_queries = [
    "deployment failed error logs",
    "laravel cloud deployment failure",
    "app-9e7e19f2 deploy error 2026-03-20",
]
```

### 3. HyDE（假设文档嵌入）—— 为 RAG 生成更优检索向量

```python
# 原始查询: "怎么配置 Redis 缓存驱动？"
# HyDE: 先让 LLM 生成一段"假设的答案文档"，用它做向量检索

hypothetical_answer = """
在 Laravel 中配置 Redis 缓存，需要在 config/cache.php 中设置
default driver 为 'redis'，并在 .env 中配置 REDIS_HOST...
"""
# 用 hypothetical_answer 的向量去检索，而非原始问题的向量
# → 检索到的文档更相关
```

---

## 实战实现

### Python 实现（learn-claude-code 风格）

```python
from anthropic import Anthropic
from dataclasses import dataclass
from typing import Optional
import json

client = Anthropic()

@dataclass
class ConversationTurn:
    role: str
    content: str

@dataclass  
class RewrittenQuery:
    original: str
    rewritten: str           # 主查询（去除代词、完整表达）
    search_queries: list[str]  # 多路搜索展开
    entities: dict           # 提取出的关键实体
    time_range: Optional[str]  # 时间范围（如有）
    intent: str              # 分类后的意图


def rewrite_query(
    user_input: str,
    conversation_history: list[ConversationTurn],
    available_tools: list[str]
) -> RewrittenQuery:
    """
    在调用工具或 RAG 之前，先改写/增强用户查询
    """
    
    # 构造上下文感知的改写提示
    history_text = "\n".join(
        f"{t.role}: {t.content}" 
        for t in conversation_history[-5:]  # 只看最近5轮
    )
    
    tools_text = ", ".join(available_tools)
    
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=512,
        system="""你是一个查询改写引擎。
        
任务：分析用户最新输入，结合对话历史，输出结构化的改写结果。

规则：
1. 消除所有代词引用（"它/那个/上次" → 具体指代）
2. 补全省略的时间/主体（"帮我查" → "查询[具体平台][具体指标]"）
3. 展开为2-3个搜索变体，覆盖不同表达方式
4. 提取关键实体（platform/metric/time/user_id等）
5. 判断意图类型: query/action/debug/report

输出 JSON，不要任何额外文字。""",
        messages=[
            {
                "role": "user",
                "content": f"""对话历史：
{history_text}

可用工具：{tools_text}
用户最新输入：{user_input}

请改写并结构化这个查询。"""
            }
        ]
    )
    
    data = json.loads(response.content[0].text)
    
    return RewrittenQuery(
        original=user_input,
        rewritten=data["rewritten"],
        search_queries=data.get("search_queries", [user_input]),
        entities=data.get("entities", {}),
        time_range=data.get("time_range"),
        intent=data.get("intent", "query")
    )


# ============ 使用示例 ============

history = [
    ConversationTurn("user", "帮我看看 mysterybox 今日开箱利润"),
    ConversationTurn("assistant", "今日（2026-03-20）开箱利润为 ¥12,450，共计 3,201 次开箱"),
]

result = rewrite_query(
    user_input="那昨天呢？跟今天比差多少",
    conversation_history=history,
    available_tools=["grafana_query", "database_sql", "web_search"]
)

print(f"原始: {result.original}")
# → "那昨天呢？跟今天比差多少"

print(f"改写: {result.rewritten}")
# → "查询 mysterybox 平台昨日（2026-03-19）开箱总利润，并与今日（2026-03-20）¥12,450 对比差值"

print(f"搜索变体: {result.search_queries}")
# → ["mysterybox 昨日开箱利润 2026-03-19",
#    "open_box_results profit yesterday comparison",
#    "mysterybox daily profit trend 2026-03-19 vs 2026-03-20"]

print(f"提取实体: {result.entities}")
# → {"platform": "mysterybox", "metric": "开箱利润", 
#    "date_from": "2026-03-19", "date_to": "2026-03-20"}

print(f"意图: {result.intent}")
# → "query"
```

---

### TypeScript 实现（pi-mono 风格）

```typescript
// src/pipeline/query-rewriter.ts
import Anthropic from "@anthropic-ai/sdk";

interface RewriteResult {
  original: string;
  rewritten: string;
  searchQueries: string[];
  entities: Record<string, string>;
  intent: "query" | "action" | "debug" | "report";
  confidence: number;
}

// 轻量级规则预处理（不调用 LLM）
function preprocess(input: string): string {
  return input
    .trim()
    .replace(/\s+/g, " ")
    // 扩展常见缩写
    .replace(/\bMB\b/g, "mysterybox")
    .replace(/\bcdk\b/gi, "CDK Shop");
}

// 主改写函数
async function rewriteQuery(
  input: string,
  history: Array<{ role: string; content: string }>,
  client: Anthropic
): Promise<RewriteResult> {
  const preprocessed = preprocess(input);

  // 快速路径：输入已经足够清晰，跳过 LLM 改写
  const isAlreadyClear =
    preprocessed.length > 30 && // 足够长
    !/\b(它|那个|上次|这个|这里|那里)\b/.test(preprocessed) && // 无代词
    !/^(帮我|看看|查一下)$/.test(preprocessed.trim()); // 非纯动词

  if (isAlreadyClear && history.length === 0) {
    return {
      original: input,
      rewritten: preprocessed,
      searchQueries: [preprocessed],
      entities: {},
      intent: "query",
      confidence: 0.9,
    };
  }

  const recentHistory = history
    .slice(-4)
    .map((t) => `${t.role}: ${t.content}`)
    .join("\n");

  const response = await client.messages.create({
    model: "claude-3-5-haiku-20241022",
    max_tokens: 400,
    messages: [
      {
        role: "user",
        content: `历史:\n${recentHistory}\n\n当前输入: ${preprocessed}\n\n改写为独立完整的查询，输出JSON: {rewritten, searchQueries, entities, intent, confidence}`,
      },
    ],
    system:
      "查询改写引擎。消除代词，补全主体，提取实体。只输出JSON。",
  });

  const text =
    response.content[0].type === "text" ? response.content[0].text : "{}";

  try {
    const data = JSON.parse(text);
    return {
      original: input,
      rewritten: data.rewritten ?? preprocessed,
      searchQueries: data.searchQueries ?? [preprocessed],
      entities: data.entities ?? {},
      intent: data.intent ?? "query",
      confidence: data.confidence ?? 0.7,
    };
  } catch {
    // 解析失败时回退到预处理结果
    return {
      original: input,
      rewritten: preprocessed,
      searchQueries: [preprocessed],
      entities: {},
      intent: "query",
      confidence: 0.5,
    };
  }
}

// ============ 接入 Agent Pipeline ============

async function agentTurnWithRewrite(
  userInput: string,
  conversationHistory: Array<{ role: string; content: string }>,
  tools: Anthropic.Tool[],
  client: Anthropic
) {
  // Step 1: 改写查询
  const rewritten = await rewriteQuery(userInput, conversationHistory, client);

  console.log(`[Rewriter] ${rewritten.original} → ${rewritten.rewritten}`);
  console.log(`[Rewriter] 意图: ${rewritten.intent}, 置信度: ${rewritten.confidence}`);

  // Step 2: 用改写后的内容注入系统提示
  const contextNote =
    rewritten.confidence < 0.7
      ? `[注意: 用户输入可能有歧义，已改写为: ${rewritten.rewritten}]`
      : "";

  // Step 3: 用改写后的查询执行 Agent Turn
  const messages: Anthropic.MessageParam[] = [
    ...conversationHistory,
    {
      role: "user",
      // 关键：向 LLM 透传改写后的版本，帮助它更准确地选工具
      content: rewritten.rewritten + (contextNote ? `\n\n${contextNote}` : ""),
    },
  ];

  return client.messages.create({
    model: "claude-opus-4-5",
    max_tokens: 4096,
    tools,
    messages,
  });
}
```

---

## OpenClaw 中的查询改写实践

OpenClaw 在 Skills 机制中内置了查询增强思路。以 `mysterybox` 技能为例：

```markdown
<!-- skills/mysterybox/SKILL.md 中的关键设计 -->
## 意图识别规则

当用户说：
- "利润怎么样" → 标准化为：查询过去7天开箱净利润（SUM(price) - SUM(cost)）
- "有没有卡住的" → 标准化为：查询 status='initial' AND created_at < NOW()-15min 的提币订单
- "对战情况" → 标准化为：查询过去24h completed battle_rooms 的 bet/payout 汇总
```

这就是技能层面的**静态查询改写规则**——不需要 LLM，直接靠规则把模糊表达映射成精确 SQL 意图。

---

## HyDE 实现（RAG 场景专用）

```python
def hyde_retrieve(user_question: str, vector_store, client: Anthropic) -> list[str]:
    """
    Hypothetical Document Embeddings:
    先生成假设答案，再用假设答案检索，而非用问题检索
    """
    
    # Step 1: 生成假设答案
    hypothesis = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": f"请写一段简短的文档，内容是关于这个问题的答案（即使你不确定）：{user_question}"
        }]
    ).content[0].text
    
    # Step 2: 用假设答案的 embedding 去检索（而非用问题的 embedding）
    hypothesis_embedding = embed(hypothesis)  # 你的 embedding 函数
    
    # Step 3: 检索相似文档
    results = vector_store.search(hypothesis_embedding, top_k=5)
    
    return results

# 为什么有效？
# 问题向量 和 答案向量 的语义空间不同。
# "怎么配置Redis?" 的向量 ≠ "Redis配置方法: 在config/cache.php中..." 的向量
# 后者能匹配到文档库中真实的配置文档。
```

---

## 多路查询展开（Multi-Query Retrieval）

```python
def multi_query_retrieve(question: str, vector_store, client: Anthropic) -> list[str]:
    """
    生成多个查询变体，取并集，提高 RAG 召回率
    """
    
    # 生成5个改写变体
    variants_response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""为以下问题生成5个语义等价但表达不同的搜索查询，每行一个：
            
{question}"""
        }]
    )
    
    variants = variants_response.content[0].text.strip().split("\n")
    variants.append(question)  # 包含原始查询
    
    # 对每个变体检索，取并集去重
    all_results = set()
    for variant in variants:
        results = vector_store.search(embed(variant), top_k=3)
        all_results.update(results)
    
    # 按相关性重排
    return rerank(question, list(all_results))[:5]
```

---

## 关键设计原则

### 1. 快速路径（Fast Path）
不要无脑把每个查询都走 LLM 改写——成本和延迟不值得。先用规则判断是否需要改写：

```python
def needs_rewrite(query: str, has_history: bool) -> bool:
    PRONOUNS = ["它", "那个", "上次", "这个", "他", "她", "那里", "这里"]
    SHORT_COMMANDS = ["帮我查", "看看", "怎么了", "呢？"]
    
    has_pronoun = any(p in query for p in PRONOUNS)
    is_too_short = len(query) < 10
    has_short_cmd = any(query.strip().startswith(c) for c in SHORT_COMMANDS)
    
    return (has_pronoun and has_history) or is_too_short or has_short_cmd
```

### 2. 向 LLM 透传改写结果
改写后的查询**替换**原始输入传给 LLM，让 LLM 做工具调用决策时获益。

### 3. 记录改写日志
改写结果是调试利器——当 Agent 答非所问时，先看改写后的查询是否正确。

---

## 总结

| 技术 | 适用场景 | 成本 |
|------|----------|------|
| 规则预处理 | 缩写展开、格式标准化 | 零 |
| 代词解析 | 多轮对话上下文依赖 | 低（小模型） |
| 意图扩展 | 短命令 → 完整查询 | 低 |
| HyDE | RAG 检索质量差时 | 中（额外 LLM 调用） |
| 多路展开 | 召回率不足时 | 中高（多次检索） |

**核心思想：** 用一次廉价的小模型调用，换来主流程更高质量的执行。把"翻译用户意图"这个工作从主模型剥离出来，做成专用的预处理环节。

---

*下一课预告：Agent 渐进式工具暴露（Progressive Tool Disclosure）—— 从最小工具集开始，按需动态扩展可用工具范围，降低 LLM 的选择负担。*
