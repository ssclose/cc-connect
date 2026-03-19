# Cursor Agent 集成原理

本文档说明 cc-connect 如何与 [Cursor](https://cursor.com) 集成 —— 架构原理、通信协议和配置方法。

## 概述

cc-connect 通过 **Cursor Agent CLI**（`agent` 命令行工具）来操控 Cursor，**而不是直接操控 Cursor IDE 编辑器**。`agent` CLI 是一个无界面的终端工具，提供与 Cursor 编辑器相同的 AI 编程能力，以子进程方式由 cc-connect 启动和管理。

```
用户（手机/聊天应用）
     │
     ▼
聊天平台（飞书 / Telegram / Slack / ...）
     │
     ▼
cc-connect 引擎（消息路由）
     │
     ▼
agent/cursor/（适配层）
     │  每轮对话启动子进程：
     │  agent --print --output-format stream-json --trust ...
     ▼
Cursor Agent CLI（`agent` 可执行文件）
     │
     │  stdout：逐行输出 JSON 事件流
     │
     ▼
cc-connect 引擎（解析事件 → 发送到聊天平台）
     │
     ▼
用户在聊天中看到回复
```

## 工作原理

### 1. 子进程通信

用户每发送一条消息，cc-connect 就启动一个 `agent` CLI **子进程**，通过以下命令行参数实现程序化通信：

```bash
agent \
  --print \                          # 启用非交互式输出
  --output-format stream-json \      # 以逐行 JSON 格式输出
  --trust \                          # 信任工作区
  [--force] \                        # 自动批准工具调用（mode=force）
  [--mode plan|ask] \                # 规划模式或问答模式
  [--resume <session_id>] \          # 恢复之前的对话
  [--model <model_id>] \             # 指定 AI 模型
  --workspace /path/to/project \     # 项目目录
  -- "<用户消息>"                     # 用户的提问
```

### 2. JSON 事件流

`agent` CLI 向 stdout 写入结构化 JSON 事件，每行一个事件。cc-connect 实时读取这些事件并转换为聊天消息：

| 事件类型 | 说明 | 示例 |
|---------|------|------|
| `system` | 会话初始化 | `{"type":"system","session_id":"abc123","model":"claude-sonnet-4"}` |
| `thinking` | 扩展推理 | `{"type":"thinking","subtype":"delta","text":"让我分析一下..."}` |
| `assistant` | 回复文本 | `{"type":"assistant","message":{"content":[{"type":"text","text":"结果是..."}]}}` |
| `tool_call` | 工具调用 | `{"type":"tool_call","subtype":"started","tool_call":{"shellToolCall":{...}}}` |
| `interaction_query` | 权限请求 | `{"type":"interaction_query","subtype":"request","query_type":"shellRequestQuery",...}` |
| `result` | 最终结果 | `{"type":"result","result":"完成。","session_id":"abc123"}` |

### 3. 会话连续性

多轮对话通过会话恢复机制实现：

1. 第一条消息 → `agent --print --output-format stream-json --trust -- "用户提问"`
2. agent 返回包含 `session_id` 的 `system` 事件
3. cc-connect 保存此 `session_id`
4. 后续消息 → `agent --print --output-format stream-json --trust --resume <session_id> -- "后续提问"`

agent CLI 在内部维护对话历史，因此每次恢复的会话都拥有完整上下文。

### 4. 工具调用处理

当 agent 调用工具（文件编辑、Shell 命令、Web 请求等）时，cc-connect 提取工具信息并在聊天中展示：

- **Bash** — Shell 命令（`shellToolCall`）
- **Read** — 文件读取（`readToolCall`）
- **Edit** — 文件编辑（`editToolCall`）
- **Write** — 文件写入（`writeToolCall`）
- **Grep** — 模式搜索（`grepToolCall`）
- **Glob** — 文件匹配（`globToolCall`）
- **WebFetch** — Web 请求（`webFetchToolCall`）

### 5. 权限模式

Cursor Agent CLI 支持不同的信任级别，通过命令行参数控制：

| 模式 | CLI 参数 | 行为 |
|------|---------|------|
| `default` | `--trust` | 信任工作区，工具调用前询问 |
| `force` | `--trust --force` | 自动批准所有工具调用 |
| `plan` | `--trust --mode plan` | 只读分析，不做修改 |
| `ask` | `--trust --mode ask` | 问答风格，只读 |

在聊天中通过 `/mode` 命令可以实时切换模式。

### 6. 会话存储

Cursor 将聊天会话存储在本地：

```
~/.cursor/chats/<工作目录的MD5哈希>/
```

每个会话目录包含一个 `store.db` SQLite 数据库，其中存储对话元数据和消息。cc-connect 读取此数据库来实现 `/list`、`/switch` 等会话管理命令。

---

## 前提条件

1. **Cursor Agent CLI**（`agent`）必须已安装并在 `$PATH` 中可用。

   安装地址：https://docs.cursor.com/agent

2. 验证安装：

   ```bash
   agent --version
   agent models      # 列出可用模型
   ```

---

## 配置

### 基本配置

```toml
[[projects]]
name = "my-project"

[projects.agent]
type = "cursor"

[projects.agent.options]
work_dir = "/path/to/project"
```

### 完整配置

```toml
[[projects]]
name = "my-project"

[projects.agent]
type = "cursor"

[projects.agent.options]
work_dir = "/path/to/project"
mode = "force"                       # "default" | "force" | "plan" | "ask"
cmd = "agent"                        # CLI 二进制名称（默认："agent"）
model = "claude-sonnet-4-20250514"   # 可选：指定模型

# 可选：API 提供商管理
[[projects.agent.providers]]
name = "anthropic"
api_key = "sk-ant-xxx"

[[projects.platforms]]
type = "feishu"   # 或 telegram, slack, discord 等

[projects.platforms.options]
# ... 平台相关配置
```

### 配置选项

| 选项 | 类型 | 默认值 | 说明 |
|------|------|-------|------|
| `work_dir` | string | `"."` | 项目工作目录 |
| `mode` | string | `"default"` | 权限模式 |
| `cmd` | string | `"agent"` | CLI 二进制名称 |
| `model` | string | *（自动）* | 使用的 AI 模型 |

---

## 聊天命令

连接后，在聊天应用中可使用以下命令：

| 命令 | 说明 |
|------|------|
| `/mode` | 查看可用权限模式 |
| `/mode force` | 切换到强制模式（自动批准所有操作） |
| `/model` | 列出可用模型 |
| `/model <名称>` | 切换到指定模型 |
| `/list` | 列出所有 Cursor Agent 会话 |
| `/switch <id>` | 切换到其他会话 |
| `/new [名称]` | 创建新会话 |
| `/current` | 显示当前会话信息 |
| `/stop` | 停止当前执行 |

---

## 架构细节

### 代码结构

```
agent/cursor/
├── cursor.go              # Agent 接口：模式、模型、提供商、会话列表
└── session.go             # 会话：子进程管理、JSON 事件解析
```

### 实现的接口

Cursor agent 实现了以下核心接口：

| 接口 | 用途 |
|------|------|
| `Agent` | 基本代理契约（Name, StartSession, ListSessions, Stop） |
| `WorkDirSwitcher` | 运行时切换工作目录 |
| `ModelSwitcher` | 运行时切换模型 |
| `ModeSwitcher` | 切换权限模式 |
| `ProviderSwitcher` | API 提供商管理 |
| `SessionDeleter` | 删除会话 |
| `SessionEnvInjector` | 为会话注入环境变量 |
| `SkillProvider` | 自定义技能发现 |
| `AgentDoctorInfo` | CLI 诊断信息 |

### 为什么是 CLI 而不是 IDE？

cc-connect 集成的是 **Cursor Agent CLI** 而非 Cursor IDE，原因如下：

1. **无界面运行** — CLI 不需要图形界面，适合服务器或守护进程部署
2. **程序化控制** — JSON 流式输出便于精确解析事件
3. **会话管理** — `--resume` 支持有状态的多轮对话
4. **进程隔离** — 每轮对话运行在独立进程中
5. **跨平台** — 只需 Node.js 即可运行，无需显示服务器

---

## 常见问题

### 找不到 "agent" CLI

```
cursor: "agent" CLI not found in PATH
```

从 https://docs.cursor.com/agent 安装 Cursor Agent CLI，并确保 `agent` 二进制文件在 `$PATH` 中。

### 模型列表为空

如果 `/model` 只显示备用模型，请验证 CLI 能否独立运行：

```bash
agent models
```

### 会话列表为空

会话按工作目录存储。确保配置中的 `work_dir` 与使用 agent 聊天时的目录一致。会话存储在 `~/.cursor/chats/<work_dir的md5值>/`。
