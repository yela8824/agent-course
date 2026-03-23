# 第121课：Agent 置信度感知路由与多候选评估

> Confidence-Aware Routing & Multi-Candidate Evaluation

---

## 问题场景

用户说："帮我处理一下那个文件"

LLM 不知道：
- 哪个文件？
- 什么叫"处理"？读取？压缩？上传？删除？

传统做法：**直接问用户**（体验差）或**随便猜一个执行**（风险高）。

**更好的做法**：给每个可能的意图打置信度分，用分数决定"执行/候选展示/主动澄清"的路由策略。

---

## 核心思路

```
用户输入
  → 意图识别 → 多候选 + 置信度分
  → 置信度路由
      HIGH (>0.85)  → 直接执行
      MID  (0.5-0.85) → 展示候选让用户确认
      LOW  (<0.5)   → 主动澄清
```

---

## 代码实现（TypeScript / pi-mono 风格）

### 1. 候选意图类型定义

```typescript
interface IntentCandidate {
  intent: string;          // 意图名称
  toolName: string;        // 对应工具
  params: Record<string, unknown>; // 预填参数
  confidence: number;      // 0~1 置信度
  reason: string;          // 判断依据（给 LLM 看，也供调试）
}

interface ConfidenceRoutingResult {
  strategy: 'execute' | 'confirm' | 'clarify';
  chosen?: IntentCandidate;    // execute 时有值
  candidates?: IntentCandidate[]; // confirm 时有值
  clarifyQuestion?: string;    // clarify 时有值
}
```

### 2. 置信度评分器

```typescript
// 用 LLM 本身来打分——结构化输出
async function scoreIntents(
  userMessage: string,
  context: string,
  possibleIntents: string[]
): Promise<IntentCandidate[]> {
  const prompt = `
你是意图分析器。分析用户消息，对每个可能的意图打置信度分。

用户消息: "${userMessage}"
上下文: ${context}
候选意图: ${possibleIntents.join(', ')}

返回 JSON 数组，按置信度降序：
[
  {
    "intent": "意图名",
    "toolName": "对应工具",
    "params": {},
    "confidence": 0.85,
    "reason": "为什么这个意图可能性高"
  }
]
只返回 JSON，不要解释。`;

  const response = await callLLM(prompt, { responseFormat: 'json' });
  return JSON.parse(response) as IntentCandidate[];
}
```

### 3. 置信度路由器

```typescript
const THRESHOLDS = {
  EXECUTE:  0.85,  // 高置信度直接执行
  CONFIRM:  0.50,  // 中置信度让用户确认
  // 低于 0.50 → 澄清
};

function confidenceRouter(
  candidates: IntentCandidate[]
): ConfidenceRoutingResult {
  if (candidates.length === 0) {
    return {
      strategy: 'clarify',
      clarifyQuestion: '你想让我做什么？能描述得更具体些吗？'
    };
  }

  const top = candidates[0];

  // 高置信度 → 直接执行
  if (top.confidence >= THRESHOLDS.EXECUTE) {
    return { strategy: 'execute', chosen: top };
  }

  // 中置信度 → 展示候选
  if (top.confidence >= THRESHOLDS.CONFIRM) {
    // 取置信度前3的候选
    const topCandidates = candidates
      .filter(c => c.confidence >= 0.3)
      .slice(0, 3);
    return { strategy: 'confirm', candidates: topCandidates };
  }

  // 低置信度 → 澄清
  // 用 LLM 生成有针对性的澄清问题（而非通用的"你想干啥"）
  const clarifyQuestion = generateClarifyQuestion(candidates);
  return { strategy: 'clarify', clarifyQuestion };
}

function generateClarifyQuestion(candidates: IntentCandidate[]): string {
  // 找出候选之间最大的分歧点
  const topTwo = candidates.slice(0, 2);
  if (topTwo.length >= 2) {
    return `我不太确定你的意思——你是想 ${topTwo[0].intent}，还是 ${topTwo[1].intent}？`;
  }
  return '你的请求有点模糊，能说得更具体些吗？';
}
```

### 4. 整合进 Agent Loop

```typescript
// learn-claude-code / OpenClaw 风格的 tool dispatch 扩展
async function handleAmbiguousToolCall(
  userMessage: string,
  sessionContext: string
): Promise<void> {
  const possibleIntents = [
    'read_file', 'delete_file', 'compress_file',
    'upload_file', 'rename_file', 'summarize_file'
  ];

  // 1. 打分
  const candidates = await scoreIntents(
    userMessage,
    sessionContext,
    possibleIntents
  );

  // 2. 路由
  const routing = confidenceRouter(candidates);

  switch (routing.strategy) {
    case 'execute': {
      // 直接执行，但日志记录置信度（方便事后审计）
      console.log(`[confidence=${routing.chosen!.confidence}] 执行: ${routing.chosen!.toolName}`);
      await executeTool(routing.chosen!.toolName, routing.chosen!.params);
      break;
    }

    case 'confirm': {
      // 把候选展示给用户，用内联按钮（OpenClaw Telegram inline buttons）
      const options = routing.candidates!.map((c, i) =>
        `${i + 1}. ${c.intent}（置信度 ${Math.round(c.confidence * 100)}%）`
      ).join('\n');
      await sendMessage(`我有几个理解，请确认哪个是你想要的：\n${options}`);
      // 等待用户选择，然后继续执行
      break;
    }

    case 'clarify': {
      await sendMessage(routing.clarifyQuestion!);
      break;
    }
  }
}
```

---

## OpenClaw 实战：内联按钮 + 置信度路由

OpenClaw 支持 Telegram inline buttons，正好配合"候选确认"策略：

```typescript
// OpenClaw skill 中使用 inlineButtons 能力
async function confirmWithButtons(candidates: IntentCandidate[]) {
  const buttons = candidates.map((c, i) => ({
    text: `${c.intent} (${Math.round(c.confidence * 100)}%)`,
    callbackData: `intent_select:${i}`
  }));

  await sendMessageWithButtons(
    '我不确定你的意思，请选择：',
    buttons
  );
}
```

---

## 置信度校准：避免过度自信

LLM 天生容易"过度自信"。需要校准：

```typescript
// 校准器：如果 top-1 和 top-2 分数接近，降低 top-1 的置信度
function calibrateConfidence(candidates: IntentCandidate[]): IntentCandidate[] {
  if (candidates.length < 2) return candidates;

  const [first, second] = candidates;
  const gap = first.confidence - second.confidence;

  // 如果差距小于 0.15，说明意图不明确，惩罚 top-1
  if (gap < 0.15) {
    return candidates.map((c, i) => ({
      ...c,
      confidence: i === 0
        ? c.confidence * (0.7 + gap * 2)  // 缩水
        : c.confidence
    }));
  }

  return candidates;
}
```

---

## 与已有模式的对比

| 模式 | 第52课（意图路由） | 本课（置信度感知） |
|------|------|------|
| 焦点 | 如何路由到正确处理器 | **如何量化路由的不确定性** |
| 输入 | 已识别意图 | 模糊输入 + 多候选 |
| 输出 | 单一处理路径 | execute / confirm / clarify 三策略 |
| 关键机制 | 规则/分类器 | **置信度分 + 阈值决策** |

---

## 关键配置参数

```typescript
const CONFIG = {
  // 调高 EXECUTE 阈值 → 更保守，更多确认步骤（适合高风险操作）
  executeThreshold: 0.85,
  
  // 调低 CONFIRM 阈值 → 更多候选展示（适合复杂场景）
  confirmThreshold: 0.50,
  
  // 最多展示几个候选（太多用户看烦）
  maxCandidates: 3,
  
  // 每个会话最多主动澄清几次（防止 Agent 反复提问）
  maxClarifyPerSession: 2,
};
```

---

## 总结

| 置信度 | 策略 | 体验 |
|--------|------|------|
| > 0.85 | 直接执行 | 流畅 |
| 0.5–0.85 | 展示候选 | 透明可控 |
| < 0.5 | 精准澄清 | 有目的地问 |

**核心价值**：把"我不确定"从一个 bug 变成一个**显式的路由分支**，让 Agent 的不确定性对用户透明、可控、可信。

与第52课（意图路由）、第101课（对话修复与澄清）一起读效果最佳。
