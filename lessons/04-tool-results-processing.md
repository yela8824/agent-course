# 04 - 工具结果处理：Agent 如何理解工具的回答

上节课讲了错误处理，这节课讲一个更基础但经常被忽视的问题：**工具执行完后，结果怎么传回给 LLM？**

这看起来简单——把结果塞回 prompt 不就行了？但实际上，工具结果处理的质量直接决定了 Agent 的"理解能力"。

## 核心问题

工具结果处理要解决三个问题：

1. **格式化**：工具返回的原始数据如何变成 LLM 能理解的格式
2. **截断**：结果太长怎么办（10MB 的文件内容？）
3. **上下文**：如何让 LLM 知道这个结果对应哪个工具调用

## 实际实现

### learn-claude-code 的方式

```python
# utils/message.py
def format_tool_result(tool_use_id: str, result: Any) -> dict:
    """格式化工具执行结果"""
    
    # 把任何类型的结果转成字符串
    if isinstance(result, str):
        content = result
    elif isinstance(result, dict):
        content = json.dumps(result, indent=2, ensure_ascii=False)
    elif isinstance(result, list):
        content = json.dumps(result, indent=2, ensure_ascii=False)
    else:
        content = str(result)
    
    # 截断过长的结果
    MAX_LENGTH = 100000  # ~100KB
    if len(content) > MAX_LENGTH:
        content = content[:MAX_LENGTH] + f"\n\n[Truncated: {len(content)} chars total]"
    
    return {
        "type": "tool_result",
        "tool_use_id": tool_use_id,  # 关键：关联到对应的 tool_use
        "content": content
    }
```

关键点：
- `tool_use_id` 把结果和调用关联起来
- 所有类型都转成字符串（LLM 只懂文本）
- 超长内容必须截断

### pi-mono 的高级处理

pi-mono 对不同类型的工具结果有专门的处理：

```typescript
// packages/agent/src/tools/results.ts

export function formatToolResult(
  toolName: string,
  result: ToolResult
): string {
  // 文件内容：加上行号便于定位
  if (toolName === 'Read' && typeof result === 'string') {
    return addLineNumbers(result);
  }
  
  // 命令输出：区分 stdout 和 stderr
  if (toolName === 'Bash' && isCommandResult(result)) {
    return formatCommandOutput(result);
  }
  
  // 搜索结果：结构化展示
  if (toolName === 'Search' && Array.isArray(result)) {
    return formatSearchResults(result);
  }
  
  // 默认：JSON 序列化
  return JSON.stringify(result, null, 2);
}

function formatCommandOutput(result: CommandResult): string {
  let output = '';
  
  if (result.stdout) {
    output += result.stdout;
  }
  
  if (result.stderr) {
    output += '\n[stderr]\n' + result.stderr;
  }
  
  output += `\n[exit code: ${result.exitCode}]`;
  
  return output;
}

function addLineNumbers(content: string): string {
  return content
    .split('\n')
    .map((line, i) => `${(i + 1).toString().padStart(4)} | ${line}`)
    .join('\n');
}
```

为什么要这样？
- **行号**：让 LLM 能精确引用"第 42 行"
- **区分 stderr**：错误信息和正常输出要区分开
- **exit code**：让 LLM 知道命令是否成功

### OpenClaw 的结果压缩

OpenClaw 面对的是长时间运行的 Agent，工具结果会累积。它的策略是**按重要性压缩**：

```typescript
// 结果压缩策略
function compressToolResults(messages: Message[]): Message[] {
  return messages.map(msg => {
    if (msg.role !== 'tool') return msg;
    
    // 旧的工具结果可以压缩
    const age = Date.now() - msg.timestamp;
    if (age > 30 * 60 * 1000) { // 30 分钟前的结果
      return {
        ...msg,
        content: summarizeResult(msg.content) // 保留摘要
      };
    }
    
    return msg;
  });
}

function summarizeResult(content: string): string {
  // 只保留前 500 字符 + 后 200 字符
  if (content.length <= 1000) return content;
  
  return (
    content.slice(0, 500) +
    '\n\n[... content compressed ...]\n\n' +
    content.slice(-200)
  );
}
```

## 图片和二进制结果

现代 Agent 经常需要处理图片。这时候不能直接把二进制塞进文本：

```typescript
// 处理图片结果
function formatImageResult(imagePath: string): ToolResultContent {
  const buffer = fs.readFileSync(imagePath);
  const base64 = buffer.toString('base64');
  const mimeType = getMimeType(imagePath);
  
  return {
    type: 'image',
    source: {
      type: 'base64',
      media_type: mimeType,
      data: base64
    }
  };
}

// 混合结果：文本 + 图片
function formatMixedResult(
  text: string,
  images: string[]
): ToolResultContent[] {
  const result: ToolResultContent[] = [
    { type: 'text', text }
  ];
  
  for (const img of images) {
    result.push(formatImageResult(img));
  }
  
  return result;
}
```

## 错误结果的特殊处理

工具失败时，结果格式很重要：

```typescript
function formatErrorResult(error: Error): ToolResultContent {
  // 结构化错误信息
  return {
    type: 'text',
    text: `[Error] ${error.name}: ${error.message}

Stack trace:
${error.stack?.split('\n').slice(0, 5).join('\n')}

Suggestion: ${getSuggestion(error)}`
  };
}

function getSuggestion(error: Error): string {
  // 根据错误类型给建议
  if (error.message.includes('ENOENT')) {
    return 'File not found. Check the path or use Search to find the file.';
  }
  if (error.message.includes('EACCES')) {
    return 'Permission denied. The file may be read-only or protected.';
  }
  if (error.message.includes('timeout')) {
    return 'Operation timed out. Try with a smaller scope or retry.';
  }
  return 'Review the error and try an alternative approach.';
}
```

给 LLM **可执行的建议**比只给错误信息有用得多。

## 并行工具调用的结果处理

当 Agent 一次调用多个工具时，结果的顺序和关联很重要：

```typescript
async function executeParallelTools(
  toolCalls: ToolCall[]
): Promise<ToolResultMessage[]> {
  // 并行执行
  const results = await Promise.all(
    toolCalls.map(async (call) => {
      try {
        const result = await executeTool(call);
        return { call, result, error: null };
      } catch (error) {
        return { call, result: null, error };
      }
    })
  );
  
  // 按原始顺序返回结果
  return results.map(({ call, result, error }) => ({
    role: 'tool',
    tool_use_id: call.id,
    content: error
      ? formatErrorResult(error)
      : formatToolResult(call.name, result)
  }));
}
```

关键：**保持顺序**。如果 LLM 调用了 `[Read A, Read B, Read C]`，返回顺序必须一致，否则 LLM 会搞混。

## 实战技巧

### 1. 结果预览

对于大文件，先给摘要再给全文：

```typescript
function formatLargeFileResult(content: string, path: string): string {
  const lines = content.split('\n');
  
  // 元信息
  let result = `File: ${path}\n`;
  result += `Lines: ${lines.length}\n`;
  result += `Size: ${content.length} bytes\n`;
  result += `---\n\n`;
  
  // 如果太长，只给前后部分
  if (lines.length > 200) {
    result += lines.slice(0, 100).map((l, i) => `${i + 1} | ${l}`).join('\n');
    result += `\n\n... [${lines.length - 150} lines omitted] ...\n\n`;
    result += lines.slice(-50).map((l, i) => `${lines.length - 49 + i} | ${l}`).join('\n');
  } else {
    result += lines.map((l, i) => `${i + 1} | ${l}`).join('\n');
  }
  
  return result;
}
```

### 2. 结构化 vs 自由文本

有时候结构化输出更好：

```typescript
// 差：自由文本
"Found 3 matches in file.js: line 42 has 'TODO', line 78 has 'FIXME'..."

// 好：结构化
`Search results for "TODO|FIXME" in file.js:

| Line | Type  | Content                    |
|------|-------|----------------------------|
| 42   | TODO  | // TODO: refactor this     |
| 78   | FIXME | // FIXME: potential bug    |
| 156  | TODO  | // TODO: add error handling|

Total: 3 matches`
```

### 3. 增量结果

对于流式工具（比如长时间运行的命令），可以增量返回：

```typescript
async function* executeStreamingTool(
  command: string
): AsyncGenerator<PartialResult> {
  const proc = spawn(command);
  
  let buffer = '';
  
  for await (const chunk of proc.stdout) {
    buffer += chunk;
    
    // 每 1KB 或每秒返回一次部分结果
    if (buffer.length >= 1024) {
      yield { partial: true, content: buffer };
      buffer = '';
    }
  }
  
  // 最终结果
  yield { partial: false, content: buffer };
}
```

## 小结

工具结果处理的核心原则：

1. **可读性**：LLM 是"阅读"结果的，格式要清晰
2. **可定位**：行号、路径、ID 让 LLM 能精确引用
3. **可操作**：错误要带建议，不只是报错
4. **可管理**：长结果要截断，旧结果要压缩

下节课讲 **Cost Optimization（成本优化）**——怎么让 Agent 又聪明又省钱。

---

*Agent 开发课程 #04 - 2026-03-08*
