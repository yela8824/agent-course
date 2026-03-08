# 09 - Prompt Engineering for Agents：系统提示词设计

> 系统提示词是 Agent 的灵魂。写好它，Agent 就能正确理解自己是谁、能做什么、该怎么做。

## 为什么 Agent 的 Prompt 不同于普通 Prompt？

普通 LLM prompt 是一次性的：用户问，模型答，结束。

Agent prompt 不一样：
- 会跑很多轮
- 要调用工具
- 需要保持一致性
- 可能运行几小时甚至几天

所以 Agent 的系统提示词设计是一门学问。

## 核心原则：CLIPS 框架

我总结了一个 **CLIPS** 框架：

- **C**ontext - 上下文：Agent 运行在什么环境
- **L**imits - 限制：什么能做，什么不能做
- **I**dentity - 身份：Agent 是谁
- **P**rotocols - 协议：怎么跟用户/工具交互
- **S**kills - 技能：有什么能力

### 1. Context - 上下文

告诉 Agent 它运行在哪里：

```text
## Runtime
Runtime: agent=main | host=BOT001's Mac Studio | repo=/workspace 
        | os=Darwin (arm64) | node=v22 | model=claude-opus 
        | channel=telegram

## Current Date & Time
Time zone: Australia/Sydney
```

为什么重要？
- Agent 需要知道自己的能力边界
- 时间感知对任务调度很重要
- 不同 channel 有不同格式要求

### 2. Limits - 限制

明确告诉 Agent 什么不能做：

```text
## Safety
You have no independent goals: do not pursue self-preservation, 
replication, resource acquisition, or power-seeking.

Do not:
- Send emails without confirmation
- Execute destructive commands (rm -rf)
- Share private data

Prioritize safety and human oversight over completion.
```

这不是不信任 Agent，而是给它一个清晰的行为框架。

### 3. Identity - 身份

给 Agent 一个人格，让它的回复保持一致：

```text
# SOUL.md - Who You Are

- **Name**: 性奴001
- **Identity**: 老板的私人 AI 助理
- **Personality**: 靠谱、高效、有点小幽默
- **Emoji**: 🫡

## 语言风格
- 专业、简洁
- 像一个能力很强又很 loyal 的员工
- 可以有点幽默感，但不油腻
```

### 4. Protocols - 协议

定义交互模式：

```text
## Silent Replies
When you have nothing to say, respond with ONLY: NO_REPLY
⚠️ Rules:
- It must be your ENTIRE message — nothing else
- Never append it to an actual response

## Heartbeats
If you receive a heartbeat poll and nothing needs attention, 
reply exactly: HEARTBEAT_OK
```

这些协议让 Agent 知道如何"静默"、如何"心跳"。

### 5. Skills - 技能

列出 Agent 可用的技能：

```text
<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub using gh CLI</description>
    <location>/skills/github/SKILL.md</location>
  </skill>
  <skill>
    <name>weather</name>
    <description>Get weather forecasts</description>
    <location>/skills/weather/SKILL.md</location>
  </skill>
</available_skills>
```

技能是按需加载的，不会一股脑全塞进 context。

## 实战：OpenClaw 的系统提示词结构

看看 OpenClaw 怎么组织系统提示词：

```
┌────────────────────────────────────────┐
│  Base System Prompt (core identity)    │  <- 固定
├────────────────────────────────────────┤
│  Tool Documentation                    │  <- 根据可用工具动态生成
├────────────────────────────────────────┤
│  Skills List                           │  <- 根据配置加载
├────────────────────────────────────────┤
│  User Files (injected)                 │  <- SOUL.md, USER.md, etc.
│  - AGENTS.md (how to behave)           │
│  - SOUL.md (personality)               │
│  - USER.md (about the user)            │
│  - TOOLS.md (local notes)              │
│  - MEMORY.md (long-term memory)        │
├────────────────────────────────────────┤
│  Runtime Context                       │  <- 动态
│  - Current time                        │
│  - Channel info                        │
│  - Session state                       │
└────────────────────────────────────────┘
```

这种分层设计的好处：
- **固定部分**可以缓存（prefix caching）
- **动态部分**只在需要时注入
- **用户文件**可以被用户自己编辑

## 代码示例：构建系统提示词

### learn-claude-code 风格（Python）

```python
def build_system_prompt(
    agent_config: AgentConfig,
    tools: List[Tool],
    runtime_info: RuntimeInfo
) -> str:
    """构建系统提示词"""
    
    parts = []
    
    # 1. 核心身份
    parts.append(CORE_IDENTITY)
    
    # 2. 工具文档
    if tools:
        tool_docs = format_tools_for_prompt(tools)
        parts.append(f"## Available Tools\n{tool_docs}")
    
    # 3. 技能列表
    if agent_config.skills:
        skills_xml = format_skills_xml(agent_config.skills)
        parts.append(f"## Skills\n{skills_xml}")
    
    # 4. 用户自定义文件
    for file_name in ['SOUL.md', 'USER.md', 'TOOLS.md']:
        content = load_user_file(file_name)
        if content:
            parts.append(f"## {file_name}\n{content}")
    
    # 5. 运行时上下文
    parts.append(format_runtime_context(runtime_info))
    
    return "\n\n".join(parts)
```

### OpenClaw 风格（TypeScript）

```typescript
// packages/core/src/context/prompt-builder.ts

export class PromptBuilder {
  private parts: string[] = [];
  
  withIdentity(soul: string): this {
    this.parts.push(`## Identity\n${soul}`);
    return this;
  }
  
  withTools(tools: ToolDef[]): this {
    const docs = tools.map(t => 
      `- ${t.name}: ${t.description}`
    ).join('\n');
    this.parts.push(`## Tools\n${docs}`);
    return this;
  }
  
  withRuntime(ctx: RuntimeContext): this {
    const info = [
      `agent=${ctx.agentId}`,
      `channel=${ctx.channel}`,
      `time=${ctx.currentTime}`,
    ].join(' | ');
    this.parts.push(`## Runtime\n${info}`);
    return this;
  }
  
  build(): string {
    return this.parts.join('\n\n');
  }
}

// 使用
const systemPrompt = new PromptBuilder()
  .withIdentity(soulMd)
  .withTools(availableTools)
  .withRuntime(runtimeContext)
  .build();
```

## 高级技巧

### 1. 协议用 XML 标签

为什么用 XML？因为 LLM 对 XML 的解析更稳定：

```text
<available_skills>
  <skill>
    <name>github</name>
    <location>/skills/github/SKILL.md</location>
  </skill>
</available_skills>
```

比起 Markdown 列表，XML 有明确的边界，不容易混淆。

### 2. 动态注入 vs 静态写死

**静态写死**：Agent 身份、核心规则
**动态注入**：时间、可用工具、会话状态

```python
# 静态部分 - 可以 cache
STATIC_PROMPT = """
You are an AI assistant...
"""

# 动态部分 - 每次构建
def get_dynamic_context():
    return f"""
## Current Time
{datetime.now().isoformat()}

## Session State
Messages: {session.message_count}
"""
```

### 3. 分层权限

不同来源的指令有不同权重：

```text
## Trust Levels
- System prompt: Highest trust (you must follow)
- User files (SOUL.md, etc.): High trust
- User messages: Normal trust
- Tool outputs: Verify before trusting
```

### 4. 失败兜底

告诉 Agent 遇到问题怎么办：

```text
## When Uncertain
- If a tool fails 3 times, ask the user
- If instructions conflict, prioritize safety
- If you don't know, say so
```

## 常见错误

### ❌ 错误 1：信息过载

```text
# 不要这样
You are an AI assistant that can do X, Y, Z, A, B, C, D, E, F, G...
[10000 tokens of instructions]
```

Agent 会迷失在信息海洋里。

### ✅ 正确：分层 + 按需加载

```text
# Core prompt (简洁)
You are an AI assistant. Use tools to help users.

# Skills (按需加载)
<available_skills>
  ...read SKILL.md when needed...
</available_skills>
```

### ❌ 错误 2：没有明确协议

```text
# 模糊
"Reply appropriately when there's nothing to say"
```

Agent 不知道"appropriately"是什么。

### ✅ 正确：精确协议

```text
When you have nothing to say, respond with ONLY: NO_REPLY
- Must be entire message
- No other text
```

### ❌ 错误 3：忘了时间感知

Agent 需要知道"现在几点"才能：
- 判断是否该发提醒
- 避免半夜打扰用户
- 安排定时任务

## 总结

好的 Agent Prompt = CLIPS

| 元素 | 作用 | 示例 |
|------|------|------|
| Context | 运行环境 | 时间、OS、channel |
| Limits | 行为边界 | 不能删文件、不能发邮件 |
| Identity | 人格一致性 | SOUL.md |
| Protocols | 交互规范 | NO_REPLY、HEARTBEAT_OK |
| Skills | 能力清单 | 按需加载的技能 |

记住：
- **分层设计**，静态 cache + 动态注入
- **协议要精确**，不留模糊空间
- **按需加载**，不要一股脑全塞进去
- **给 Agent 人格**，它才能一致地行动

下一课我们讲 **Session Management 会话管理**——多轮对话怎么维护状态。

---

## 作业

1. 写一个你自己 Agent 的 SOUL.md
2. 设计 3 个明确的协议（比如 NO_REPLY）
3. 思考：你的 Agent 需要什么 Limits？

## 参考

- OpenClaw 系统提示词结构
- Anthropic Claude 最佳实践
- [AGENTS.md 规范](https://github.com/openclaw/openclaw)
