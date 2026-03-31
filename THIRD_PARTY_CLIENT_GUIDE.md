# Claude Code 第三方客户端开发指南

基于 v2.1.88 源码逆向分析，提供完整的 API 调用和认证实现方案。

---

## 目录

1. [两种认证方式](#1-两种认证方式)
2. [方式一：API Key 直接调用](#2-方式一api-key-直接调用)
3. [方式二：OAuth 登录（Claude Max/Pro 订阅）](#3-方式二oauth-登录claude-maxpro-订阅)
4. [Messages API 完整请求格式](#4-messages-api-完整请求格式)
5. [流式响应处理](#5-流式响应处理)
6. [工具调用循环](#6-工具调用循环)
7. [系统提示词构建](#7-系统提示词构建)
8. [完整代码示例](#8-完整代码示例)
9. [辅助 API 端点](#9-辅助-api-端点)
10. [注意事项](#10-注意事项)

---

## 1. 两种认证方式

| 方式 | 适用场景 | 计费 |
|------|---------|------|
| **API Key** | 开发者，按量付费 | Console 账单 |
| **OAuth 登录** | Claude Max/Pro/Team 订阅用户 | 包含在订阅中 |

---

## 2. 方式一：API Key 直接调用

最简单的方式，只需要一个 API Key。

### 2.1 获取 API Key

从 https://console.anthropic.com/settings/keys 创建。

### 2.2 调用方式

```javascript
const response = await fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'sk-ant-xxxxx',            // 你的 API Key
    'anthropic-version': '2023-06-01',       // API 版本
    'anthropic-beta': 'interleaved-thinking-2025-05-14', // 启用思考
    'x-app': 'cli',                          // Claude Code 标识
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 16384,
    messages: [
      { role: 'user', content: 'Hello, Claude!' }
    ],
    stream: true
  })
})
```

---

## 3. 方式二：OAuth 登录（Claude Max/Pro 订阅）

这是 Claude Code 实际使用的方式，通过 OAuth PKCE 流程获取 Access Token。

### 3.1 关键参数

```
CLIENT_ID:          9d1c250a-e61b-44d9-88ed-5944d1962f5e
AUTHORIZE_URL:      https://claude.com/cai/oauth/authorize
TOKEN_URL:          https://platform.claude.com/v1/oauth/token
API_BASE_URL:       https://api.anthropic.com
PROFILE_URL:        https://api.anthropic.com/api/oauth/profile
CALLBACK_PATH:      /callback
```

### 3.2 OAuth Scopes

```
user:profile                    — 读取用户资料
user:inference                  — 使用 Claude 推理
user:sessions:claude_code       — Claude Code 会话管理
user:mcp_servers                — MCP 服务器访问
user:file_upload                — 文件上传
```

### 3.3 完整 OAuth PKCE 流程

#### Step 1: 生成 PKCE 参数

```javascript
import crypto from 'crypto'

// Code Verifier: 随机 32 字节 → base64url
const codeVerifier = crypto.randomBytes(32)
  .toString('base64url')

// Code Challenge: SHA256(verifier) → base64url
const codeChallenge = crypto.createHash('sha256')
  .update(codeVerifier)
  .digest('base64url')

// State: 随机值防 CSRF
const state = crypto.randomBytes(32).toString('base64url')
```

#### Step 2: 启动本地回调服务器

```javascript
import http from 'http'

function startCallbackServer() {
  return new Promise((resolve, reject) => {
    const server = http.createServer((req, res) => {
      const url = new URL(req.url, `http://localhost`)
      if (url.pathname === '/callback') {
        const code = url.searchParams.get('code')
        const returnedState = url.searchParams.get('state')

        res.writeHead(302, {
          Location: 'https://platform.claude.com/oauth/code/success?app=claude-code'
        })
        res.end()
        server.close()
        resolve({ code, state: returnedState })
      }
    })

    // 让 OS 分配随机端口
    server.listen(0, '127.0.0.1', () => {
      const port = server.address().port
      console.log(`Callback server on port ${port}`)
      resolve({ server, port })
    })
  })
}
```

#### Step 3: 打开浏览器授权

```javascript
const port = callbackServer.port
const scopes = [
  'user:profile',
  'user:inference',
  'user:sessions:claude_code',
  'user:mcp_servers',
  'user:file_upload'
].join(' ')

const authUrl = new URL('https://claude.com/cai/oauth/authorize')
authUrl.searchParams.set('client_id', '9d1c250a-e61b-44d9-88ed-5944d1962f5e')
authUrl.searchParams.set('response_type', 'code')
authUrl.searchParams.set('redirect_uri', `http://localhost:${port}/callback`)
authUrl.searchParams.set('scope', scopes)
authUrl.searchParams.set('code_challenge', codeChallenge)
authUrl.searchParams.set('code_challenge_method', 'S256')
authUrl.searchParams.set('state', state)
authUrl.searchParams.set('code', 'true')  // 显示 Claude Max 升级提示

// 打开浏览器
import { exec } from 'child_process'
exec(`start "${authUrl.toString()}"`)  // Windows
// exec(`open "${authUrl.toString()}"`)  // macOS
// exec(`xdg-open "${authUrl.toString()}"`)  // Linux
```

#### Step 4: 用 Authorization Code 换取 Token

```javascript
async function exchangeToken(code, port, codeVerifier, state) {
  const response = await fetch('https://platform.claude.com/v1/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'authorization_code',
      code: code,
      redirect_uri: `http://localhost:${port}/callback`,
      client_id: '9d1c250a-e61b-44d9-88ed-5944d1962f5e',
      code_verifier: codeVerifier,
      state: state
    })
  })

  return await response.json()
  // 返回:
  // {
  //   access_token: "cact-xxx...",
  //   refresh_token: "cart-xxx...",
  //   expires_in: 3600,        // 秒
  //   scope: "user:profile user:inference ...",
  //   account: { uuid: "...", email_address: "..." },
  //   organization: { uuid: "..." }
  // }
}
```

#### Step 5: Token 刷新

Access Token 通常 1 小时过期，需要用 Refresh Token 刷新：

```javascript
async function refreshToken(refreshToken) {
  const response = await fetch('https://platform.claude.com/v1/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: '9d1c250a-e61b-44d9-88ed-5944d1962f5e',
      scope: 'user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload'
    })
  })

  return await response.json()
  // 返回新的 access_token 和 refresh_token
}
```

#### Step 6: Token 过期检测

```javascript
function isTokenExpired(expiresAt) {
  const BUFFER_MS = 5 * 60 * 1000  // 提前 5 分钟刷新
  return (Date.now() + BUFFER_MS) >= expiresAt
}
```

### 3.4 Token 存储

Claude Code 的 Token 存储位置：

| 平台 | 存储方式 | 位置 |
|------|---------|------|
| macOS | Keychain | 服务名: `dev.anthropic.claude-code.oauth` |
| Linux/Windows | 明文文件 | `~/.claude/.credentials.json` |

`~/.claude/.credentials.json` 格式：
```json
{
  "claudeAiOauth": {
    "accessToken": "cact-xxx...",
    "refreshToken": "cart-xxx...",
    "expiresAt": 1711900000000,
    "scopes": ["user:profile", "user:inference", "..."],
    "subscriptionType": "max",
    "rateLimitTier": "tier4"
  }
}
```

### 3.5 使用 OAuth Token 调用 API

```javascript
const response = await fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${accessToken}`,  // OAuth Token
    'anthropic-version': '2023-06-01',
    'anthropic-beta': 'oauth-2025-04-20,interleaved-thinking-2025-05-14',
    'x-app': 'cli',
  },
  body: JSON.stringify({ /* ... */ })
})
```

---

## 4. Messages API 完整请求格式

以下是 Claude Code 实际发送的完整请求结构：

### 4.1 请求头

```http
POST /v1/messages HTTP/1.1
Host: api.anthropic.com
Content-Type: application/json

# 认证（二选一）
x-api-key: sk-ant-xxxxx                    # API Key 方式
Authorization: Bearer cact-xxxxx           # OAuth 方式

# 必需
anthropic-version: 2023-06-01

# Claude Code 标识
x-app: cli
User-Agent: claude-code/2.1.88
X-Claude-Code-Session-Id: {uuid}
x-client-request-id: {uuid}

# Beta 特性（按需启用）
anthropic-beta: oauth-2025-04-20,interleaved-thinking-2025-05-14,prompt-cache-control-ephemeral-2025-02-24
```

### 4.2 请求体

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 16384,
  "stream": true,

  "system": [
    {
      "type": "text",
      "text": "You are Claude Code, Anthropic's official CLI...",
      "cache_control": { "type": "ephemeral" }
    }
  ],

  "messages": [
    {
      "role": "user",
      "content": "Fix the bug in auth.ts"
    }
  ],

  "tools": [
    {
      "name": "Read",
      "description": "Reads a file from the local filesystem...",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": { "type": "string", "description": "The absolute path to the file to read" },
          "offset": { "type": "number", "description": "Line number to start reading from" },
          "limit": { "type": "number", "description": "Number of lines to read" }
        },
        "required": ["file_path"]
      }
    },
    {
      "name": "Edit",
      "description": "Performs exact string replacements in files...",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": { "type": "string" },
          "old_string": { "type": "string" },
          "new_string": { "type": "string" }
        },
        "required": ["file_path", "old_string", "new_string"]
      }
    },
    {
      "name": "Bash",
      "description": "Executes a given bash command...",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": { "type": "string" },
          "timeout": { "type": "number" },
          "description": { "type": "string" }
        },
        "required": ["command"]
      }
    }
  ],

  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },

  "metadata": {
    "user_id": "{\"device_id\":\"abc123...\",\"account_uuid\":\"uuid-xxx\",\"session_id\":\"uuid-yyy\"}"
  }
}
```

### 4.3 可用 Beta Header 列表

| Beta Header | 功能 |
|-------------|------|
| `oauth-2025-04-20` | OAuth 认证支持 |
| `interleaved-thinking-2025-05-14` | 交错思考（Extended Thinking） |
| `prompt-cache-control-ephemeral-2025-02-24` | 提示缓存 |
| `web-search-2025-03-05` | 网络搜索工具 |
| `context-1m-2025-08-07` | 1M 上下文窗口 |
| `context-management-2025-06-27` | API 侧上下文管理 |
| `structured-outputs-2025-12-15` | 结构化输出 |
| `effort-2025-11-24` | 推理努力程度控制 |
| `fast-mode-2026-02-01` | 快速模式 |
| `redact-thinking-2026-02-12` | 超时时清除思考内容 |
| `task-budgets-2026-03-13` | 任务预算控制 |

### 4.4 可用模型

```
claude-opus-4-20250918
claude-sonnet-4-20250514
claude-haiku-4-20250414

# 别名
opus   → claude-opus-4-20250918
sonnet → claude-sonnet-4-20250514
haiku  → claude-haiku-4-20250414
```

---

## 5. 流式响应处理

### 5.1 SSE 事件类型

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_xxx","role":"assistant","content":[],"model":"claude-sonnet-4-20250514","usage":{"input_tokens":1234,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"Let me analyze..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"I'll read the file..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: content_block_start
data: {"type":"content_block_start","index":2,"content_block":{"type":"tool_use","id":"toolu_xxx","name":"Read","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":2,"delta":{"type":"input_json_delta","partial_json":"{\"file_path\":\"/src/auth.ts\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":2}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use"},"usage":{"output_tokens":150}}

event: message_stop
data: {"type":"message_stop"}
```

### 5.2 流式解析代码

```javascript
async function* parseSSE(response) {
  const reader = response.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ''

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    buffer += decoder.decode(value, { stream: true })
    const lines = buffer.split('\n')
    buffer = lines.pop() // 保留最后一个不完整的行

    let eventType = null
    for (const line of lines) {
      if (line.startsWith('event: ')) {
        eventType = line.slice(7)
      } else if (line.startsWith('data: ') && eventType) {
        yield { event: eventType, data: JSON.parse(line.slice(6)) }
        eventType = null
      }
    }
  }
}
```

---

## 6. 工具调用循环

Claude Code 的核心是一个 **消息循环**：发送消息 → 收到 tool_use → 执行工具 → 把结果发回 → 继续，直到 `stop_reason` 不是 `tool_use`。

### 6.1 循环逻辑

```javascript
async function conversationLoop(systemPrompt, tools, userMessage) {
  const messages = [{ role: 'user', content: userMessage }]

  while (true) {
    // 1. 发送请求
    const response = await callClaudeAPI({
      system: systemPrompt,
      messages,
      tools,
      stream: true
    })

    // 2. 解析流式响应，收集完整的 assistant 消息
    const assistantMessage = await collectStreamResponse(response)
    messages.push({ role: 'assistant', content: assistantMessage.content })

    // 3. 检查是否需要执行工具
    if (assistantMessage.stop_reason !== 'tool_use') {
      // 对话结束，输出最终文本
      return assistantMessage
    }

    // 4. 执行所有工具调用
    const toolResults = []
    for (const block of assistantMessage.content) {
      if (block.type === 'tool_use') {
        const result = await executeTool(block.name, block.input)
        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: result
        })
      }
    }

    // 5. 把工具结果作为 user 消息发回
    messages.push({ role: 'user', content: toolResults })
  }
}
```

### 6.2 工具执行示例

```javascript
async function executeTool(name, input) {
  switch (name) {
    case 'Read': {
      const content = fs.readFileSync(input.file_path, 'utf-8')
      const lines = content.split('\n')
      const start = (input.offset || 1) - 1
      const end = input.limit ? start + input.limit : lines.length
      const numbered = lines.slice(start, end)
        .map((line, i) => `${start + i + 1}\t${line}`)
        .join('\n')
      return numbered
    }

    case 'Edit': {
      let content = fs.readFileSync(input.file_path, 'utf-8')
      if (!content.includes(input.old_string)) {
        return `Error: old_string not found in file`
      }
      content = content.replace(input.old_string, input.new_string)
      fs.writeFileSync(input.file_path, content)
      return `File edited successfully`
    }

    case 'Bash': {
      const { stdout, stderr } = await execAsync(input.command, {
        timeout: input.timeout || 120000,
        cwd: process.cwd()
      })
      return JSON.stringify({ stdout, stderr })
    }

    case 'Glob': {
      const files = glob.sync(input.pattern, { cwd: input.path || process.cwd() })
      return JSON.stringify({ filenames: files.slice(0, 100) })
    }

    case 'Grep': {
      const { stdout } = await execAsync(
        `rg ${input.pattern} ${input.path || '.'} --json`,
        { cwd: process.cwd() }
      )
      return stdout
    }

    default:
      return `Unknown tool: ${name}`
  }
}
```

### 6.3 并发执行

Claude Code 对工具调用有并发优化：

```javascript
// 只读工具可以并行
const readOnlyTools = ['Read', 'Glob', 'Grep']

async function executeToolsConcurrently(toolCalls) {
  const readOnly = toolCalls.filter(t => readOnlyTools.includes(t.name))
  const writeTools = toolCalls.filter(t => !readOnlyTools.includes(t.name))

  // 只读工具并行执行
  const readResults = await Promise.all(
    readOnly.map(t => executeTool(t.name, t.input).then(result => ({
      type: 'tool_result',
      tool_use_id: t.id,
      content: result
    })))
  )

  // 写工具串行执行
  const writeResults = []
  for (const t of writeTools) {
    const result = await executeTool(t.name, t.input)
    writeResults.push({ type: 'tool_result', tool_use_id: t.id, content: result })
  }

  return [...readResults, ...writeResults]
}
```

---

## 7. 系统提示词构建

### 7.1 系统提示词结构

Claude Code 的系统提示词分层组织：

```
┌──────────────────────────────────────┐
│  Attribution Header                  │  ← 版本/构建指纹
├──────────────────────────────────────┤
│  CLI Prefix                          │  ← 非交互模式说明
├──────────────────────────────────────┤
│  Static Sections (可全局缓存)         │
│  ├─ Role Introduction                │  ← "You are Claude Code..."
│  ├─ System Instructions             │  ← 工具权限、系统提醒
│  ├─ Doing Tasks                      │  ← 代码风格、安全规范
│  ├─ Executing Actions               │  ← 操作安全指导
│  ├─ Using your Tools                 │  ← 工具使用指导
│  ├─ Tone and Style                   │  ← 输出风格
│  └─ Output Efficiency               │  ← 简洁性要求
├──────── CACHE BOUNDARY ─────────────┤
│  Dynamic Sections (用户/会话特定)     │
│  ├─ Memory (CLAUDE.md)              │  ← 项目上下文
│  ├─ Environment Info                │  ← 工作目录、平台、模型
│  ├─ Language Preference             │  ← 语言偏好
│  ├─ Output Style Config             │  ← 自定义输出样式
│  ├─ MCP Instructions                │  ← MCP 服务器工具说明
│  └─ Scratchpad                      │  ← 临时目录说明
└──────────────────────────────────────┘
```

### 7.2 简化版系统提示词

对于第三方客户端，可以使用简化版：

```javascript
const systemPrompt = `You are Claude Code, an AI coding assistant.

# Environment
- Working directory: ${process.cwd()}
- Platform: ${process.platform}
- Date: ${new Date().toISOString().split('T')[0]}

# Instructions
- Read files before editing them
- Use the tools provided to accomplish tasks
- Be concise and direct

# Available Tools
- Read: Read file contents
- Edit: Edit files with string replacement
- Write: Create new files
- Bash: Execute shell commands
- Glob: Find files by pattern
- Grep: Search file contents
`
```

### 7.3 CLAUDE.md 注入

Claude Code 会自动加载项目中的 CLAUDE.md 文件作为上下文：

```javascript
function loadClaudeMd(cwd) {
  const locations = [
    path.join(cwd, 'CLAUDE.md'),
    path.join(cwd, '.claude', 'CLAUDE.md'),
    path.join(os.homedir(), '.claude', 'CLAUDE.md'),
  ]

  const contents = []
  for (const loc of locations) {
    if (fs.existsSync(loc)) {
      contents.push(`# ${loc}\n${fs.readFileSync(loc, 'utf-8')}`)
    }
  }

  // 注入到 user context 中
  if (contents.length > 0) {
    return `<system-reminder>\n${contents.join('\n\n')}\n</system-reminder>`
  }
  return null
}
```

---

## 8. 完整代码示例

### 8.1 最小可运行的 API Key 客户端

```javascript
// minimal-client.mjs
// 用法: ANTHROPIC_API_KEY=sk-xxx node minimal-client.mjs "你的问题"

import fs from 'fs'
import { execSync } from 'child_process'
import path from 'path'

const API_KEY = process.env.ANTHROPIC_API_KEY
const API_URL = 'https://api.anthropic.com/v1/messages'
const MODEL = 'claude-sonnet-4-20250514'
const userPrompt = process.argv[2] || 'What files are in the current directory?'

// ---- 工具定义 ----
const tools = [
  {
    name: 'Read',
    description: 'Read a file from disk',
    input_schema: {
      type: 'object',
      properties: {
        file_path: { type: 'string', description: 'Absolute file path' },
        limit: { type: 'number', description: 'Max lines to read' },
        offset: { type: 'number', description: 'Start line (1-based)' }
      },
      required: ['file_path']
    }
  },
  {
    name: 'Bash',
    description: 'Execute a shell command',
    input_schema: {
      type: 'object',
      properties: {
        command: { type: 'string', description: 'Shell command to run' },
        description: { type: 'string', description: 'What this command does' }
      },
      required: ['command']
    }
  },
  {
    name: 'Edit',
    description: 'Replace text in a file',
    input_schema: {
      type: 'object',
      properties: {
        file_path: { type: 'string' },
        old_string: { type: 'string' },
        new_string: { type: 'string' }
      },
      required: ['file_path', 'old_string', 'new_string']
    }
  },
  {
    name: 'Write',
    description: 'Write content to a file (creates or overwrites)',
    input_schema: {
      type: 'object',
      properties: {
        file_path: { type: 'string' },
        content: { type: 'string' }
      },
      required: ['file_path', 'content']
    }
  }
]

// ---- 工具执行 ----
function executeTool(name, input) {
  try {
    switch (name) {
      case 'Read': {
        const content = fs.readFileSync(input.file_path, 'utf-8')
        const lines = content.split('\n')
        const start = (input.offset || 1) - 1
        const end = input.limit ? start + input.limit : lines.length
        return lines.slice(start, end)
          .map((l, i) => `${start + i + 1}\t${l}`)
          .join('\n')
      }
      case 'Bash': {
        const result = execSync(input.command, {
          encoding: 'utf-8',
          timeout: 120000,
          cwd: process.cwd(),
          stdio: ['pipe', 'pipe', 'pipe']
        })
        return result || '(no output)'
      }
      case 'Edit': {
        let content = fs.readFileSync(input.file_path, 'utf-8')
        if (!content.includes(input.old_string)) {
          return 'Error: old_string not found in file'
        }
        content = content.replace(input.old_string, input.new_string)
        fs.writeFileSync(input.file_path, content)
        return 'File edited successfully'
      }
      case 'Write': {
        fs.mkdirSync(path.dirname(input.file_path), { recursive: true })
        fs.writeFileSync(input.file_path, input.content)
        return 'File written successfully'
      }
      default:
        return `Unknown tool: ${name}`
    }
  } catch (err) {
    return `Error: ${err.message}`
  }
}

// ---- 流式请求 ----
async function streamRequest(messages) {
  const response = await fetch(API_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY,
      'anthropic-version': '2023-06-01',
      'anthropic-beta': 'interleaved-thinking-2025-05-14',
    },
    body: JSON.stringify({
      model: MODEL,
      max_tokens: 16384,
      stream: true,
      system: `You are a coding assistant. Working directory: ${process.cwd()}. Platform: ${process.platform}.`,
      messages,
      tools,
      thinking: { type: 'enabled', budget_tokens: 8000 }
    })
  })

  if (!response.ok) {
    const text = await response.text()
    throw new Error(`API error ${response.status}: ${text}`)
  }

  // 解析 SSE
  const content = []
  let stopReason = null
  const reader = response.body.getReader()
  const decoder = new TextDecoder()
  let buf = ''

  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    buf += decoder.decode(value, { stream: true })

    const lines = buf.split('\n')
    buf = lines.pop()

    for (const line of lines) {
      if (!line.startsWith('data: ')) continue
      const data = JSON.parse(line.slice(6))

      switch (data.type) {
        case 'content_block_start':
          if (data.content_block.type === 'tool_use') {
            content.push({ ...data.content_block, input: '' })
          } else if (data.content_block.type === 'text') {
            content.push({ type: 'text', text: '' })
          } else if (data.content_block.type === 'thinking') {
            content.push({ type: 'thinking', thinking: '' })
          }
          break

        case 'content_block_delta':
          const block = content[data.index]
          if (data.delta.type === 'text_delta') {
            block.text += data.delta.text
            process.stdout.write(data.delta.text)
          } else if (data.delta.type === 'input_json_delta') {
            block.input += data.delta.partial_json
          } else if (data.delta.type === 'thinking_delta') {
            block.thinking += data.delta.thinking
          }
          break

        case 'message_delta':
          stopReason = data.delta.stop_reason
          break
      }
    }
  }

  // 解析工具输入 JSON
  for (const block of content) {
    if (block.type === 'tool_use' && typeof block.input === 'string') {
      try { block.input = JSON.parse(block.input) } catch {}
    }
  }

  return { content, stop_reason: stopReason }
}

// ---- 主循环 ----
async function main() {
  console.log(`\n> ${userPrompt}\n`)
  const messages = [{ role: 'user', content: userPrompt }]

  let turns = 0
  while (turns++ < 20) {
    const result = await streamRequest(messages)
    messages.push({ role: 'assistant', content: result.content })

    if (result.stop_reason !== 'tool_use') {
      console.log('\n')
      break
    }

    // 执行工具
    const toolResults = []
    for (const block of result.content) {
      if (block.type !== 'tool_use') continue
      console.log(`\n[Tool: ${block.name}] ${JSON.stringify(block.input).slice(0, 100)}`)
      const output = executeTool(block.name, block.input)
      console.log(`  → ${output.slice(0, 200)}${output.length > 200 ? '...' : ''}`)
      toolResults.push({
        type: 'tool_result',
        tool_use_id: block.id,
        content: output
      })
    }
    messages.push({ role: 'user', content: toolResults })
    console.log('')
  }
}

main().catch(console.error)
```

### 8.2 OAuth 登录客户端

```javascript
// oauth-client.mjs
// 用法: node oauth-client.mjs

import http from 'http'
import crypto from 'crypto'
import fs from 'fs'
import os from 'os'
import path from 'path'
import { exec } from 'child_process'

const CLIENT_ID = '9d1c250a-e61b-44d9-88ed-5944d1962f5e'
const TOKEN_URL = 'https://platform.claude.com/v1/oauth/token'
const AUTH_URL = 'https://claude.com/cai/oauth/authorize'
const CREDENTIALS_PATH = path.join(os.homedir(), '.claude', '.credentials.json')

const SCOPES = [
  'user:profile',
  'user:inference',
  'user:sessions:claude_code',
  'user:mcp_servers',
  'user:file_upload'
]

// ---- 读取已保存的 Token ----
function loadSavedTokens() {
  try {
    const data = JSON.parse(fs.readFileSync(CREDENTIALS_PATH, 'utf-8'))
    return data.claudeAiOauth || null
  } catch { return null }
}

// ---- 保存 Token ----
function saveTokens(tokens) {
  const dir = path.dirname(CREDENTIALS_PATH)
  fs.mkdirSync(dir, { recursive: true })
  fs.writeFileSync(CREDENTIALS_PATH, JSON.stringify({
    claudeAiOauth: tokens
  }, null, 2), { mode: 0o600 })
}

// ---- 检查是否过期 ----
function isExpired(expiresAt) {
  return (Date.now() + 5 * 60 * 1000) >= expiresAt
}

// ---- 刷新 Token ----
async function refreshAccessToken(refreshToken) {
  const resp = await fetch(TOKEN_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: CLIENT_ID,
      scope: SCOPES.join(' ')
    })
  })

  if (!resp.ok) throw new Error(`Refresh failed: ${resp.status}`)
  const data = await resp.json()

  const tokens = {
    accessToken: data.access_token,
    refreshToken: data.refresh_token,
    expiresAt: Date.now() + data.expires_in * 1000,
    scopes: data.scope.split(' ')
  }
  saveTokens(tokens)
  return tokens
}

// ---- OAuth PKCE 登录 ----
async function oauthLogin() {
  const codeVerifier = crypto.randomBytes(32).toString('base64url')
  const codeChallenge = crypto.createHash('sha256')
    .update(codeVerifier).digest('base64url')
  const state = crypto.randomBytes(32).toString('base64url')

  // 启动回调服务器
  const { code, port } = await new Promise((resolve, reject) => {
    const server = http.createServer((req, res) => {
      const url = new URL(req.url, `http://localhost`)
      if (url.pathname === '/callback') {
        const code = url.searchParams.get('code')
        res.writeHead(200, { 'Content-Type': 'text/html' })
        res.end('<h1>Login successful!</h1><p>You can close this tab.</p>')
        server.close()
        resolve({ code, port: server.address().port })
      }
    })
    server.listen(0, '127.0.0.1', () => {
      const port = server.address().port
      const authUrl = new URL(AUTH_URL)
      authUrl.searchParams.set('client_id', CLIENT_ID)
      authUrl.searchParams.set('response_type', 'code')
      authUrl.searchParams.set('redirect_uri', `http://localhost:${port}/callback`)
      authUrl.searchParams.set('scope', SCOPES.join(' '))
      authUrl.searchParams.set('code_challenge', codeChallenge)
      authUrl.searchParams.set('code_challenge_method', 'S256')
      authUrl.searchParams.set('state', state)

      console.log('Opening browser for login...')
      console.log(authUrl.toString())

      // 打开浏览器
      const cmd = process.platform === 'win32' ? 'start ""'
        : process.platform === 'darwin' ? 'open' : 'xdg-open'
      exec(`${cmd} "${authUrl.toString()}"`)
    })
  })

  // 用 code 换 token
  const resp = await fetch(TOKEN_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'authorization_code',
      code,
      redirect_uri: `http://localhost:${port}/callback`,
      client_id: CLIENT_ID,
      code_verifier: codeVerifier,
      state
    })
  })

  if (!resp.ok) throw new Error(`Token exchange failed: ${resp.status}`)
  const data = await resp.json()

  const tokens = {
    accessToken: data.access_token,
    refreshToken: data.refresh_token,
    expiresAt: Date.now() + data.expires_in * 1000,
    scopes: data.scope.split(' '),
    accountUuid: data.account?.uuid,
    email: data.account?.email_address
  }
  saveTokens(tokens)
  console.log(`Logged in as: ${tokens.email}`)
  return tokens
}

// ---- 获取有效 Token ----
async function getValidToken() {
  let tokens = loadSavedTokens()

  if (!tokens) {
    console.log('No saved tokens, starting OAuth login...')
    tokens = await oauthLogin()
  } else if (isExpired(tokens.expiresAt)) {
    console.log('Token expired, refreshing...')
    tokens = await refreshAccessToken(tokens.refreshToken)
  }

  return tokens.accessToken
}

// ---- 使用 OAuth Token 调用 API ----
async function callClaude(accessToken, prompt) {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${accessToken}`,
      'anthropic-version': '2023-06-01',
      'anthropic-beta': 'oauth-2025-04-20,interleaved-thinking-2025-05-14',
      'x-app': 'cli',
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 16384,
      messages: [{ role: 'user', content: prompt }],
      thinking: { type: 'enabled', budget_tokens: 5000 }
    })
  })

  if (!response.ok) {
    const err = await response.text()
    throw new Error(`API error ${response.status}: ${err}`)
  }

  const data = await response.json()
  for (const block of data.content) {
    if (block.type === 'text') {
      console.log(block.text)
    }
  }
}

// ---- 主函数 ----
async function main() {
  const token = await getValidToken()
  const prompt = process.argv[2] || 'Say hello in Chinese'
  await callClaude(token, prompt)
}

main().catch(console.error)
```

---

## 9. 辅助 API 端点

### 9.1 用户资料

```http
GET https://api.anthropic.com/api/oauth/profile
Authorization: Bearer {accessToken}

Response:
{
  "account": {
    "uuid": "...",
    "email": "user@example.com",
    "display_name": "User",
    "created_at": "2024-01-01T00:00:00Z"
  },
  "organization": {
    "uuid": "...",
    "organization_type": "claude_max",
    "rate_limit_tier": "tier4",
    "has_extra_usage_enabled": false,
    "billing_type": "subscription",
    "subscription_created_at": "2024-01-01T00:00:00Z"
  }
}
```

### 9.2 用户角色

```http
GET https://api.anthropic.com/api/oauth/claude_cli/roles
Authorization: Bearer {accessToken}

Response:
{
  "organization_role": "owner",
  "workspace_role": "admin",
  "organization_name": "My Org"
}
```

### 9.3 通过 OAuth 创建 API Key

```http
POST https://api.anthropic.com/api/oauth/claude_cli/create_api_key
Authorization: Bearer {accessToken}

Response:
{
  "raw_key": "sk-ant-xxxxx"
}
```

### 9.4 Device ID 生成

```javascript
import crypto from 'crypto'
import fs from 'fs'
import path from 'path'
import os from 'os'

function getOrCreateDeviceId() {
  const configPath = path.join(os.homedir(), '.claude', 'config.json')
  try {
    const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'))
    if (config.userID) return config.userID
  } catch {}

  // 生成新的 device ID: 64 个十六进制字符
  const deviceId = crypto.randomBytes(32).toString('hex')

  // 保存
  const dir = path.dirname(configPath)
  fs.mkdirSync(dir, { recursive: true })
  let config = {}
  try { config = JSON.parse(fs.readFileSync(configPath, 'utf-8')) } catch {}
  config.userID = deviceId
  fs.writeFileSync(configPath, JSON.stringify(config, null, 2))

  return deviceId
}
```

---

## 10. 注意事项

### 10.1 关键差异

| 方面 | API Key | OAuth Token |
|------|---------|-------------|
| 请求头 | `x-api-key: sk-ant-xxx` | `Authorization: Bearer cact-xxx` |
| Beta | 不需要 oauth beta | 需要 `oauth-2025-04-20` |
| 计费 | Console 按量计费 | 订阅包含 |
| 限流 | API 限流规则 | 订阅限流规则 |
| 过期 | 永不过期 | 1 小时过期需刷新 |

### 10.2 请求限制

- 单次请求最多 **100 个媒体项**（图片+文档）
- 默认超时 **600 秒**（远程会话 120 秒）
- 流式响应**空闲看门狗 90 秒**
- 停顿超过 **30 秒**记录警告

### 10.3 Metadata 格式

Claude Code 发送的 metadata.user_id 是 JSON 字符串：

```json
{
  "device_id": "64位hex字符串",
  "account_uuid": "OAuth账户UUID或空字符串",
  "session_id": "当前会话UUID"
}
```

### 10.4 安全建议

- **永远不要**在代码中硬编码 API Key
- OAuth Token 的 `refresh_token` 要安全存储（600 权限）
- 对用户输入的文件路径进行验证，防止路径遍历
- Bash 命令执行需要沙盒或权限控制
- 生产环境建议使用 OAuth 方式而非 API Key

### 10.5 可复用 Claude Code 的已保存凭据

如果用户已经通过 `claude auth login` 登录过，可以直接读取：

```javascript
// 读取 Claude Code 已保存的 OAuth Token
import fs from 'fs'
import os from 'os'
import path from 'path'

const credPath = path.join(os.homedir(), '.claude', '.credentials.json')
const creds = JSON.parse(fs.readFileSync(credPath, 'utf-8'))
const accessToken = creds.claudeAiOauth.accessToken
const expiresAt = creds.claudeAiOauth.expiresAt

// 检查是否过期
if (Date.now() + 300000 >= expiresAt) {
  // 需要刷新，使用 refreshToken
}
```

---

*基于 Claude Code v2.1.88 源码逆向分析*
*生成时间：2026-03-31*