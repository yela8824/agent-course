# 45 - Agent Sandbox 代码执行：让 Agent 安全运行代码

> "Agent 能生成代码，但能不能运行代码，是两回事。" — 每个踩过沙箱坑的工程师

## 为什么 Agent 需要沙箱？

LLM 很快就能生成代码，但"能生成"≠"能安全运行"。

**不加沙箱直接运行 Agent 生成的代码，后果可以是：**
- `rm -rf /`（删库）
- 泄露环境变量里的 API Key
- 建立反弹 Shell
- 无限循环耗尽资源

所以 **代码执行必须在隔离环境中进行**，这就是 Sandbox（沙箱）。

---

## 沙箱的几个维度

| 维度 | 问题 | 沙箱解法 |
|------|------|---------|
| **文件系统** | 读写任意文件 | chroot / overlay FS / 只读挂载 |
| **网络访问** | 外联、数据外泄 | 网络命名空间 / 防火墙规则 |
| **进程权限** | 提权、fork bomb | seccomp / capabilities 限制 |
| **执行时间** | 死循环 | 超时 + SIGKILL |
| **资源占用** | 内存/CPU 耗尽 | cgroup 限制 |

---

## 方案对比

### 1. 进程级沙箱（最简单）

直接 `subprocess` 加超时和资源限制：

```python
# learn-claude-code 风格的最简实现
import subprocess
import resource

def run_sandboxed(code: str, timeout: int = 5) -> dict:
    """最简单的沙箱：超时 + 独立进程"""
    try:
        result = subprocess.run(
            ["python3", "-c", code],
            capture_output=True,
            text=True,
            timeout=timeout,
            # 禁止写文件（只读工作目录）
            cwd="/tmp/sandbox",
        )
        return {
            "stdout": result.stdout,
            "stderr": result.stderr,
            "returncode": result.returncode,
        }
    except subprocess.TimeoutExpired:
        return {"error": "timeout", "stdout": "", "stderr": ""}
```

**缺点**：没有网络隔离，没有文件系统保护，只防无限循环。

---

### 2. Docker 容器沙箱（实用方案）

```python
import docker

def run_in_docker(code: str, timeout: int = 10) -> dict:
    """Docker 沙箱：完整隔离"""
    client = docker.from_env()
    
    try:
        container = client.containers.run(
            image="python:3.12-slim",
            command=["python3", "-c", code],
            # 🔒 关键限制
            network_disabled=True,          # 断网
            read_only=True,                 # 只读文件系统
            mem_limit="128m",               # 内存上限
            cpu_quota=50000,                # CPU 50%
            pids_limit=64,                  # 限制进程数（防 fork bomb）
            security_opt=["no-new-privileges"],  # 禁止提权
            # 允许写 /tmp
            tmpfs={"/tmp": "size=64m"},
            remove=True,                    # 运行完自动删除
            detach=True,
        )
        
        result = container.wait(timeout=timeout)
        logs = container.logs(stdout=True, stderr=True).decode()
        return {"output": logs, "exitcode": result["StatusCode"]}
        
    except Exception as e:
        return {"error": str(e)}
```

---

### 3. E2B 云沙箱（最省事）

[E2B](https://e2b.dev) 是专门为 AI 代码执行设计的云沙箱服务：

```python
from e2b_code_interpreter import Sandbox

async def run_with_e2b(code: str) -> dict:
    """E2B 云沙箱：开箱即用，支持持久化状态"""
    async with Sandbox() as sbx:
        # 安装依赖
        await sbx.commands.run("pip install pandas numpy")
        
        # 执行代码，保持状态（变量跨次调用保留！）
        result = await sbx.run_code(code)
        
        return {
            "results": [r.text for r in result.results],
            "stdout": result.logs.stdout,
            "stderr": result.logs.stderr,
            "error": result.error,
        }

# 多轮对话中保持沙箱状态
async def multi_turn_execution():
    async with Sandbox() as sbx:
        # 第一轮：定义变量
        await sbx.run_code("df = pd.DataFrame({'a': [1,2,3]})")
        # 第二轮：使用变量（状态保留！）
        result = await sbx.run_code("print(df.describe())")
```

---

### 4. WebAssembly 沙箱（前端友好）

Deno/Pyodide 可以在 WASM 里跑代码，天然隔离：

```typescript
// 用 Deno 运行用户代码（自带沙箱）
import { run } from "jsr:@deno/sandbox";

const result = await run({
  code: userCode,
  permissions: {
    read: ["/tmp"],   // 只允许读 /tmp
    net: false,       // 禁止网络
    env: false,       // 禁止读环境变量
  },
  timeout: 5000,
});
```

---

## OpenClaw 的做法：工具级安全策略

OpenClaw 把安全控制嵌入工具调用层，不是在代码执行层：

```typescript
// OpenClaw exec 工具的安全参数
{
  "command": "python3 script.py",
  "security": "allowlist",     // deny | allowlist | full
  "elevated": false,           // 不要 sudo
  "timeout": 30,               // 超时秒数
  "workdir": "/tmp/sandbox"    // 限制工作目录
}
```

**security 模式说明：**
- `deny`：完全禁止执行（安全优先）
- `allowlist`：只允许白名单命令（平衡模式）
- `full`：完全权限（需要 elevated=true 且用户授权）

OpenClaw 还有 **ask 模式**：
```typescript
{
  "command": "rm -rf /tmp/cache",
  "ask": "on-miss"   // 如果命令不在信任列表，先问用户
}
```

这是"人在环路"（Human-in-the-Loop）的自然体现。

---

## pi-mono 的沙箱设计

pi-mono 采用**工具权限声明**而非运行时拦截：

```typescript
// pi-mono: 工具定义时声明权限需求
const codeRunnerTool: Tool = {
  name: "run_python",
  description: "在沙箱中运行 Python 代码",
  inputSchema: {
    type: "object",
    properties: {
      code: { type: "string" },
      timeout_ms: { type: "number", default: 5000 },
    },
  },
  // 权限声明
  permissions: {
    network: false,
    filesystem: "readonly",
    env: [],  // 不暴露任何环境变量
  },
  execute: async ({ code, timeout_ms }) => {
    return await runInSandbox(code, timeout_ms);
  },
};
```

---

## 实战：给 Agent 加代码执行能力

```python
# 完整示例：Agent + 沙箱代码执行
from anthropic import Anthropic
import subprocess, json

client = Anthropic()

# 定义代码执行工具
tools = [{
    "name": "run_python",
    "description": "在安全沙箱中执行 Python 代码，返回输出结果",
    "input_schema": {
        "type": "object",
        "properties": {
            "code": {
                "type": "string",
                "description": "要执行的 Python 代码"
            }
        },
        "required": ["code"]
    }
}]

def execute_code(code: str) -> str:
    """带超时和资源限制的沙箱执行"""
    try:
        result = subprocess.run(
            ["python3", "-c", code],
            capture_output=True, text=True,
            timeout=10,
            cwd="/tmp",
            env={"PATH": "/usr/bin:/bin"},  # ⚠️ 不传递父进程环境变量！
        )
        output = result.stdout or result.stderr
        return output[:2000]  # 限制输出长度
    except subprocess.TimeoutExpired:
        return "Error: 执行超时（10秒）"
    except Exception as e:
        return f"Error: {e}"

def agent_with_code_execution(user_request: str):
    messages = [{"role": "user", "content": user_request}]
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )
        
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason == "end_turn":
            # 提取最终文本回复
            for block in response.content:
                if hasattr(block, "text"):
                    print(block.text)
            break
        
        # 处理工具调用
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"🔧 执行代码:\n{block.input['code']}\n")
                output = execute_code(block.input["code"])
                print(f"📤 输出:\n{output}\n")
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        
        messages.append({"role": "user", "content": tool_results})

# 使用
agent_with_code_execution("帮我计算斐波那契数列第 50 项，并画出前 20 项的折线图数据")
```

---

## 沙箱的常见陷阱

### ❌ 陷阱 1：传递父进程环境变量

```python
# ❌ 危险：API Key 泄露
subprocess.run(["python3", "-c", code])  # 默认继承所有 env

# ✅ 安全：只传必要变量
subprocess.run(["python3", "-c", code], env={"PATH": "/usr/bin:/bin"})
```

### ❌ 陷阱 2：忘记限制输出大小

```python
# ❌ 危险：Agent 生成 print("A" * 1000000000)，内存爆炸
output = result.stdout

# ✅ 安全：截断输出
output = result.stdout[:10000]
```

### ❌ 陷阱 3：沙箱逃逸（容器特权模式）

```bash
# ❌ 危险：--privileged 模式下可以挂载宿主机磁盘
docker run --privileged ...

# ✅ 安全：最小权限
docker run --read-only --network=none --cap-drop=ALL ...
```

### ❌ 陷阱 4：多轮状态污染

```python
# ❌ 陷阱：用户代码在上一轮修改了全局变量/文件
# 下一轮执行受到污染

# ✅ 方案 A：每次执行创建新沙箱（无状态）
# ✅ 方案 B：显式隔离状态（E2B 的 Sandbox 生命周期管理）
# ✅ 方案 C：快照 + 恢复（VM 方案）
```

---

## 选型建议

| 场景 | 推荐方案 |
|------|---------|
| 快速原型 / 单机 | 进程级沙箱 + 超时 |
| 生产环境 / 自托管 | Docker + seccomp + 网络隔离 |
| SaaS / 多租户 | E2B 或 Modal |
| 前端代码执行 | Pyodide（WASM）|
| 需要保持状态 | E2B Sandbox（跨轮次保活）|
| 极高安全要求 | gVisor / Firecracker MicroVM |

---

## 总结

```
Agent 代码执行 = LLM 生成 + 沙箱运行 + 结果反馈
             ↑                  ↑
          任何代码          必须隔离！
```

**核心原则：**
1. **永不信任 LLM 生成的代码** — 哪怕是你自己的 Agent
2. **最小权限** — 只给沙箱它需要的权限
3. **超时必加** — 防止死循环耗尽资源
4. **环境变量隔离** — 不暴露 API Key 等敏感信息
5. **输出截断** — 防止内存爆炸
6. **无状态优先** — 每次执行用新沙箱，除非明确需要状态

沙箱代码执行是 Agent 从"会聊天"升级为"会干活"的关键能力，做好安全是基本功。

---

*下一课预告：Agent 知识图谱与实体记忆 — 让 Agent 拥有结构化世界观*
