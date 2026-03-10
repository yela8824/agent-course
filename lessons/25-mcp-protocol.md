# 第 25 课：MCP (Model Context Protocol) - Agent 工具交互的标准化协议

## 🎯 本课目标

理解 MCP 协议的设计理念、核心概念，以及如何在 Agent 中集成 MCP 服务器。

## 为什么需要 MCP？

在 Agent 开发中，我们面临一个现实问题：**工具集成的碎片化**。

```
传统方式：
┌─────────┐     ┌─────────────┐
│  Agent  │────▶│  Tool A     │  (自定义协议)
└─────────┘     └─────────────┘
     │          ┌─────────────┐
     └─────────▶│  Tool B     │  (另一种协议)
                └─────────────┘
     │          ┌─────────────┐
     └─────────▶│  Tool C     │  (又一种协议)
                └─────────────┘

MCP 方式：
┌─────────┐     ┌─────────────────────────────┐
│  Agent  │────▶│  MCP Protocol (统一接口)    │
└─────────┘     └─────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ MCP 服务 │   │ MCP 服务 │   │ MCP 服务 │
   │ (文件系统)│   │ (数据库) │   │ (API)    │
   └──────────┘   └──────────┘   └──────────┘
```

## MCP 核心概念

### 1. 三层架构

```typescript
// MCP 的三个核心原语

interface MCPPrimitives {
  // 1. Tools - 可执行的函数
  tools: {
    name: string;
    description: string;
    inputSchema: JSONSchema;
    handler: (input: unknown) => Promise<unknown>;
  }[];

  // 2. Resources - 可读取的数据源
  resources: {
    uri: string;           // 如 "file:///path" 或 "db://table"
    name: string;
    mimeType: string;
    read: () => Promise<string | Buffer>;
  }[];

  // 3. Prompts - 可复用的提示模板
  prompts: {
    name: string;
    description: string;
    arguments: { name: string; required: boolean }[];
    generate: (args: Record<string, string>) => Promise<string>;
  }[];
}
```

### 2. 传输层

MCP 支持多种传输方式：

```typescript
// stdio 传输 - 适合本地进程
class StdioTransport implements Transport {
  constructor(
    private command: string,
    private args: string[]
  ) {}

  async start(): Promise<void> {
    this.process = spawn(this.command, this.args);
    // 通过 stdin/stdout 通信
  }
}

// HTTP SSE 传输 - 适合远程服务
class SSETransport implements Transport {
  constructor(private url: string) {}

  async start(): Promise<void> {
    this.eventSource = new EventSource(this.url);
    // 通过 SSE 接收消息，POST 发送消息
  }
}
```

## 实战：实现 MCP Client

### 基础 Client 实现

```typescript
// learn-claude-code 风格的简化实现
class MCPClient {
  private transport: Transport;
  private requestId = 0;
  private pendingRequests = new Map<number, {
    resolve: (value: unknown) => void;
    reject: (error: Error) => void;
  }>();

  async connect(serverConfig: MCPServerConfig): Promise<void> {
    // 1. 建立传输层
    if (serverConfig.command) {
      this.transport = new StdioTransport(
        serverConfig.command,
        serverConfig.args || []
      );
    } else if (serverConfig.url) {
      this.transport = new SSETransport(serverConfig.url);
    }

    await this.transport.start();

    // 2. 握手：发送 initialize 请求
    const initResult = await this.request('initialize', {
      protocolVersion: '2024-11-05',
      capabilities: {
        roots: { listChanged: true },
      },
      clientInfo: {
        name: 'my-agent',
        version: '1.0.0'
      }
    });

    // 3. 确认初始化
    await this.notify('initialized', {});

    console.log('MCP 连接成功:', initResult.serverInfo);
  }

  async request(method: string, params: unknown): Promise<unknown> {
    const id = ++this.requestId;

    return new Promise((resolve, reject) => {
      this.pendingRequests.set(id, { resolve, reject });

      this.transport.send({
        jsonrpc: '2.0',
        id,
        method,
        params
      });
    });
  }

  async notify(method: string, params: unknown): Promise<void> {
    this.transport.send({
      jsonrpc: '2.0',
      method,
      params
    });
  }
}
```

### 发现和调用工具

```typescript
class MCPClient {
  // ... 上面的代码

  async listTools(): Promise<MCPTool[]> {
    const result = await this.request('tools/list', {});
    return result.tools;
  }

  async callTool(name: string, args: unknown): Promise<MCPToolResult> {
    const result = await this.request('tools/call', {
      name,
      arguments: args
    });

    // MCP 工具结果格式
    // { content: [{ type: 'text', text: '...' }], isError?: boolean }
    return result;
  }

  async listResources(): Promise<MCPResource[]> {
    const result = await this.request('resources/list', {});
    return result.resources;
  }

  async readResource(uri: string): Promise<string> {
    const result = await this.request('resources/read', { uri });
    return result.contents[0].text;
  }
}
```

## OpenClaw 的 MCP 集成

OpenClaw 通过 `mcporter` 技能集成 MCP：

```typescript
// OpenClaw 的 MCP 配置
interface MCPConfig {
  servers: {
    [name: string]: {
      command?: string;        // stdio 模式
      args?: string[];
      url?: string;            // HTTP 模式
      env?: Record<string, string>;
      disabled?: boolean;
    };
  };
}

// 配置示例
const mcpConfig: MCPConfig = {
  servers: {
    'filesystem': {
      command: 'npx',
      args: ['-y', '@anthropic/mcp-server-filesystem', '/path/to/allowed/dir'],
    },
    'github': {
      command: 'npx',
      args: ['-y', '@anthropic/mcp-server-github'],
      env: {
        GITHUB_TOKEN: process.env.GITHUB_TOKEN
      }
    },
    'postgres': {
      command: 'npx',
      args: ['-y', '@anthropic/mcp-server-postgres', process.env.DATABASE_URL],
    }
  }
};
```

### 工具发现与合并

```typescript
// Agent 启动时发现所有 MCP 工具
async function discoverMCPTools(config: MCPConfig): Promise<Tool[]> {
  const allTools: Tool[] = [];

  for (const [serverName, serverConfig] of Object.entries(config.servers)) {
    if (serverConfig.disabled) continue;

    const client = new MCPClient();
    await client.connect(serverConfig);

    const mcpTools = await client.listTools();

    // 将 MCP 工具转换为 Agent 工具格式
    for (const tool of mcpTools) {
      allTools.push({
        name: `${serverName}__${tool.name}`,  // 命名空间隔离
        description: tool.description,
        parameters: tool.inputSchema,
        handler: async (args) => {
          const result = await client.callTool(tool.name, args);
          return formatMCPResult(result);
        }
      });
    }
  }

  return allTools;
}
```

## 实战：实现 MCP Server

### 文件系统 Server 示例

```typescript
import { Server } from '@modelcontextprotocol/sdk/server';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio';
import * as fs from 'fs/promises';
import * as path from 'path';

const server = new Server(
  { name: 'filesystem-server', version: '1.0.0' },
  { capabilities: { tools: {}, resources: {} } }
);

// 定义工具
server.setRequestHandler('tools/list', async () => ({
  tools: [
    {
      name: 'read_file',
      description: '读取文件内容',
      inputSchema: {
        type: 'object',
        properties: {
          path: { type: 'string', description: '文件路径' }
        },
        required: ['path']
      }
    },
    {
      name: 'write_file',
      description: '写入文件',
      inputSchema: {
        type: 'object',
        properties: {
          path: { type: 'string' },
          content: { type: 'string' }
        },
        required: ['path', 'content']
      }
    },
    {
      name: 'list_directory',
      description: '列出目录内容',
      inputSchema: {
        type: 'object',
        properties: {
          path: { type: 'string' }
        },
        required: ['path']
      }
    }
  ]
}));

// 实现工具调用
server.setRequestHandler('tools/call', async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case 'read_file': {
        const content = await fs.readFile(args.path, 'utf-8');
        return {
          content: [{ type: 'text', text: content }]
        };
      }

      case 'write_file': {
        await fs.writeFile(args.path, args.content);
        return {
          content: [{ type: 'text', text: `已写入 ${args.path}` }]
        };
      }

      case 'list_directory': {
        const entries = await fs.readdir(args.path, { withFileTypes: true });
        const list = entries.map(e => 
          `${e.isDirectory() ? '📁' : '📄'} ${e.name}`
        ).join('\n');
        return {
          content: [{ type: 'text', text: list }]
        };
      }

      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error) {
    return {
      content: [{ type: 'text', text: `Error: ${error.message}` }],
      isError: true
    };
  }
});

// 定义资源
server.setRequestHandler('resources/list', async () => ({
  resources: [
    {
      uri: 'file:///etc/hosts',
      name: 'Hosts 文件',
      mimeType: 'text/plain'
    }
  ]
}));

// 启动服务器
const transport = new StdioServerTransport();
await server.connect(transport);
```

## MCP 与传统工具集成的对比

```typescript
// 传统方式：每个工具都要写完整的集成代码
const tools = {
  github_create_issue: {
    handler: async (args) => {
      const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });
      return await octokit.issues.create({
        owner: args.owner,
        repo: args.repo,
        title: args.title,
        body: args.body
      });
    }
  },
  // 每个工具都要重新写...
};

// MCP 方式：统一接口，工具由 MCP Server 提供
const mcpTools = await discoverMCPTools({
  servers: {
    github: {
      command: 'npx',
      args: ['-y', '@anthropic/mcp-server-github']
    }
  }
});

// 自动获得所有 GitHub 相关工具：
// - github__create_issue
// - github__list_issues
// - github__create_pr
// - github__list_commits
// ... 等等
```

## MCP 安全考虑

### 1. Server 隔离

```typescript
// 每个 MCP Server 在独立进程中运行
class MCPServerManager {
  private servers = new Map<string, ChildProcess>();

  async startServer(name: string, config: ServerConfig): Promise<void> {
    const proc = spawn(config.command, config.args, {
      env: {
        ...process.env,
        ...config.env
      },
      // 限制权限
      cwd: config.workdir,
      stdio: ['pipe', 'pipe', 'pipe']
    });

    this.servers.set(name, proc);
  }

  async stopServer(name: string): Promise<void> {
    const proc = this.servers.get(name);
    if (proc) {
      proc.kill('SIGTERM');
      this.servers.delete(name);
    }
  }
}
```

### 2. 权限控制

```typescript
// 在 Agent 层面控制哪些 MCP 工具可用
interface MCPPolicy {
  allowedServers: string[];
  deniedTools: string[];  // 黑名单
  requireConfirmation: string[];  // 需要人工确认的工具
}

function filterMCPTools(
  tools: Tool[],
  policy: MCPPolicy
): Tool[] {
  return tools.filter(tool => {
    const [server, toolName] = tool.name.split('__');

    // 检查服务器白名单
    if (!policy.allowedServers.includes(server)) {
      return false;
    }

    // 检查工具黑名单
    if (policy.deniedTools.includes(tool.name)) {
      return false;
    }

    return true;
  });
}
```

## 实战练习

### 练习 1：实现 MCP 资源订阅

```typescript
// MCP 支持资源变更通知
class MCPClient {
  async subscribeToResource(uri: string): Promise<void> {
    await this.request('resources/subscribe', { uri });
  }

  handleNotification(method: string, params: unknown): void {
    if (method === 'notifications/resources/updated') {
      // 资源已更新，重新读取
      this.emit('resourceUpdated', params.uri);
    }
  }
}
```

### 练习 2：实现带重试的 MCP 调用

```typescript
async function callMCPToolWithRetry(
  client: MCPClient,
  toolName: string,
  args: unknown,
  maxRetries = 3
): Promise<unknown> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await client.callTool(toolName, args);
    } catch (error) {
      if (attempt === maxRetries) throw error;

      // 检查是否值得重试
      if (error.code === -32600) {  // Invalid Request
        throw error;  // 不重试
      }

      // 指数退避
      await sleep(Math.pow(2, attempt) * 1000);
    }
  }
}
```

## 核心要点总结

1. **MCP 解决了工具集成碎片化问题**
   - 统一的协议，不同工具只需实现一次
   - 工具自动发现，无需硬编码

2. **三层原语：Tools、Resources、Prompts**
   - Tools 是可执行的函数
   - Resources 是可读取的数据源
   - Prompts 是可复用的模板

3. **传输层抽象**
   - stdio 适合本地进程
   - HTTP SSE 适合远程服务
   - 未来可能支持 WebSocket

4. **安全性通过隔离实现**
   - 每个 Server 独立进程
   - Agent 层面的权限控制
   - 工具调用确认机制

## 下一步

- 尝试运行官方 MCP 服务器
- 为自己的服务编写 MCP Server
- 探索 MCP 的资源订阅和变更通知

---

> 💡 **思考题**：MCP 和 OpenAPI/REST API 有什么区别？为什么 Agent 需要 MCP 这样的协议？
>
> 提示：考虑工具发现、双向通信、资源订阅等能力。
