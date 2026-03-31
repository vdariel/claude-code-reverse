# Claude Code v2.1.88 源码分析报告

hello，我是applek，某个软件站的站长。
今天拿到了泄露的map文件，本项目已经解析成了报告。

并且，如何制作第三客户端，也写出来了，本项目旨在学习claude code的功能，请勿用于违法用途


## 目录

1. [概述](#1-概述)
2. [程序启动流程](#2-程序启动流程)
3. [核心架构](#3-核心架构)
4. [模块详解](#4-模块详解)
5. [API 通信与数据流](#5-api-通信与数据流)
6. [工具系统](#6-工具系统)
7. [权限与安全系统](#7-权限与安全系统)
8. [UI 渲染系统](#8-ui-渲染系统)
9. [扩展机制](#9-扩展机制)
10. [关键数据流图](#10-关键数据流图)
11. [构建系统与运行测试](#11-构建系统与运行测试)
12. [如何构建第三方客户端](THIRD_PARTY_CLIENT_GUIDE.md)

---

## 1. 概述

Claude Code 是 Anthropic 官方的 CLI 编程助手，基于 TypeScript 开发，使用 React Ink 进行终端 UI 渲染。整个程序打包为单个 `cli.js` 文件（13MB），源码包含 **1906 个自有源文件** 和 2850 个第三方依赖文件。

### 技术栈

| 层级 | 技术 |
|------|------|
| 语言 | TypeScript |
| 运行时 | Node.js (Bun 构建) |
| 终端 UI | React Ink (自定义 fork) |
| 状态管理 | Zustand |
| API 客户端 | @anthropic-ai/sdk |
| CLI 框架 | Commander.js |
| 搜索引擎 | ripgrep (内置二进制) |
| 协议扩展 | MCP (Model Context Protocol) |

### 源码规模

| 模块 | 文件数 | 说明 |
|------|--------|------|
| tools/ | 40+ 个工具目录 | 每个工具一个子目录 |
| services/ | 15+ 个服务目录 | API、MCP、分析、OAuth 等 |
| components/ | 100+ 组件 | React Ink UI 组件 |
| utils/ | 200+ 工具文件 | 权限、Bash 解析、设置等 |
| commands/ | 90+ 命令 | 斜杠命令实现 |

---

## 2. 程序启动流程

### 2.1 入口点 (entrypoints/cli.tsx)

程序启动采用"快速路径"设计，根据命令行参数尽早分流，避免加载不需要的模块：

```
node cli.js [args]
      │
      ├─ --version          → 直接输出版本号，零导入退出
      ├─ --dump-system-prompt → 输出系统提示词（内部调试）
      ├─ --chrome-native-host → Chrome 浏览器扩展集成
      ├─ --computer-use-mcp  → Computer Use MCP 服务器
      ├─ --daemon-worker     → 内部守护进程 worker
      ├─ remote/bridge       → 远程会话桥接模式
      ├─ daemon              → 长时运行守护进程
      ├─ ps/logs/attach/kill → 会话管理命令
      ├─ new/list/reply      → 模板任务命令
      └─ [default]           → 加载完整 CLI (main.tsx)
```

### 2.2 初始化阶段 (entrypoints/init.ts)

`init()` 函数采用 memoize 模式确保只执行一次，按顺序执行：

**第一阶段：配置系统**
1. `enableConfigs()` — 启用并验证配置系统
2. `applySafeConfigEnvironmentVariables()` — 设置安全的环境变量
3. `applyExtraCACertsFromConfig()` — 加载 TLS 证书（必须在首次 TLS 连接前）

**第二阶段：系统设置**
4. `setupGracefulShutdown()` — 注册退出清理处理
5. `registerCleanup()` — LSP 服务、Team 资源清理
6. `ensureScratchpadDir()` — 创建临时目录

**第三阶段：异步初始化（非阻塞）**
7. OAuth 账户信息获取
8. JetBrains IDE 检测
9. GitHub 仓库检测
10. 远程设置加载
11. 策略限制加载

**第四阶段：网络配置**
12. `configureGlobalMTLS()` — mTLS 双向认证
13. `configureGlobalAgents()` — HTTP 代理配置
14. `preconnectAnthropicApi()` — **TCP+TLS 握手预热**（提前建立连接）
15. `initUpstreamProxy()` — 上游代理中继

### 2.3 全局状态初始化 (bootstrap/state.ts)

模块加载时立即初始化全局状态单例 `STATE`，包含 **260+ 属性**：

- 会话标识：sessionId、projectRoot、originalCwd
- 指标：totalCostUSD、totalAPIDuration、tokenUsage
- 遥测：meter、counters、tracerProvider
- 配置：mainLoopModel、permissionMode、autoMode
- 认证：oauthToken、apiKey、sessionIngressToken
- 工具：plugins、skills、cronTasks
- UI：agentColorMap、slowOperations

通过 **210+ getter/setter 函数** 提供原子访问。

### 2.4 主循环启动 (main.tsx)

1. 启动性能分析检查点
2. 从 Keychain 预取 OAuth + API Key
3. 加载 GrowthBook 特性开关
4. 调用 `init()` 完成初始化
5. 配置模型和权限
6. 加载 Plugins、Skills、Agents
7. 启动 MCP 服务器
8. 注册 Commander.js 命令
9. 渲染 React Ink UI (`REPL.tsx`)

---

## 3. 核心架构

### 3.1 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                    用户输入 (PromptInput)                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                 主消息循环 (query.ts)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │系统提示构建│  │消息标准化│  │  上下文管理/压缩      │   │
│  └──────────┘  └──────────┘  └──────────────────────┘   │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              Anthropic API 客户端 (claude.ts)             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │流式请求   │  │重试逻辑  │  │ 多供应商支持          │   │
│  │          │  │          │  │ (Direct/Bedrock/      │   │
│  │          │  │          │  │  Vertex/Azure)        │   │
│  └──────────┘  └──────────┘  └──────────────────────┘   │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│           工具执行引擎 (StreamingToolExecutor)             │
│  ┌──────────────────────────────────────────────────┐   │
│  │ 并发安全工具 → 并行执行    非并发工具 → 串行执行     │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │
│  │ Bash │ │ Edit │ │ Read │ │ Grep │ │ Agent│ ...      │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              UI 渲染层 (React Ink)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │消息列表   │  │权限对话框│  │ 后台任务面板          │   │
│  └──────────┘  └──────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 3.2 目录结构与模块关系

```
src/src/
├── entrypoints/       # 入口点 — 程序启动分发
├── bootstrap/         # 全局状态初始化
├── screens/           # 顶层屏幕组件 (REPL.tsx 是主界面)
├── components/        # UI 组件 (React Ink)
│   ├── messages/      #   消息渲染
│   ├── permissions/   #   权限对话框
│   ├── PromptInput/   #   用户输入框
│   ├── diff/          #   差异显示
│   ├── tasks/         #   后台任务 UI
│   ├── mcp/           #   MCP 服务器管理 UI
│   └── agents/        #   Agent 创建向导
├── tools/             # 工具实现 (40+ 工具)
│   ├── BashTool/      #   Shell 命令执行
│   ├── FileEditTool/  #   文件编辑
│   ├── FileReadTool/  #   文件读取
│   ├── AgentTool/     #   子 Agent 生成
│   ├── WebFetchTool/  #   网页获取
│   └── ...
├── services/          # 服务层
│   ├── api/           #   Anthropic API 通信
│   ├── tools/         #   工具执行编排
│   ├── compact/       #   上下文压缩
│   ├── mcp/           #   MCP 协议客户端
│   ├── analytics/     #   遥测与分析
│   ├── oauth/         #   OAuth 认证
│   ├── lsp/           #   IDE 语言服务
│   └── plugins/       #   插件管理
├── commands/          # 斜杠命令 (/help, /compact, /model 等)
├── hooks/             # Hook 系统 (PreToolUse, PostToolUse, Stop)
├── state/             # 应用状态管理 (Zustand)
├── context/           # React Context 提供者
├── tasks/             # 后台任务执行器
│   ├── LocalShellTask/    # 本地 Shell 任务
│   ├── LocalAgentTask/    # 本地 Agent 任务
│   ├── RemoteAgentTask/   # 远程 Agent 任务
│   └── DreamTask/         # 后台规划任务
├── skills/            # 技能/命令系统
├── plugins/           # 插件架构
├── coordinator/       # 多 Agent 协调器
├── remote/            # 远程会话管理
├── utils/             # 工具库
│   ├── permissions/   #   权限引擎 (400KB+)
│   ├── bash/          #   Bash AST 解析器 (600KB+)
│   ├── model/         #   模型选择与能力检测
│   ├── settings/      #   设置文件管理
│   ├── sandbox/       #   沙盒隔离
│   ├── telemetry/     #   OpenTelemetry 集成
│   ├── git/           #   Git 集成
│   └── hooks/         #   Hook 加载与执行
├── types/             # TypeScript 类型定义
├── constants/         # 常量与系统提示词
├── voice/             # 语音输入
├── vim/               # Vim 模式
└── ink/               # 自定义 Ink 组件
```

---

## 4. 模块详解

### 4.1 entrypoints/ — 入口点

| 文件 | 大小 | 功能 |
|------|------|------|
| cli.tsx | 39KB | 主 CLI 启动分发器，根据参数路由到不同模式 |
| init.ts | 14KB | 核心初始化：配置、网络、遥测、清理 |
| mcp.ts | 6KB | MCP 服务器启动（stdio 传输） |
| agentSdkTypes.ts | 13KB | Agent SDK 类型导出 |
| sandboxTypes.ts | 6KB | 沙盒环境类型定义 |

### 4.2 bootstrap/ — 全局状态

`state.ts`（56KB）是整个应用的状态核心：
- 声明为"import DAG 的叶节点"以避免循环依赖
- 模块加载时立即创建 `STATE` 单例
- 210+ 导出的 getter/setter 函数
- 支持会话切换（`switchSession()`原子更新 sessionId + projectDir）
- 支持会话 ID 重新生成（清理旧的 plan-slug 缓存）

### 4.3 screens/ — 屏幕组件

| 文件 | 大小 | 功能 |
|------|------|------|
| REPL.tsx | **895KB** | 主交互界面，消息列表、输入框、Footer、推测执行 |
| Doctor.tsx | 73KB | 诊断/排错界面，系统信息、权限检查 |
| ResumeConversation.tsx | 59KB | 会话恢复界面 |

`REPL.tsx` 是最大的单文件，实现了：
- 消息列表渲染与滚动
- 用户输入处理
- 推测执行引擎（用户输入时预取响应）
- 后台任务管理
- Footer 状态条（tasks/teams/bridge pills）

### 4.4 commands/ — 斜杠命令

90+ 个斜杠命令，每个命令一个子目录。主要命令：

| 命令 | 功能 |
|------|------|
| /help | 帮助信息 |
| /compact | 手动压缩上下文 |
| /model | 切换模型 |
| /clear | 清除对话 |
| /config | 配置管理 |
| /login, /logout | 认证管理 |
| /mcp | MCP 服务器管理 |
| /hooks | Hook 管理 |
| /skills | 技能管理 |
| /tasks | 任务管理 |
| /diff | 查看文件差异 |
| /review | 代码审查 |
| /plan | 计划模式 |
| /resume | 恢复会话 |
| /share | 分享对话 |
| /voice | 语音输入 |
| /vim | Vim 编辑模式 |
| /fast | 切换快速模式 |
| /cost | 查看费用 |
| /doctor | 系统诊断 |
| /memory | 记忆管理 |
| /upgrade | 自动更新 |
| /desktop | 桌面应用 |
| /theme | 主题切换 |
| /permissions | 权限管理 |
| /stickers | 贴纸（彩蛋） |

### 4.5 coordinator/ — 多 Agent 协调

`coordinatorMode.ts`（19KB）实现了多 Agent 协调模式：
- 主 Agent 作为"协调者"分配工作
- Worker Agent 通过 AgentTool 生成
- Worker 拥有受限的工具集（Bash、Edit、Read、MCP 工具）
- Worker 可以进一步生成子 Agent

### 4.6 remote/ — 远程会话

| 文件 | 功能 |
|------|------|
| RemoteSessionManager.ts | 远程会话控制，WebSocket 消息流 |
| SessionsWebSocket.ts | WebSocket 客户端，自动重连 |
| sdkMessageAdapter.ts | SDK 消息格式适配器 |
| remotePermissionBridge.ts | 远程权限请求桥接 |

### 4.7 tasks/ — 后台任务执行

| 任务类型 | 文件 | 功能 |
|----------|------|------|
| LocalShellTask | 66KB | 本地 Shell 命令执行，流式输出 |
| LocalAgentTask | 82KB | 本地 Agent 查询执行 |
| RemoteAgentTask | 126KB | 远程 Agent 会话控制 |
| InProcessTeammateTask | — | 进程内 Teammate 执行 |
| DreamTask | — | 后台规划（Dream 模式） |

任务状态结构：
```typescript
TaskState {
  id: string           // 前缀区分: b(bash)/a(agent)/r(remote)/t(teammate)
  type: TaskType       // local_bash / local_agent / remote_agent / ...
  status: TaskStatus   // pending → running → completed / failed / killed
  description: string  // 用户可见的任务名
  outputFile: string   // 输出文件路径
  startTime: number
  endTime?: number
}
```

### 4.8 voice/ — 语音输入

最小模块，仅含 `voiceModeEnabled.ts`，通过特性开关控制。

### 4.9 vim/ — Vim 模式

提供 Vim 风格的编辑快捷键支持。

### 4.10 ink/ — 自定义 Ink 组件

自定义的 React Ink 渲染层：
- `components/` — 基础 UI 组件
- `events/` — 事件系统
- `hooks/` — 自定义 React Hooks
- `layout/` — 布局引擎（基于 yoga-layout）
- `termio/` — 终端 I/O

---

## 5. API 通信与数据流

### 5.1 客户端初始化 (services/api/client.ts)

支持 4 种 API 供应商：

| 供应商 | SDK | 认证方式 |
|--------|-----|----------|
| Anthropic Direct | Anthropic | API Key 或 OAuth Token |
| AWS Bedrock | AnthropicBedrock | AWS IAM |
| Google Vertex AI | AnthropicVertex | Google Auth |
| Azure Foundry | AnthropicFoundry | Azure AD |

默认请求头：
```
x-app: 'cli'
User-Agent: claude-code/{version}
X-Claude-Code-Session-Id: {sessionId}
x-client-request-id: {uuid}
Authorization: Bearer {token}
```

### 5.2 系统提示词构建

系统提示词分为两部分，利用 **Prompt Caching** 优化：

**静态部分（跨用户可缓存）**：
1. 角色介绍与基本指令
2. 工具使用说明文档
3. 任务定义
4. 设置/Hook 说明
5. 系统提醒

**动态部分（用户/会话特定）**：
6. 输出风格配置
7. 语言偏好
8. 工具描述（基于可用工具列表）
9. MCP 服务器说明
10. 上下文管理指令
11. CLAUDE.md 文件内容
12. 记忆系统指令

缓存策略：
- 静态前缀：全局作用域，1 小时 TTL
- 动态后缀：临时作用域

### 5.3 请求构建 (services/api/claude.ts)

每次 API 调用构建的参数结构：

```typescript
{
  model: "claude-sonnet-4-20250514",  // 标准化后的模型名
  messages: [...],                     // 带缓存断点的消息数组
  system: TextBlockParam[],            // 分段系统提示词
  tools: ToolSchema[],                 // 工具定义 schema
  tool_choice: "auto" | "any" | {...}, // 工具选择策略

  // Beta 特性标头（按需启用）
  betas: [
    "prompt-caching-2024-07-31",        // 提示缓存
    "interleaved-thinking-2025-05-14",  // 自适应思考
    "fast-mode-2025-04-18",             // 快速模式
    "tool-search-2025-04-15",           // 工具搜索/延迟加载
    "cache-editing-2025-05-15",         // 缓存微调
    "context-management-2025-06-01",    // API 侧上下文管理
    "structured-outputs-2025-06-01",    // 结构化输出
    "task-budgets-2025-06-01",          // 任务预算
  ],

  // 元数据
  metadata: {
    user_id: JSON.stringify({
      device_id: "...",
      account_uuid: "...",
      session_id: "...",
    })
  },

  max_tokens: 16384,                // 最大输出 token
  thinking: { type: 'adaptive' },   // 自适应思考
  temperature: 1,                    // 温度

  // 输出配置
  output_config: {
    effort: "high",                  // 推理努力程度
    task_budget: { type: 'tokens', total: N },
  },

  speed: 'fast',                     // 快速模式标记
  stream: true                       // 流式响应
}
```

### 5.4 流式响应处理

```
API Response Stream
      │
      ├─ message_start       → 初始化 partialMessage，读取 usage
      ├─ content_block_start → 初始化 text/tool_use/thinking 块
      ├─ content_block_delta → 累积文本/工具输入/思考内容
      ├─ message_delta       → 更新 usage
      └─ message_stop        → 最终消息 + stop_reason
```

保护机制：
- **首 Token 超时**：测量 TTFB（Time To First Byte）
- **停顿检测**：30 秒无数据输出警告
- **空闲看门狗**：90 秒无活动终止流
- **资源清理**：`cleanupStream()` + `Response.body.cancel()`

### 5.5 重试与错误处理

| HTTP 状态码 | 处理方式 |
|-------------|----------|
| 408 (超时) | 立即重试 |
| 429 (限流) | 指数退避重试 |
| 500-599 (服务器错误) | 指数退避重试 |
| 529 (过载) | 退避重试 + 连续计数器 |
| 401 (认证失败) | 降级或失败 |
| 400 (验证失败) | 立即失败 |
| 连接超时 | 降级为非流式请求 |

非流式降级路径：
- 流式失败时回退到 `executeNonStreamingRequest()`
- 最大 64K tokens 输出（vs 流式无限制）
- 300 秒超时

### 5.6 上下文压缩 (services/compact/)

当上下文接近窗口限制时自动触发：

| 文件 | 功能 |
|------|------|
| compact.ts | 主压缩编排，使用分支 Agent 生成摘要 |
| autoCompact.ts | 自动压缩触发器（基于 token 阈值） |
| microCompact.ts | 微压缩，缓存键重新生成 |
| apiMicrocompact.ts | API 侧上下文管理集成 |
| postCompactCleanup.ts | 压缩后清理（移除附件、更新索引） |

---

## 6. 工具系统

### 6.1 工具框架 (Tool.ts)

所有工具通过 `buildTool()` 函数构建，定义接口：

```typescript
ToolDef<Input, Output> {
  name: string                      // 工具名称
  description(input?): string       // 工具描述
  prompt(): string                  // 给模型的使用说明
  inputSchema: ZodSchema            // 输入参数验证
  outputSchema: ZodSchema           // 输出结构验证
  isConcurrencySafe(input?): bool   // 是否可并发执行
  isReadOnly(input?): bool          // 是否只读操作
  isDestructive(input?): bool       // 是否破坏性操作
  checkPermissions(input): Result   // 权限检查
  call(input, context): Result      // 执行逻辑
}
```

执行上下文 `ToolUseContext` 包含：
- 命令注册表、调试标志
- MCP 客户端/资源
- 文件状态缓存（LRU）
- 权限上下文、中止信号
- 消息历史、工作目录

### 6.2 工具列表

#### 文件操作工具

| 工具 | 功能 | 关键特性 |
|------|------|----------|
| **FileReadTool** | 读取文件 | 支持 PDF/图片/Notebook，大文件分段读取 |
| **FileEditTool** | 编辑文件 | 字符串替换，Git diff 跟踪，LSP 通知 |
| **FileWriteTool** | 写入文件 | 创建新文件或完全覆盖 |
| **GlobTool** | 文件搜索 | Glob 模式匹配，最多返回 100 个结果 |
| **GrepTool** | 内容搜索 | 使用内置 ripgrep，支持正则、上下文行、多行匹配 |

#### 执行工具

| 工具 | 功能 | 关键特性 |
|------|------|----------|
| **BashTool** | Shell 命令 | 沙盒支持、安全分析、后台任务、大输出持久化 |
| **PowerShellTool** | PowerShell | Windows 专用 |
| **REPLTool** | Python/Node REPL | 交互式执行 |

#### 搜索/网络工具

| 工具 | 功能 | 关键特性 |
|------|------|----------|
| **WebFetchTool** | 网页获取 | HTML→Markdown 转换，模型处理内容 |
| **WebSearchTool** | 网络搜索 | 使用 Anthropic web_search API |
| **ToolSearchTool** | 延迟工具发现 | 按需加载工具 schema |

#### 多 Agent 工具

| 工具 | 功能 | 关键特性 |
|------|------|----------|
| **AgentTool** | 生成子 Agent | 流式执行、模型覆盖、Worktree 隔离、后台执行 |
| **SendMessageTool** | 发送消息给 Agent | 进程内通信 |
| **SkillTool** | 调用技能命令 | 技能解析、MCP 集成 |

#### 任务管理工具

| 工具 | 功能 |
|------|------|
| **TaskCreateTool** | 创建任务 |
| **TaskUpdateTool** | 更新任务状态 |
| **TaskGetTool** | 获取任务详情 |
| **TaskListTool** | 列出所有任务 |
| **TaskStopTool** | 停止任务 |
| **TaskOutputTool** | 读取任务输出 |

#### 调度工具

| 工具 | 功能 |
|------|------|
| **CronCreateTool** | 创建定时任务（cron 表达式） |
| **CronDeleteTool** | 删除定时任务 |
| **CronListTool** | 列出定时任务 |
| **RemoteTriggerTool** | 远程 Agent 触发器管理 |

#### MCP 工具

| 工具 | 功能 |
|------|------|
| **MCPTool** | 执行 MCP 工具调用 |
| **ListMcpResourcesTool** | 列出 MCP 资源 |
| **ReadMcpResourceTool** | 读取 MCP 资源 |
| **McpAuthTool** | MCP OAuth 认证 |

#### 工作流工具

| 工具 | 功能 |
|------|------|
| **EnterWorktreeTool** | 进入 Git Worktree 隔离环境 |
| **ExitWorktreeTool** | 退出 Worktree |
| **EnterPlanModeTool** | 进入计划模式 |
| **ExitPlanModeTool** | 退出计划模式 |

#### 其他工具

| 工具 | 功能 |
|------|------|
| **AskUserQuestionTool** | 向用户提问 |
| **NotebookEditTool** | 编辑 Jupyter Notebook |
| **LSPTool** | 语言服务协议操作 |
| **ConfigTool** | 配置管理 |
| **SleepTool** | 暂停执行 |
| **BriefTool** | 为异步 Agent 生成摘要 |
| **TodoWriteTool** | 待办事项管理 |

### 6.3 工具执行流程

```
API 响应包含 tool_use 块
          │
          ▼
StreamingToolExecutor（并发控制）
  ├─ 分析工具并发安全性
  ├─ 并发安全工具 → 并行执行（最多 10 个）
  └─ 非并发工具 → 串行执行
          │
          ▼
toolExecution.ts（单个工具执行）
  1. 执行 PreToolUse Hooks
  2. 检查权限 (checkPermissions)
  3. 调用 tool.call(input, context)
  4. 执行 PostToolUse Hooks
  5. 格式化结果
          │
          ▼
结果组装（按工具调用顺序）
  → 创建 tool_result 消息
  → 发送到下一轮 API 调用
```

### 6.4 BashTool 深入分析

BashTool 是最复杂的工具（160KB+ 主文件），包含：

**安全分析** (`bashSecurity.ts`, 102KB)：
- 完整的 Bash AST 解析器
- 危险命令检测（rm -rf、dd、chmod 等）
- 命令链分析（pipe、重定向）
- 路径安全验证

**权限规则** (`bashPermissions.ts`, 98KB)：
- 基于 Glob 模式匹配命令
- 只读操作自动放行
- 写操作需要用户确认

**后台执行**：
- 超过 15 秒的命令可转为后台任务
- 通过 `LocalShellTask` 管理
- 输出流式写入文件
- 大输出（>30KB）持久化到 `.claude-code/tool-results/`

### 6.5 AgentTool 深入分析

AgentTool 实现了子 Agent 机制：

```
父 Agent 调用 AgentTool
      │
      ├─ prompt: 任务描述
      ├─ subagent_type: 专用 Agent 类型
      ├─ model: 模型覆盖 (sonnet/opus/haiku)
      ├─ isolation: 'worktree' (Git 隔离)
      └─ run_in_background: 后台执行
            │
            ▼
      创建新的 API 会话
      ├─ 继承父 Agent 的工具池
      ├─ 继承权限上下文
      ├─ 独立的消息历史
      └─ 流式响应回传给父 Agent
```

内置子 Agent 类型通过 `tools/AgentTool/built-in/` 定义。

---

## 7. 权限与安全系统

### 7.1 权限引擎 (utils/permissions/)

权限系统是 Claude Code 最大的子系统之一（400KB+），实现了多层安全检查：

```
工具调用请求
      │
      ▼
① 检查显式拒绝规则 → 拒绝
      │
      ▼
② 检查显式允许规则 → 允许
      │
      ▼
③ 检查分类器 (Auto Mode) → 自动决策
      │
      ▼
④ 检查拒绝模式 → 拒绝
      │
      ▼
⑤ 检查危险模式 → 拒绝
      │
      ▼
⑥ 显示权限对话框 → 用户决策
      │
      ▼
⑦ 记录决策，更新状态
```

### 7.2 权限模式

| 模式 | 行为 |
|------|------|
| default | 不确定时询问用户 |
| plan | 需要明确批准每个操作 |
| bypass | 允许所有操作（危险） |
| auto | 分类器自动决策 |

### 7.3 Bash 安全分析

`utils/bash/` 包含完整的 Bash 语法解析器（600KB+）：

| 文件 | 大小 | 功能 |
|------|------|------|
| bashParser.ts | 130KB | Bash 语法 → AST |
| ast.ts | 112KB | AST 类型定义 |
| commands.ts | 51KB | 命令分类（读/写/删除） |
| heredoc.ts | 31KB | Heredoc 解析 |
| ShellSnapshot.ts | 21KB | Shell 环境快照 |

功能：
- 解析命令为 AST 树
- 检测危险模式（命令注入、路径遍历等）
- 分类命令为只读/写入/删除
- 提取重定向和管道

### 7.4 沙盒隔离 (utils/sandbox/)

`sandbox-adapter.ts`（35KB）提供沙盒执行环境：
- 桥接到原生沙盒进程
- 文件覆盖层管理
- 读写操作拦截

### 7.5 Hook 安全检查

三种 Hook 类型在不同阶段执行：

| Hook | 触发时机 | 用途 |
|------|----------|------|
| PreToolUse | 工具执行前 | 验证参数、修改输入 |
| PostToolUse | 工具执行后 | 自动格式化、检查 |
| Stop | 会话结束时 | 最终验证 |

---

## 8. UI 渲染系统

### 8.1 技术栈

使用 **React Ink**（Claude Code 自定义 fork）在终端渲染 UI：
- 基于 yoga-layout 引擎进行布局
- 自定义 termio 模块处理终端 I/O
- 自定义事件系统

### 8.2 主界面结构 (REPL.tsx)

```
┌─────────────────────────────────────────────┐
│           消息列表 (MessageList)              │
│  ┌─────────────────────────────────────────┐ │
│  │ [User] 用户消息                          │ │
│  │ [Assistant] 助手回复                     │ │
│  │ [Tool] 工具调用与结果                    │ │
│  │ [Thinking] 思考过程                     │ │
│  │ ...                                     │ │
│  └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│           权限对话框 (Permission)             │
│  "Allow BashTool to run: npm test? [y/n]"   │
├─────────────────────────────────────────────┤
│           输入框 (PromptInput)               │
│  ┌─────────────────────────────────────────┐ │
│  │ > 用户输入...                           │ │
│  └─────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │ [Tasks:2] [Model:opus] [Cost:$0.12]     │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 8.3 消息组件

| 组件 | 功能 |
|------|------|
| AssistantTextMessage | 渲染助手文本（Markdown） |
| AssistantToolUseMessage | 显示工具调用详情 |
| AssistantThinkingMessage | 显示思考过程 |
| UserPromptMessage | 用户输入显示 |
| AttachmentMessage | 文件/图片附件 |
| SystemTextMessage | 系统通知 |
| RateLimitMessage | 限流通知 |
| PlanApprovalMessage | 计划审批对话框 |

### 8.4 权限对话框组件

| 组件 | 功能 |
|------|------|
| BashPermissionRequest | Shell 命令审批 |
| FileEditPermissionRequest | 文件修改审批 |
| FileWritePermissionRequest | 文件创建审批 |
| WebFetchPermissionRequest | 网络请求审批 |
| SkillPermissionRequest | 技能执行审批 |
| ComputerUseApproval | 计算机使用审批 |

### 8.5 后台任务面板

| 组件 | 功能 |
|------|------|
| BackgroundTasksDialog | 任务列表主对话框 |
| BackgroundTask | 单个任务卡片 |
| ShellDetailDialog | Shell 任务详情 |
| AsyncAgentDetailDialog | Agent 任务详情 |
| RemoteSessionDetailDialog | 远程会话详情 |

---

## 9. 扩展机制

### 9.1 MCP (Model Context Protocol)

MCP 是 Claude Code 最重要的扩展协议，允许连接外部工具服务器。

支持的传输方式：

| 传输 | 说明 |
|------|------|
| stdio | 本地进程（最常用） |
| SSE | Server-Sent Events |
| HTTP | HTTP REST |
| WebSocket | WebSocket 双向通信 |

配置示例（`.claude/settings.json`）：
```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["server.js"],
      "type": "stdio"
    }
  }
}
```

MCP 管理模块 (`services/mcp/`)：
- `MCPConnectionManager.tsx` — 连接生命周期管理
- `client.ts` — MCP 协议客户端
- `types.ts` — 配置 Schema
- `auth.ts` — OAuth 认证流程
- `officialRegistry.ts` — 官方 MCP 服务器注册表

### 9.2 技能系统 (Skills)

技能是 Markdown 文件，存储在 `~/.claude/skills/` 或 `.claude/skills/`：

```markdown
---
name: commit
description: Create a git commit
tools:
  - bash
  - fileEdit
---

Create a well-formatted commit with conventional commit messages...
```

加载流程 (`skills/loadSkillsDir.ts`)：
1. 扫描技能目录
2. 解析 Markdown 前置元数据
3. 提取提示内容
4. Token 计数估算
5. 注册为可调用命令

### 9.3 插件系统 (Plugins)

插件类型：
- UI 插件（添加组件）
- 工具插件（添加工具）
- Hook 插件（拦截事件）

注册在 `settings.json` 中，启动时加载。

### 9.4 Hook 系统

Hook 定义在 `settings.json` 中：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "BashTool",
        "hooks": [
          { "type": "command", "command": "echo 'checking...'" }
        ]
      }
    ],
    "PostToolUse": [...],
    "Stop": [...]
  }
}
```

Hook 执行类型：
- `command` — Shell 命令
- `agent` — Agent 提示
- `prompt` — 提示模板
- `http` — HTTP 请求

---

## 10. 关键数据流图

### 10.1 完整请求-响应循环

```
用户输入 "fix the bug in auth.ts"
    │
    ▼
[PromptInput] 捕获输入
    │
    ▼
[query.ts] 构建请求
    ├── 获取系统提示词
    ├── 标准化消息历史
    ├── 构建工具 Schema
    └── 设置缓存策略
    │
    ▼
[claude.ts] 发送 API 请求
    ├── model: "claude-sonnet-4-20250514"
    ├── messages: [{role:"user", content:"fix the bug..."}]
    ├── tools: [{name:"Read",...}, {name:"Edit",...}, ...]
    ├── system: [静态提示, 动态提示]
    └── stream: true
    │
    ▼
Anthropic API 返回流式响应
    │
    ├── text: "Let me read the file first."
    │
    ├── tool_use: {name:"Read", input:{file_path:"auth.ts"}}
    │       │
    │       ▼
    │   [StreamingToolExecutor]
    │       │
    │       ▼
    │   [toolExecution.ts]
    │       ├── PreToolUse Hook
    │       ├── 权限检查 → 自动允许（只读）
    │       ├── FileReadTool.call({file_path:"auth.ts"})
    │       ├── PostToolUse Hook
    │       └── 返回文件内容
    │       │
    │       ▼
    │   tool_result 消息发送到下一轮 API
    │
    ├── text: "I found the issue. Let me fix it."
    │
    ├── tool_use: {name:"Edit", input:{file_path:"auth.ts", ...}}
    │       │
    │       ▼
    │   [权限对话框] → 用户确认 "Allow?"
    │       │
    │       ▼
    │   FileEditTool.call({...})
    │       └── 修改文件, 返回 diff
    │
    └── text: "I've fixed the authentication bug."
    │
    ▼
[REPL.tsx] 渲染所有消息到终端
```

### 10.2 子 Agent 执行流

```
主 Agent 需要搜索代码库
    │
    ▼
调用 AgentTool({
  description: "Search codebase",
  prompt: "Find all auth-related files",
  subagent_type: "Explore",
  model: "haiku"
})
    │
    ▼
[AgentTool.tsx] 创建子 Agent
    ├── 新的 API 会话
    ├── 继承工具池（受限）
    ├── 独立消息历史
    └── 子 Agent 系统提示
    │
    ▼
子 Agent 独立运行
    ├── 调用 GrepTool
    ├── 调用 GlobTool
    ├── 调用 FileReadTool
    └── 汇总结果
    │
    ▼
子 Agent 返回结果给父 Agent
    └── "Found 12 auth-related files: ..."
    │
    ▼
父 Agent 继续处理
```

### 10.3 遥测数据流

```
应用事件
    │
    ├── logEvent() → 事件队列
    │       │
    │       ▼
    │   事件路由 (sink.ts)
    │       ├── Datadog (datadog.ts)
    │       └── 第一方日志 (firstPartyEventLogger.ts)
    │
    ├── OpenTelemetry (instrumentation.ts)
    │       ├── Metrics → Counter/Histogram
    │       ├── Traces → Span
    │       └── Logs → Logger
    │       │
    │       ▼
    │   导出器
    │       ├── OTLP Exporter
    │       ├── BigQuery Exporter
    │       └── Perfetto 格式
    │
    └── GrowthBook (growthbook.ts)
            └── 特性开关评估
```

### 10.4 状态管理流

```
[bootstrap/state.ts]     [state/AppStateStore.ts]
   全局 STATE 单例           Zustand Store
        │                        │
        │                        │
   ┌────┴────┐              ┌────┴────┐
   │ 会话状态  │              │ UI 状态   │
   │ 指标     │              │ 权限     │
   │ 配置     │              │ 任务     │
   │ 认证     │              │ MCP     │
   │ 遥测     │              │ 通知     │
   └────┬────┘              └────┬────┘
        │                        │
        ▼                        ▼
   通过 getter/setter        通过 React Context
   在服务层访问               在 UI 层访问
```

---

## 附录：关键文件索引

| 文件路径 | 大小 | 说明 |
|----------|------|------|
| screens/REPL.tsx | 895KB | 主交互界面 |
| tools/BashTool/BashTool.tsx | 160KB | Shell 执行 |
| tools/BashTool/bashSecurity.ts | 102KB | Bash 安全分析 |
| tools/BashTool/bashPermissions.ts | 98KB | Bash 权限规则 |
| tools/AgentTool/AgentTool.tsx | 160KB+ | 子 Agent |
| utils/bash/bashParser.ts | 130KB | Bash AST 解析器 |
| utils/bash/ast.ts | 112KB | AST 类型定义 |
| cli/print.ts | 212KB | 输出格式化 |
| utils/permissions/permissions.ts | 52KB | 权限引擎 |
| utils/permissions/filesystem.ts | 62KB | 文件系统权限 |
| bootstrap/state.ts | 56KB | 全局状态管理 |
| services/api/claude.ts | 大 | API 客户端 |
| services/tools/StreamingToolExecutor.ts | — | 工具并发执行 |
| Tool.ts | — | 工具框架定义 |
| constants/prompts.ts | — | 系统提示词组装 |

---

## 11. 构建系统与运行测试

### 11.1 构建工具链

Claude Code 使用 **Bun** 作为构建工具和打包器（bundler），而非传统的 webpack/esbuild/rollup。

| 组件 | 技术 |
|------|------|
| 打包器 | Bun bundler |
| 运行时 | Node.js >= 18（发布版），Bun（开发/构建时） |
| 特性开关 | `bun:bundle` 的 `feature()` API |
| 构建宏 | `MACRO.VERSION`、`MACRO.BUILD_TIME` 编译时内联 |
| 死代码消除 | `feature()` 条件编译 + Bun DCE |

### 11.2 构建机制详解

#### 11.2.1 Bun Bundle API

源码中使用 `bun:bundle` 模块进行编译时特性控制：

```typescript
// constants/system.ts
import { feature } from 'bun:bundle'

// 编译时条件 — 未启用的特性整个分支被 DCE 移除
if (feature('VOICE_MODE')) {
  // 只在启用语音模式的构建中包含
}
```

已知的特性开关：

| Feature Flag | 功能 |
|-------------|------|
| `PROACTIVE` | 主动式功能 |
| `KAIROS` | Assistant 激活模式 |
| `VOICE_MODE` | 语音输入 |
| `BRIDGE_MODE` | 远程桥接 |
| `DAEMON` | 守护进程模式 |
| `NATIVE_CLIENT_ATTESTATION` | 原生客户端认证 |
| `CHICAGO_MCP` | Computer Use MCP |

#### 11.2.2 编译时宏替换

`MACRO` 对象在构建时被内联替换为实际值：

```typescript
// 源码中的写法
console.log(MACRO.VERSION)

// 构建后的 cli.js 中变成
console.log("2.1.88")
```

构建产物中可以看到实际内联的值：
```javascript
{
  ISSUES_EXPLAINER: "report the issue at https://github.com/anthropics/claude-code/issues",
  PACKAGE_URL: "@anthropic-ai/claude-code",
  README_URL: "https://code.claude.com/docs/en/overview",
  VERSION: "2.1.88",
  FEEDBACK_CHANNEL: "https://github.com/anthropics/claude-code/issues",
  BUILD_TIME: "2026-03-30T21:59:52Z"
}
```

#### 11.2.3 Bun 原生优化

构建产物运行时检测 Bun runtime 并使用原生 API：

```typescript
// ink/stringWidth.ts — 终端字符宽度计算
if (typeof Bun !== 'undefined' && Bun.stringWidth) {
  return Bun.stringWidth(str)  // 原生 C++ 实现
}
// 回退到 JS 实现

// ink/wrapAnsi.ts — ANSI 字符串换行
if (typeof Bun !== 'undefined' && Bun.wrapAnsi) {
  return Bun.wrapAnsi(str, cols)
}
```

#### 11.2.4 单文件可执行

Bun 支持将整个项目编译为单文件可执行文件：

```typescript
// utils/bundledMode.ts
function isInBundledMode() {
  return typeof Bun !== 'undefined' && Bun.embeddedFiles !== undefined
}

function isRunningWithBun() {
  return typeof process.versions.bun === 'string'
}
```

### 11.3 发布产物结构

npm 发布的包是已构建的产物，不包含源码级构建脚本：

```
@anthropic-ai/claude-code@2.1.88
├── cli.js              # 13.0MB — 单文件 bundle（Bun 打包输出）
├── cli.js.map          # 59.8MB — Source Map（含完整源码）
├── sdk-tools.d.ts      # 117KB  — SDK 类型定义
├── package.json        # 1.2KB  — 包元数据
├── bun.lock            # 0.6KB  — Bun 锁文件
├── LICENSE.md          # 0.1KB
├── README.md           # 2.0KB
└── vendor/
    ├── ripgrep/        # 各平台 rg 二进制（搜索引擎）
    │   ├── arm64-darwin/rg
    │   ├── arm64-linux/rg
    │   ├── arm64-win32/rg.exe
    │   ├── x64-darwin/rg
    │   ├── x64-linux/rg
    │   └── x64-win32/rg.exe
    └── audio-capture/  # 各平台音频捕获（语音输入）
        ├── arm64-darwin/audio-capture.node
        ├── arm64-linux/audio-capture.node
        ├── arm64-win32/audio-capture.node
        ├── x64-darwin/audio-capture.node
        ├── x64-linux/audio-capture.node
        └── x64-win32/audio-capture.node
```

### 11.4 从源码重新构建（理论）

由于 Anthropic 未开源完整的构建配置，无法直接从还原的源码重新构建。但根据分析，构建流程大致为：

```bash
# 1. 需要 Bun 运行时
curl -fsSL https://bun.sh/install | bash

# 2. 安装依赖
bun install

# 3. 构建命令（推测）
bun build src/entrypoints/cli.tsx \
  --outfile cli.js \
  --target node \
  --sourcemap=external \
  --define "MACRO.VERSION='2.1.88'" \
  --define "MACRO.BUILD_TIME='$(date -u +%Y-%m-%dT%H:%M:%SZ)'" \
  --bundle
```

关键限制：
- 缺少完整的 `tsconfig.json` 和 `bunfig.toml`
- `bun:bundle` 的 `feature()` 宏定义列表未知
- 内部依赖的私有 npm 包无法获取
- `MACRO` 替换的完整字段列表需要构建脚本

---

*报告生成时间：2026-03-31*
*源码版本：@anthropic-ai/claude-code v2.1.88*
