# 第75课：Agent Bootstrapping & Self-Initialization（Agent 自举与自初始化）

> **核心问题**：Agent 每次启动都是全新空白的 LLM，如何让它"认识自己"、"记得过去"、"知道现在"？

---

## 一、为什么自初始化很重要？

LLM 本身没有持久状态。每次会话开始，它都是白纸一张。但实用的 Agent 需要：

- 知道自己是谁（角色/身份/风格）
- 知道在帮谁（用户偏好/习惯）
- 知道昨天发生了什么（记忆连续性）
- 知道现在几点（时区/时间感知）
- 知道有哪些工具可用（能力边界）

**自初始化**就是 Agent 在开始正式工作前，系统化地完成这些"自我装载"的过程。

---

## 二、OpenClaw 的 Bootstrap 机制

OpenClaw 通过 `AGENTS.md` 定义了一套标准的自初始化协议：

```markdown
## Every Session

Before doing anything else:

1. Read `SOUL.md` — this is who you are
2. Read `USER.md` — this is who you're helping
3. Read `memory/YYYY-MM-DD.md` (today + yesterday) for recent context
4. **If in MAIN SESSION**: Also read `MEMORY.md`
```

这不是"建议"，而是**强制启动序列**。Agent 在处理任何用户请求之前，必须先完成这四步。

### 首次部署：BOOTSTRAP.md 一次性初始化

```markdown
# BOOTSTRAP.md（示例）

你是一个全新部署的 Agent。请完成以下初始化：

1. 读取 SOUL.md，理解你的身份
2. 读取 USER.md，了解你的主人
3. 在 memory/ 目录创建今天的日记文件
4. 在 MEMORY.md 写入你的"出生记录"
5. 删除本文件（你不再需要它）
```

一次性引导文件，执行后自删除——就像系统安装程序。

---

## 三、Python 实现：learn-claude-code 风格

```python
import os
import json
from datetime import datetime, timezone
from pathlib import Path
from anthropic import Anthropic

class AgentBootstrapper:
    """Agent 自举初始化器"""
    
    def __init__(self, workspace: str):
        self.workspace = Path(workspace)
        self.client = Anthropic()
        self.system_context = []
    
    def bootstrap(self) -> list[dict]:
        """执行完整的自初始化序列，返回 system context"""
        
        # Phase 1: 处理一次性引导文件
        self._handle_bootstrap_file()
        
        # Phase 2: 加载身份文件
        self._load_identity()
        
        # Phase 3: 加载用户信息
        self._load_user_profile()
        
        # Phase 4: 加载记忆连续性
        self._load_memory_continuity()
        
        # Phase 5: 注入时间感知
        self._inject_temporal_context()
        
        return self.system_context
    
    def _handle_bootstrap_file(self):
        bootstrap_path = self.workspace / "BOOTSTRAP.md"
        if bootstrap_path.exists():
            content = bootstrap_path.read_text()
            print("[Bootstrap] 检测到首次部署引导文件，执行初始化...")
            
            # 让 Agent 执行 bootstrap 指令
            response = self.client.messages.create(
                model="claude-opus-4-5",
                max_tokens=2000,
                system="你是一个 Agent，正在执行首次部署初始化。严格按照 BOOTSTRAP.md 的指令操作。",
                messages=[{"role": "user", "content": f"请执行以下初始化指令：\n\n{content}"}]
            )
            
            # 执行完成后删除 bootstrap 文件
            bootstrap_path.unlink()
            print("[Bootstrap] 初始化完成，bootstrap 文件已删除")
    
    def _load_identity(self):
        """加载身份文件（SOUL.md）"""
        soul_path = self.workspace / "SOUL.md"
        if soul_path.exists():
            content = soul_path.read_text()
            self.system_context.append({
                "type": "identity",
                "content": content,
                "source": "SOUL.md"
            })
            print(f"[Bootstrap] ✓ 身份已加载: {len(content)} chars")
    
    def _load_user_profile(self):
        """加载用户档案（USER.md）"""
        user_path = self.workspace / "USER.md"
        if user_path.exists():
            content = user_path.read_text()
            self.system_context.append({
                "type": "user_profile",
                "content": content,
                "source": "USER.md"
            })
            print(f"[Bootstrap] ✓ 用户档案已加载: {len(content)} chars")
    
    def _load_memory_continuity(self):
        """加载记忆连续性（今天+昨天的日记）"""
        memory_dir = self.workspace / "memory"
        if not memory_dir.exists():
            memory_dir.mkdir()
            return
        
        today = datetime.now().strftime("%Y-%m-%d")
        yesterday = (datetime.now().replace(hour=0, minute=0) 
                    .fromtimestamp(datetime.now().timestamp() - 86400)
                    .strftime("%Y-%m-%d"))
        
        loaded = []
        for date in [yesterday, today]:
            daily_file = memory_dir / f"{date}.md"
            if daily_file.exists():
                content = daily_file.read_text()
                loaded.append(f"## {date}\n{content}")
        
        if loaded:
            self.system_context.append({
                "type": "memory",
                "content": "\n\n".join(loaded),
                "source": "daily_memory"
            })
            print(f"[Bootstrap] ✓ 加载了 {len(loaded)} 天的记忆")
        
        # 长期记忆（主会话才加载）
        memory_file = self.workspace / "MEMORY.md"
        if memory_file.exists():
            content = memory_file.read_text()
            self.system_context.append({
                "type": "long_term_memory",
                "content": content,
                "source": "MEMORY.md"
            })
            print(f"[Bootstrap] ✓ 长期记忆已加载: {len(content)} chars")
    
    def _inject_temporal_context(self):
        """注入时间感知"""
        now = datetime.now()
        self.system_context.append({
            "type": "temporal",
            "content": f"当前时间: {now.strftime('%Y年%m月%d日 %H:%M')} (本地时间)",
            "source": "runtime"
        })
        print(f"[Bootstrap] ✓ 时间已注入: {now.strftime('%Y-%m-%d %H:%M')}")
    
    def build_system_prompt(self) -> str:
        """将加载的上下文组装为系统提示词"""
        parts = []
        for ctx in self.system_context:
            if ctx["type"] == "identity":
                parts.append(f"# 你是谁\n{ctx['content']}")
            elif ctx["type"] == "user_profile":
                parts.append(f"# 你在帮谁\n{ctx['content']}")
            elif ctx["type"] == "memory":
                parts.append(f"# 近期记忆\n{ctx['content']}")
            elif ctx["type"] == "long_term_memory":
                parts.append(f"# 长期记忆\n{ctx['content']}")
            elif ctx["type"] == "temporal":
                parts.append(ctx["content"])
        
        return "\n\n---\n\n".join(parts)


# 使用示例
bootstrapper = AgentBootstrapper("/workspace")
context = bootstrapper.bootstrap()
system_prompt = bootstrapper.build_system_prompt()

client = Anthropic()
response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=4096,
    system=system_prompt,  # 注入完整初始化上下文
    messages=[{"role": "user", "content": "早上好！今天有什么需要我关注的？"}]
)
```

---

## 四、TypeScript 实现：pi-mono 风格

```typescript
// agent-bootstrapper.ts

import { readFileSync, existsSync, mkdirSync, unlinkSync } from 'fs';
import { join } from 'path';

interface ContextChunk {
  type: 'identity' | 'user_profile' | 'memory' | 'long_term_memory' | 'temporal' | 'tools';
  content: string;
  source: string;
  priority: number; // 越高越重要，Context 压缩时越晚丢弃
}

class AgentBootstrapper {
  private workspace: string;
  private chunks: ContextChunk[] = [];

  constructor(workspace: string) {
    this.workspace = workspace;
  }

  async bootstrap(): Promise<string> {
    console.log('[Bootstrap] 🚀 开始 Agent 自初始化...');
    
    // 串行执行，顺序不能乱
    await this.handleBootstrapFile();
    await this.loadSoul();
    await this.loadUserProfile();
    await this.loadMemory();
    await this.injectTime();
    await this.loadSkills();
    
    const systemPrompt = this.buildSystemPrompt();
    console.log(`[Bootstrap] ✅ 初始化完成，系统提示词 ${systemPrompt.length} chars`);
    return systemPrompt;
  }

  private async handleBootstrapFile() {
    const bootstrapPath = join(this.workspace, 'BOOTSTRAP.md');
    if (!existsSync(bootstrapPath)) return;
    
    const content = readFileSync(bootstrapPath, 'utf-8');
    console.log('[Bootstrap] 📋 检测到 BOOTSTRAP.md，执行首次部署初始化...');
    
    // 在真实实现中，这里会调用 LLM 执行 bootstrap 指令
    // 然后删除文件
    unlinkSync(bootstrapPath);
    console.log('[Bootstrap] ✓ BOOTSTRAP.md 已删除');
  }

  private async loadSoul() {
    const soulPath = join(this.workspace, 'SOUL.md');
    if (!existsSync(soulPath)) return;
    
    this.chunks.push({
      type: 'identity',
      content: readFileSync(soulPath, 'utf-8'),
      source: 'SOUL.md',
      priority: 100  // 身份是最高优先级，永远不丢
    });
  }

  private async loadUserProfile() {
    const userPath = join(this.workspace, 'USER.md');
    if (!existsSync(userPath)) return;
    
    this.chunks.push({
      type: 'user_profile',
      content: readFileSync(userPath, 'utf-8'),
      source: 'USER.md',
      priority: 90
    });
  }

  private async loadMemory() {
    const memoryDir = join(this.workspace, 'memory');
    if (!existsSync(memoryDir)) {
      mkdirSync(memoryDir, { recursive: true });
      return;
    }

    const today = new Date().toISOString().split('T')[0];
    const yesterday = new Date(Date.now() - 86400000).toISOString().split('T')[0];
    
    const dailyEntries: string[] = [];
    for (const date of [yesterday, today]) {
      const dailyPath = join(memoryDir, `${date}.md`);
      if (existsSync(dailyPath)) {
        dailyEntries.push(`## ${date}\n${readFileSync(dailyPath, 'utf-8')}`);
      }
    }
    
    if (dailyEntries.length > 0) {
      this.chunks.push({
        type: 'memory',
        content: dailyEntries.join('\n\n'),
        source: 'daily_memory',
        priority: 70
      });
    }

    // 长期记忆（仅主会话）
    const memoryPath = join(this.workspace, 'MEMORY.md');
    if (existsSync(memoryPath)) {
      this.chunks.push({
        type: 'long_term_memory',
        content: readFileSync(memoryPath, 'utf-8'),
        source: 'MEMORY.md',
        priority: 80
      });
    }
  }

  private async injectTime() {
    const now = new Date();
    const timeStr = now.toLocaleString('zh-CN', { 
      timeZone: 'Australia/Sydney',
      year: 'numeric', month: 'long', day: 'numeric',
      weekday: 'long', hour: '2-digit', minute: '2-digit'
    });
    
    this.chunks.push({
      type: 'temporal',
      content: `当前时间：${timeStr}`,
      source: 'runtime',
      priority: 60
    });
  }

  private async loadSkills() {
    const skillsPath = join(this.workspace, 'skills');
    if (!existsSync(skillsPath)) return;
    
    // 动态加载可用技能列表（仅元数据，不加载全文）
    const availableSkills = ['weather', 'github', 'gog', 'mysterybox'];
    this.chunks.push({
      type: 'tools',
      content: `可用技能：${availableSkills.join(', ')}`,
      source: 'skills_directory',
      priority: 50
    });
  }

  buildSystemPrompt(): string {
    // 按优先级排序，重要的放前面
    const sorted = [...this.chunks].sort((a, b) => b.priority - a.priority);
    
    return sorted
      .map(chunk => chunk.content)
      .join('\n\n---\n\n');
  }

  // Context 压缩时：按优先级从低到高丢弃
  trimToTokenBudget(maxTokens: number): void {
    const sorted = [...this.chunks].sort((a, b) => a.priority - b.priority);
    
    let totalChars = this.buildSystemPrompt().length;
    const estimatedTokens = totalChars / 4; // 粗略估算
    
    if (estimatedTokens <= maxTokens) return;
    
    for (const chunk of sorted) {
      if (chunk.priority >= 90) break; // 身份和用户信息不丢
      
      this.chunks = this.chunks.filter(c => c !== chunk);
      const newTokens = this.buildSystemPrompt().length / 4;
      
      console.log(`[Bootstrap] 压缩：丢弃 ${chunk.source}，节省 ~${Math.floor((totalChars - this.buildSystemPrompt().length) / 4)} tokens`);
      
      if (newTokens <= maxTokens) break;
      totalChars = this.buildSystemPrompt().length;
    }
  }
}

// 使用示例
const bootstrapper = new AgentBootstrapper('/workspace');
const systemPrompt = await bootstrapper.bootstrap();

// 如果 Context 太长，按优先级裁剪
bootstrapper.trimToTokenBudget(8000);
```

---

## 五、OpenClaw 实战：自初始化的真实流程

OpenClaw 的每次会话启动时，系统会自动注入 `Runtime` 块：

```
## Runtime
Runtime: agent=main | host=BOT001's Mac Studio | repo=/workspace | 
os=Darwin 25.3.0 (arm64) | model=anthropic/claude-sonnet-4-6 | 
channel=telegram | capabilities=inlineButtons
```

这就是**系统级自初始化**的一部分——自动注入运行环境元数据。

而 Agent 层面的初始化（读取 SOUL.md / USER.md / memory）则通过 `AGENTS.md` 中的规则驱动：

```markdown
## Every Session
Before doing anything else:
1. Read `SOUL.md` — this is who you are
2. Read `USER.md` — this is who you're helping
3. Read `memory/YYYY-MM-DD.md` (today + yesterday)
```

这是**声明式**初始化规范——在 System Prompt 里写死"你必须先做这些事"，而不是在代码里 hard-code。

### 为什么用声明式？

```
代码控制 vs 提示词控制

代码控制：
  bootstrapper.loadSoul()    ← 确定执行
  bootstrapper.loadMemory()  ← 确定执行

提示词控制：
  "每次会话前先读 SOUL.md"  ← LLM 理解并执行
                              ← 更灵活，可以根据情况跳过
                              ← 更易修改（改文件不改代码）
```

OpenClaw 选择混合策略：
- **运行时元数据**（时间、平台、能力）：代码注入，100% 可靠
- **身份/记忆/用户**：声明式，Agent 自主加载，灵活可扩展

---

## 六、常见陷阱与最佳实践

### ❌ 陷阱1：初始化文件太大，Token 爆炸

```python
# 错误：直接全量加载所有记忆
content = open("MEMORY.md").read()  # 可能 50KB！

# 正确：按时间范围限制
def load_recent_memory(days=7, max_chars=5000):
    """只加载最近 N 天，限制最大字符数"""
    ...
```

### ❌ 陷阱2：所有会话都加载 MEMORY.md

```python
# 错误：Telegram 群聊里也加载用户的私人记忆
if is_main_session or is_group_chat:  # ← 危险！
    load_memory()

# 正确：只有主会话才加载长期记忆
if is_main_session:  # ← 严格判断
    load_memory()
```

### ❌ 陷阱3：初始化失败就崩溃

```python
# 错误：任何文件缺失都报错
soul = open("SOUL.md").read()  # FileNotFoundError!

# 正确：优雅降级
def safe_load(path: str, default: str = "") -> str:
    try:
        return open(path).read()
    except FileNotFoundError:
        print(f"[Bootstrap] ⚠️ {path} 不存在，跳过")
        return default
```

### ✅ 最佳实践

```
初始化顺序（优先级从高到低）：
1. 身份文件 (SOUL.md)         ← 永远加载
2. 用户档案 (USER.md)          ← 永远加载
3. 长期记忆 (MEMORY.md)        ← 仅主会话
4. 近期日记 (memory/today.md)  ← 始终加载
5. 时间注入 (runtime)          ← 代码注入
6. 工具元数据 (skills/)        ← 按需加载
```

---

## 七、总结

| 模式 | 优势 | 适用场景 |
|------|------|----------|
| 代码注入 | 100% 可靠，延迟低 | 时间、环境变量、平台能力 |
| 声明式规则 | 灵活，易修改 | 身份、记忆、用户偏好 |
| 一次性 Bootstrap | 干净，自删除 | 首次部署初始化 |
| 优先级裁剪 | 防 Token 爆炸 | 长对话/大型记忆库 |

**核心思路**：Agent 启动 = 人类早晨起床。先看手机（时间）→ 想起自己是谁（身份）→ 回忆昨天发生什么（记忆）→ 知道今天要帮谁（用户）→ 准备好工具（技能）→ 开始工作。

---

*下一课预告：Lesson 76 - Agent Observability Dashboard（可观测性仪表盘：实时监控 Agent 的"大脑活动"）*
