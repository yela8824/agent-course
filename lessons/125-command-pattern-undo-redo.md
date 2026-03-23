# 125 - Agent 命令模式与撤销机制（Command Pattern & Undo/Redo）

> 把每次工具调用封装成可撤销的 Command 对象，实现事务级回滚、操作历史追踪与"后悔药"机制。

---

## 为什么需要 Undo/Redo？

Agent 执行工具调用时，常见的破坏性操作：

- 删除文件、覆盖数据库记录
- 发送邮件/消息（部分可撤回）
- 部署代码（可以 rollback）
- 创建资源（可以删除）

传统 Agent 一旦执行就无法反悔。**Command Pattern** 让每个操作都携带"如何撤销"的信息，Agent 随时可以回头。

---

## 核心概念

```
┌─────────────────────────────────────────┐
│              Command Interface          │
│  execute() → Result                     │
│  undo()    → void                       │
│  describe()→ string                     │
└─────────────────────────────────────────┘
           ↑
    ┌──────┴──────┐
    │             │
 ConcreteCmd   ConcreteCmd
 (WriteFile)   (DeleteRecord)

┌─────────────────────────────────────────┐
│            CommandHistory               │
│  stack: Command[]    (undo stack)       │
│  redoStack: Command[]                   │
│  execute(cmd) → push & run              │
│  undo()       → pop & revert            │
│  redo()       → restore from redoStack  │
└─────────────────────────────────────────┘
```

---

## TypeScript 实现（结合 pi-mono 工具调用模式）

```typescript
// command.ts - 核心接口
export interface Command<T = unknown> {
  /** 执行操作，返回结果 */
  execute(): Promise<T>;
  /** 撤销操作 */
  undo(): Promise<void>;
  /** 人类可读描述 */
  describe(): string;
  /** 是否支持撤销 */
  readonly reversible: boolean;
}

// history.ts - 命令历史管理器
export class CommandHistory {
  private undoStack: Command[] = [];
  private redoStack: Command[] = [];
  private readonly maxSize: number;

  constructor(maxSize = 50) {
    this.maxSize = maxSize;
  }

  async execute<T>(cmd: Command<T>): Promise<T> {
    const result = await cmd.execute();

    if (cmd.reversible) {
      this.undoStack.push(cmd);
      this.redoStack = []; // 执行新命令清空 redo 栈

      // 防止无限增长
      if (this.undoStack.length > this.maxSize) {
        this.undoStack.shift();
      }
    }

    console.log(`[CMD] executed: ${cmd.describe()}`);
    return result;
  }

  async undo(): Promise<string> {
    const cmd = this.undoStack.pop();
    if (!cmd) return 'nothing to undo';

    await cmd.undo();
    this.redoStack.push(cmd);
    return `undone: ${cmd.describe()}`;
  }

  async redo(): Promise<string> {
    const cmd = this.redoStack.pop();
    if (!cmd) return 'nothing to redo';

    await cmd.execute();
    this.undoStack.push(cmd);
    return `redone: ${cmd.describe()}`;
  }

  getHistory(): string[] {
    return this.undoStack.map(c => c.describe());
  }
}
```

---

## 具体命令实现：文件操作

```typescript
import fs from 'fs/promises';

// 写文件命令（可撤销）
export class WriteFileCommand implements Command<void> {
  readonly reversible = true;
  private previousContent: string | null = null;
  private fileExistedBefore = false;

  constructor(
    private readonly path: string,
    private readonly newContent: string
  ) {}

  async execute(): Promise<void> {
    // 保存旧内容用于撤销
    try {
      this.previousContent = await fs.readFile(this.path, 'utf-8');
      this.fileExistedBefore = true;
    } catch {
      this.fileExistedBefore = false;
    }
    await fs.writeFile(this.path, this.newContent, 'utf-8');
  }

  async undo(): Promise<void> {
    if (!this.fileExistedBefore) {
      await fs.unlink(this.path); // 原本不存在，撤销就删掉
    } else {
      await fs.writeFile(this.path, this.previousContent!, 'utf-8');
    }
  }

  describe(): string {
    return `WriteFile(${this.path}, ${this.newContent.length} bytes)`;
  }
}

// 删除文件命令（可撤销）
export class DeleteFileCommand implements Command<void> {
  readonly reversible = true;
  private deletedContent: string | null = null;

  constructor(private readonly path: string) {}

  async execute(): Promise<void> {
    this.deletedContent = await fs.readFile(this.path, 'utf-8');
    await fs.unlink(this.path);
  }

  async undo(): Promise<void> {
    if (this.deletedContent !== null) {
      await fs.writeFile(this.path, this.deletedContent, 'utf-8');
    }
  }

  describe(): string {
    return `DeleteFile(${this.path})`;
  }
}
```

---

## 不可撤销操作的处理

有些操作真的无法撤销（发邮件、转账），但仍应纳入历史追踪：

```typescript
export class SendEmailCommand implements Command<void> {
  readonly reversible = false; // 明确标记不可撤销

  constructor(
    private readonly to: string,
    private readonly subject: string,
    private readonly body: string
  ) {}

  async execute(): Promise<void> {
    // 调用邮件 API...
    console.log(`Email sent to ${this.to}`);
  }

  async undo(): Promise<void> {
    // 无法真正撤销，但可以发一封"撤回/更正"邮件
    throw new Error(`SendEmail is not reversible. Consider sending a correction email.`);
  }

  describe(): string {
    return `SendEmail(to=${this.to}, subject="${this.subject}")`;
  }
}
```

---

## 宏命令：事务组合

把多个命令组合成一个原子事务，要么全成功，要么全回滚：

```typescript
export class MacroCommand implements Command<void[]> {
  readonly reversible = true;
  private executed: Command[] = [];

  constructor(
    private readonly commands: Command[],
    private readonly name: string
  ) {}

  async execute(): Promise<void[]> {
    this.executed = [];
    try {
      const results: void[] = [];
      for (const cmd of this.commands) {
        await cmd.execute();
        this.executed.push(cmd);
        results.push();
      }
      return results;
    } catch (err) {
      // 部分失败 → 逆序撤销已执行的命令
      console.error(`MacroCommand "${this.name}" failed, rolling back...`);
      for (const cmd of [...this.executed].reverse()) {
        if (cmd.reversible) await cmd.undo().catch(console.error);
      }
      throw err;
    }
  }

  async undo(): Promise<void> {
    for (const cmd of [...this.executed].reverse()) {
      if (cmd.reversible) await cmd.undo();
    }
  }

  describe(): string {
    return `Macro(${this.name}: ${this.commands.map(c => c.describe()).join(' → ')})`;
  }
}
```

---

## 与 Agent 工具系统集成

在 Agent 的 tool dispatch 层包一层 CommandHistory：

```typescript
// agent-tool-dispatcher.ts
export class CommandAwareToolDispatcher {
  private history = new CommandHistory(100);

  async dispatch(toolName: string, params: Record<string, unknown>) {
    const cmd = this.buildCommand(toolName, params);
    return this.history.execute(cmd);
  }

  // 暴露给 LLM 的特殊工具
  async undoLast() {
    return this.history.undo();
  }

  async redoLast() {
    return this.history.redo();
  }

  showHistory() {
    return this.history.getHistory();
  }

  private buildCommand(toolName: string, params: Record<string, unknown>): Command {
    switch (toolName) {
      case 'write_file':
        return new WriteFileCommand(params.path as string, params.content as string);
      case 'delete_file':
        return new DeleteFileCommand(params.path as string);
      case 'send_email':
        return new SendEmailCommand(
          params.to as string,
          params.subject as string,
          params.body as string
        );
      default:
        throw new Error(`Unknown tool: ${toolName}`);
    }
  }
}
```

---

## OpenClaw 里的实际应用

OpenClaw 的 Skills 执行时也可以应用这个模式：

```typescript
// skill-runner.ts (伪代码)
const dispatcher = new CommandAwareToolDispatcher();

// 用户说"帮我重构这三个文件"
await dispatcher.dispatch('write_file', { path: 'a.ts', content: newA });
await dispatcher.dispatch('write_file', { path: 'b.ts', content: newB });
await dispatcher.dispatch('write_file', { path: 'c.ts', content: newC });

// 用户说"不对，撤回"
await dispatcher.undoLast(); // 撤销 c.ts
await dispatcher.undoLast(); // 撤销 b.ts
await dispatcher.undoLast(); // 撤销 a.ts

// 用户说"算了还是用新版本"
await dispatcher.redoLast(); // 恢复 a.ts
```

---

## 设计要点总结

| 要点 | 说明 |
|------|------|
| 快照 vs 逆操作 | 写文件→保存旧内容；删记录→保存被删数据 |
| 不可逆操作 | 明确标记 `reversible=false`，history 仍记录但不进 undo 栈 |
| 宏命令 | 多步操作原子化，失败时逆序回滚 |
| 栈大小限制 | 防止内存泄漏，超限丢弃最老的 |
| redo 栈清空 | 执行新命令时必须清空 redo 栈（经典行为） |
| 持久化 | 命令可序列化到 Redis，支持跨进程 undo |

---

## 一句话总结

> **Command Pattern = 工具调用 + 如何反悔。** 把每次操作封装成携带 undo 信息的对象，Agent 就有了"后悔药"，从"执行了就完了"升级到"随时可回退"。

---

*Lesson 125 | Agent 开发课程 | 2026-03-24*
