# OpenClaw 项目架构整体详解

> 本文档详细描述了 OpenClaw 项目的架构设计、组件关系和核心功能调用链。

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [架构分层详解](#3-架构分层详解)
4. [核心功能调用链](#4-核心功能调用链)
5. [关键设计模式](#5-关键设计模式)
6. [扩展点说明](#6-扩展点说明)
7. [数据流图](#7-数据流图)

---

## 1. 项目概述

### 1.1 项目定位

OpenClaw 是一个**多渠道 AI 网关系统**，核心功能是：
- 将各种消息平台（Telegram、Discord、WhatsApp、Slack、Signal、iMessage）连接到 AI 代理
- 提供统一的 CLI 和网关接口
- 支持插件化扩展新的渠道和功能
- 实现智能消息路由和会话管理

### 1.2 技术栈

| 层级 | 技术 |
|------|------|
| 运行时 | Node.js 22+, Bun |
| 语言 | TypeScript (ESM) |
| CLI 框架 | Commander.js |
| 通信协议 | WebSocket, HTTP/HTTPS |
| 数据库 | SQLite (with sqlite-vec) |
| 测试框架 | Vitest |
| 构建工具 | TypeScript Compiler, Rolldown |
| 代码质量 | Oxlint, Oxfmt |

### 1.3 目录结构概览

```
openclaw/
├── src/                          # 核心源代码
│   ├── cli/                      # CLI 层
│   ├── commands/                 # 命令实现
│   ├── agents/                   # AI 代理系统
│   ├── channels/                 # 渠道抽象和注册
│   ├── gateway/                  # 网关服务器
│   ├── routing/                  # 消息路由
│   ├── config/                   # 配置管理
│   ├── media/                    # 媒体处理
│   ├── media-understanding/      # AI 媒体理解
│   ├── providers/                # AI 提供者集成
│   ├── memory/                   # 记忆和搜索
│   ├── plugins/                  # 插件系统
│   └── [渠道实现]/
│       ├── telegram/
│       ├── discord/
│       ├── slack/
│       ├── signal/
│       ├── imessage/
│       └── web/ (WhatsApp)
├── extensions/                   # 外部插件
│   ├── msteams/
│   ├── matrix/
│   ├── zalo/
│   └── ...
├── apps/                         # 移动应用
│   ├── ios/
│   ├── android/
│   └── macos/
├── docs/                         # 文档
└── dist/                         # 构建输出
```

---

## 2. 整体架构

### 2.1 系统架构图

```mermaid
graph TB
    subgraph "用户交互层"
        CLI["CLI (openclaw 命令)"]
        Gateway["Gateway (WebSocket/HTTP)"]
        WebUI["Web UI / Control Panel"]
        MobileApp["Mobile App (iOS/Android)"]
    end

    subgraph "命令处理层"
        CommandRegistry["Command Registry<br/>命令注册中心"]
        Commands["Commands<br/>(agent, channels, message, etc.)"]
    end

    subgraph "业务逻辑层"
        AgentSystem["Agent System<br/>AI 代理核心"]
        ChannelManager["Channel Manager<br/>渠道管理器"]
        Router["Message Router<br/>消息路由器"]
        SessionMgr["Session Manager<br/>会话管理器"]
    end

    subgraph "渠道适配层"
        Telegram["Telegram Bot"]
        Discord["Discord Bot"]
        WhatsApp["WhatsApp Web"]
        Slack["Slack Socket Mode"]
        Signal["Signal CLI"]
        iMessage["iMessage Bridge"]
    end

    subgraph "AI 服务层"
        PiAgent["Pi Agent Core"]
        OpenAI["OpenAI API"]
        Anthropic["Claude API"]
        Google["Gemini API"]
        LocalLLM["Local LLM (Ollama)"]
    end

    subgraph "基础设施层"
        Config["Config System<br/>配置系统"]
        Storage["Storage (SQLite)<br/>存储引擎"]
        MediaPipeline["Media Pipeline<br/>媒体管道"]
        MemoryIndex["Memory Index<br/>记忆索引"]
        PluginSystem["Plugin System<br/>插件系统"]
    end

    CLI --> CommandRegistry
    Gateway --> CommandRegistry
    WebUI --> Gateway
    MobileApp --> Gateway

    CommandRegistry --> Commands
    Commands --> AgentSystem
    Commands --> ChannelManager

    AgentSystem --> Router
    Router --> SessionMgr
    SessionMgr --> PiAgent

    ChannelManager --> Telegram
    ChannelManager --> Discord
    ChannelManager --> WhatsApp
    ChannelManager --> Slack
    ChannelManager --> Signal
    ChannelManager --> iMessage

    PiAgent --> OpenAI
    PiAgent --> Anthropic
    PiAgent --> Google
    PiAgent --> LocalLLM

    AgentSystem --> Config
    SessionMgr --> Storage
    AgentSystem --> MediaPipeline
    PiAgent --> MemoryIndex
    ChannelManager --> PluginSystem

    style CLI fill:#e1f5ff
    style Gateway fill:#e1f5ff
    style AgentSystem fill:#fff4e1
    style Router fill:#ffe1f5
    style PiAgent fill:#e1ffe1
```

### 2.2 核心组件关系图

```mermaid
graph LR
    subgraph "入口点"
        Entry["openclaw.mjs"]
        MainEntry["dist/entry.js"]
        CliRun["dist/cli/run-main.js"]
    end

    subgraph "CLI 核心"
        BuildProgram["build-program.ts<br/>构建程序"]
        Deps["deps.ts<br/>依赖工厂"]
        Runtime["runtime.ts<br/>运行时环境"]
    end

    subgraph "命令系统"
        Registry["command-registry.ts<br/>命令注册表"]
        AgentCmd["agent/ 命令"]
        ChannelCmd["channels/ 命令"]
        MessageCmd["message/ 命令"]
    end

    subgraph "执行层"
        RuntimeEnv["defaultRuntime<br/>运行时包装"]
        ErrorHandler["错误处理"]
    end

    Entry --> MainEntry
    MainEntry --> CliRun
    CliRun --> BuildProgram
    BuildProgram --> Registry
    Registry --> AgentCmd
    Registry --> ChannelCmd
    Registry --> MessageCmd

    BuildProgram --> Deps
    AgentCmd --> RuntimeEnv
    ChannelCmd --> RuntimeEnv
    MessageCmd --> RuntimeEnv

    RuntimeEnv --> ErrorHandler

    style Entry fill:#e1f5ff
    style BuildProgram fill:#fff4e1
    style Registry fill:#ffe1f5
    style RuntimeEnv fill:#e1ffe1
```

---

## 3. 架构分层详解

### 3.1 CLI 层 (`src/cli/`)

#### 职责
- 解析命令行参数
- 构建命令树
- 管理命令执行生命周期
- 提供依赖注入

#### 关键文件

| 文件 | 职责 |
|------|------|
| `run-main.js` | CLI 主入口，处理环境配置 |
| `program/build-program.ts` | 构建 Commander 程序 |
| `program/command-registry.ts` | 命令注册表 |
| `deps.ts` | 依赖工厂，创建渠道发送函数 |
| `runtime.ts` | 运行时环境抽象 |
| `progress.ts` | 进度显示和用户反馈 |

#### 程序构建流程

```mermaid
sequenceDiagram
    participant User
    participant Entry as openclaw.mjs
    participant Main as entry.js
    participant CliRun as run-main.js
    participant Builder as build-program.ts
    participant Registry as command-registry.ts
    participant Cmd as Command

    User->>Entry: openclaw agent --message "hello"
    Entry->>Main: import("./dist/entry.js")
    Main->>Main: parseCliProfileArgs()
    Main->>Main: applyCliProfileEnv()
    Main->>CliRun: import("./cli/run-main.js")
    CliRun->>Builder: buildProgram()
    Builder->>Builder: createProgramContext()
    Builder->>Registry: 遍历 commandRegistry
    loop 每个命令
        Registry->>Cmd: register({program, ctx})
    end
    Builder-->>CliRun: program
    CliRun->>CliRun: program.parseAsync(argv)
    Cmd->>Cmd: action(opts)
    Cmd-->>User: 输出结果
```

### 3.2 命令层 (`src/commands/`)

#### 命令分类

| 类别 | 命令 | 说明 |
|------|------|------|
| 代理 | `agent` | 运行 AI 代理回合 |
| 渠道 | `channels` | 管理消息渠道 |
| 消息 | `message send` | 发送消息 |
| 配置 | `config` | 管理配置 |
| 状态 | `status` | 查看系统状态 |
| 引导 | `onboarding` | 首次配置向导 |
| 会话 | `sessions` | 管理会话 |

#### 依赖注入模式

```typescript
// src/cli/deps.ts
export type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  sendMessageSlack: typeof sendMessageSlack;
  sendMessageSignal: typeof sendMessageSignal;
  sendMessageIMessage: typeof sendMessageIMessage;
  // ...
};

export function createDefaultDeps(): CliDeps {
  return {
    sendMessageWhatsApp,
    sendMessageTelegram,
    sendMessageDiscord,
    sendMessageSlack,
    sendMessageSignal,
    sendMessageIMessage,
    // ...
  };
}
```

### 3.3 Agent 系统层 (`src/agents/`)

#### 核心组件

```
src/agents/
├── cli-session.ts              # CLI 会话 ID 管理
├── auth-profiles/              # 认证配置系统
│   ├── types.ts                # 认证类型定义
│   ├── store.ts                # 配置存储
│   ├── profiles.ts             # 配置文件操作
│   ├── usage.ts                # 使用统计和冷却
│   └── order.ts                # 配置文件顺序管理
├── bash-tools/                 # Shell 工具执行
│   ├── exec.ts                 # 直接执行
│   └── process.ts              # 进程管理
├── pi-embedded-runner/         # Pi Agent 集成
│   ├── run.ts                  # 运行时管理
│   ├── subscribe.ts            # 订阅机制
│   └── tools.ts                # 工具适配
├── model-auth.ts               # 模型认证处理
├── routing/                    # 消息路由
│   └── session-key.ts          # 会话密钥管理
└── sandbox/                    # 代码执行沙盒
```

#### 认证类型

```typescript
// API Key 认证
type ApiKeyCredential = {
  type: "api_key";
  provider: string;
  key: string;
  email?: string;
};

// Token 认证
type TokenCredential = {
  type: "token";
  provider: string;
  token: string;
  expires?: number;
  email?: string;
};

// OAuth 认证
type OAuthCredential = {
  type: "oauth";
  provider: string;
  clientId?: string;
  email?: string;
};
```

#### 认证配置轮询机制

```mermaid
graph TB
    Start["需要认证"] --> CheckProfiles{有多个配置文件?}
    CheckProfiles -->|否| UseSingle["使用唯一配置"]
    CheckProfiles -->|是| CheckUsage{"检查使用统计"}

    CheckUsage --> GetAvailable["获取可用配置<br/>过滤冷却中的"]
    GetAvailable --> SelectBy["选择策略"]

    SelectBy -->|轮询| RoundRobin["轮询选择<br/>循环使用"]
    SelectBy -->|失败计数| FailureCount["选择失败次数最少的"]

    RoundRobin --> UseProfile
    FailureCount --> UseProfile

    UseProfile --> Execute["执行请求"]
    Execute --> Success{成功?}

    Success -->|是| RecordSuccess["记录成功<br/>重置失败计数"]
    Success -->|否| RecordFailure["记录失败<br/>增加失败计数"]

    RecordSuccess --> End
    RecordFailure --> CheckCooldown{"达到冷却阈值?"}

    CheckCooldown -->|是| SetCooldown["设置冷却状态"]
    CheckCooldown -->|否| End
    SetCooldown --> End

    style Start fill:#e1f5ff
    style UseProfile fill:#fff4e1
    style Execute fill:#ffe1f5
    style Success fill:#e1ffe1
    style RecordFailure fill:#ffe1e1
```

### 3.4 渠道层 (`src/channels/`)

#### 渠道插件接口

```typescript
type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;                          // 渠道唯一标识
  meta: ChannelMeta;                      // 渠道元数据
  capabilities: ChannelCapabilities;      // 能力声明
  config: ChannelConfigAdapter<ResolvedAccount>;     // 配置适配器
  outbound?: ChannelOutboundAdapter;      // 出站消息适配器
  status?: ChannelStatusAdapter<ResolvedAccount>;    // 状态监控适配器
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;  // 网关适配器
  security?: ChannelSecurityAdapter<ResolvedAccount>;// 安全适配器
  actions?: ChannelMessageActionAdapter;  // 消息动作适配器
};
```

#### 渠道能力声明

```typescript
type ChannelCapabilities = {
  chatTypes: ChatType[];          // ["direct", "group", "channel", "thread"]
  reactions?: boolean;            // 支持表情反应
  threads?: boolean;              // 支持主题线程
  media?: boolean;                // 支持媒体消息
  nativeCommands?: boolean;       // 支持斜杠命令
  blockStreaming?: boolean;       // 阻塞流式输出
  polls?: boolean;                // 支持投票
};
```

#### 支持的渠道

| 渠道 | 实现 | 特点 |
|------|------|------|
| Telegram | Bot API | 简单易用，支持 Webhook |
| Discord | Bot API | 丰富的交互功能，支持 Slash 命令 |
| WhatsApp | Baileys Web | 无需官方 API，功能完整 |
| Slack | Socket Mode | 企业集成友好 |
| Signal | signal-cli | 端到端加密 |
| iMessage | AppleScript | macOS 平台原生 |

### 3.5 网关层 (`src/gateway/`)

#### 网关架构

```
src/gateway/
├── boot.ts                      # 启动引导
├── client.ts                    # WebSocket 客户端
├── server-http.ts               # HTTP 服务器
├── server-channels.ts           # 通道管理器
├── server-chat.ts               # 聊天处理器
├── server-methods/              # 方法处理器
│   ├── agent.ts                 # 代理执行
│   ├── chat.ts                  # 聊天管理
│   ├── send.ts                  # 消息发送
│   ├── channels.ts              # 通道管理
│   ├── config.ts                # 配置管理
│   └── sessions.ts              # 会话管理
└── protocol/                    # 协议定义
    ├── frames.ts                # 帧结构
    ├── index.ts                 # 协议验证
    └── schema/                  # 类型定义
```

#### 请求/响应帧结构

```typescript
// 请求帧
type RequestFrame = {
  id: string;                    // 请求 ID
  method: string;                // 方法名
  params?: unknown;              // 参数
  device?: {                     // 设备认证
    id: string;
    role: "operator" | "node";
    token?: string;
    tlsFingerprint?: string;
  };
};

// 响应帧
type ResponseFrame = {
  id: string;                    // 对应的请求 ID
  result?: unknown;              // 成功结果
  error?: {                      // 错误信息
    code: string;
    message: string;
    details?: unknown;
  };
};

// 事件帧
type EventFrame = {
  event: string;                 // 事件名
  data?: unknown;                // 事件数据
};
```

#### 权限系统

```typescript
// 角色
type Role = "operator" | "node";

// 权限范围
type Scope =
  | "operator.admin"       // 管理员权限
  | "operator.read"       // 只读权限
  | "operator.write"      // 写入权限
  | "operator.approvals"  // 审批权限
  | "operator.pairing";   // 配对权限
```

### 3.6 路由层 (`src/routing/`)

#### 路由解析优先级

```
1. Peer 绑定      → 精确匹配特定用户/群组
2. 父级 Peer 绑定 → 线程继承父级绑定
3. Guild/Team 绑定 → 服务器/团队级别
4. 账户绑定        → 特定账户
5. 通道绑定        → 默认通道
6. 默认代理        → 系统默认代理
```

#### 路由匹配流程

```mermaid
graph TD
    Start["收到消息"] --> Parse["解析消息元数据<br/>channel, accountId, peer, guildId"]

    Parse --> Check1{有 Peer 绑定?}
    Check1 -->|是| MatchPeer["匹配 Peer 绑定<br/>精确匹配"]
    Check1 -->|否| Check2{有父级 Peer 绑定?<br/>线程场景}

    MatchPeer --> ReturnAgent["返回绑定的 Agent"]

    Check2 -->|是| MatchParent["匹配父级 Peer<br/>线程继承"]
    Check2 -->|否| Check3{有 Guild/Team 绑定?}

    MatchParent --> ReturnAgent

    Check3 -->|是| MatchGuild["匹配 Guild/Team<br/>服务器级别"]
    Check3 -->|否| Check4{有账户绑定?}

    MatchGuild --> ReturnAgent

    Check4 -->|是| MatchAccount["匹配账户绑定<br/>账户级别"]
    Check4 -->|否| Check5{有通道绑定?}

    MatchAccount --> ReturnAgent

    Check5 -->|是| MatchChannel["匹配通道绑定<br/>默认通道"]
    Check5 -->|否| UseDefault["使用默认代理"]

    MatchChannel --> ReturnAgent
    UseDefault --> ReturnAgent

    ReturnAgent --> End["执行代理"]

    style Start fill:#e1f5ff
    style ReturnAgent fill:#e1ffe1
    style End fill:#ffe1f5
```

#### 会话键生成

```typescript
// 主会话键
"agent:main"                    // 默认代理
"agent:project-x"               // 特定代理

// 完整会话键（带身份链接）
"agent:main:telegram:dm:123456789"
"agent:project-x:discord:guild:987654321:channel:111222333"

// 全局会话
"global"
```

### 3.7 配置层 (`src/config/`)

#### 配置加载流程

```mermaid
sequenceDiagram
    participant App
    participant Loader as config.ts
    participant Parser as JSON5 Parser
    participant Resolver as resolveConfigIncludes
    participant Substitutor as resolveConfigEnvVars
    participant Validator as validateConfig

    App->>Loader: loadConfig()
    Loader->>Loader: 检查缓存

    alt 缓存有效
        Loader-->>App: 返回缓存配置
    else 缓存无效
        Loader->>Loader: 读取配置文件
        Loader->>Parser: JSON5.parse()
        Parser-->>Loader: 解析结果

        Loader->>Resolver: 处理 $include
        Resolver->>Resolver: 加载引用的文件
        Resolver-->>Loader: 合并结果

        Loader->>Substitutor: 替换环境变量
        Substitutor->>Substitutor: ${VAR:default} 语法
        Substitutor-->>Loader: 替换结果

        Loader->>Validator: 验证配置
        Validator->>Validator: Zod Schema 验证
        Validator-->>Loader: 验证结果

        Loader->>Loader: 应用默认值
        Loader-->>App: 最终配置
    end
```

#### 配置结构

```typescript
type OpenClawConfig = {
  $meta?: {                      // 元数据
    versionModified?: string;    // 修改版本
  };

  // 核心配置
  gateway?: {                    // 网关配置
    mode?: "local" | "remote";
    port?: number;
    bind?: "all" | "loopback";
  };
  agent?: {                      // 代理配置
    defaultAgentId?: string;
    model?: string;
  };

  // 渠道配置
  channels?: {
    telegram?: TelegramConfig;
    discord?: DiscordConfig;
    whatsapp?: WhatsAppConfig;
    // ...
  };

  // 工具配置
  tools?: {
    exec?: {                     // 命令执行
      enabled?: boolean;
      allow?: string[];
      deny?: string[];
      requireApproval?: string[];
    };
    media?: {                    // 媒体处理
      concurrency?: number;
      models?: MediaModelEntry[];
    };
  };

  // 会话配置
  sessions?: {
    reset?: {                    // 会话重置
      daily?: { time: string; timezone?: string };
      idle?: number;
    };
    store?: {                    // 会话存储
      path?: string;
      ttl?: number;
    };
  };

  // 插件配置
  plugins?: {
    entries?: PluginEntry[];
  };

  // Hooks 配置
  hooks?: {
    enabled?: boolean;
    mappings?: HookMapping[];
  };
};
```

### 3.8 媒体处理层 (`src/media/` & `src/media-understanding/`)

#### 媒体处理管道

```mermaid
graph TB
    Start["接收媒体附件"] --> Normalize["归一化附件<br/>normalizeMediaAttachments"]

    Normalize --> Cache["创建附件缓存<br/>MediaAttachmentCache"]

    Cache --> Extract["提取文件块<br/>extractFileBlocks"]

    Extract --> SplitByType["按媒体类型分组"]

    SplitByType --> ImageGroup{图像?}
    SplitByType --> AudioGroup{音频?}
    SplitByType --> VideoGroup{视频?}

    ImageGroup --> ImageProc["图像理解<br/>describeImage"]
    AudioGroup --> AudioProc["音频转录<br/>transcribeAudio"]
    VideoGroup --> VideoProc["视频描述<br/>describeVideo"]

    ImageProc --> SelectProvider["选择 AI 提供者"]
    AudioProc --> SelectProvider
    VideoProc --> SelectProvider

    SelectProvider --> OpenAI["OpenAI"]
    SelectProvider --> Anthropic["Anthropic"]
    SelectProvider --> Google["Google Gemini"]
    SelectProvider --> MiniMax["MiniMax"]
    SelectProvider --> Groq["Groq"]
    SelectProvider --> Deepgram["Deepgram"]

    OpenAI --> Process["处理请求"]
    Anthropic --> Process
    Google --> Process
    MiniMax --> Process
    Groq --> Process
    Deepgram --> Process

    Process --> Collect["收集结果"]

    Collect --> Format["格式化输出<br/>formatMediaUnderstandingBody"]

    Format --> End["返回处理结果"]

    style Start fill:#e1f5ff
    style Process fill:#fff4e1
    style Format fill:#ffe1f5
    style End fill:#e1ffe1
```

#### 媒体提供者能力

| 提供者 | 图像 | 音频 | 视频 | 默认模型 |
|--------|------|------|------|----------|
| OpenAI | ✓ | ✓ | ✗ | gpt-4o |
| Anthropic | ✓ | ✗ | ✗ | claude-opus-4-5 |
| Google | ✓ | ✓ | ✓ | gemini-2-flash |
| Groq | ✗ | ✓ | ✗ | - |
| Deepgram | ✗ | ✓ | ✗ | - |
| MiniMax | ✓ | ✗ | ✗ | MiniMax-VL-01 |

### 3.9 记忆和搜索层 (`src/memory/`)

#### 混合搜索架构

```mermaid
graph LR
    Query["搜索查询"] --> VectorSearch["向量搜索<br/>sqlite-vec"]
    Query --> KeywordSearch["关键词搜索<br/>FTS5"]

    VectorSearch --> VectorResults["向量结果<br/>semantic score"]
    KeywordSearch --> KeywordResults["关键词结果<br/>text score"]

    VectorResults --> Merge["合并结果<br/>mergeHybridResults"]
    KeywordResults --> Merge

    Merge --> Weighted["加权计算<br/>vectorWeight * vs + textWeight * ks"]

    Weighted --> Sort["排序<br/>按最终分数"]
    Sort --> Output["搜索结果"]

    style Query fill:#e1f5ff
    style VectorSearch fill:#fff4e1
    style KeywordSearch fill:#fff4e1
    style Merge fill:#ffe1f5
    style Output fill:#e1ffe1
```

#### 记忆索引管理

```typescript
class MemoryIndexManager {
  // 多来源索引
  private readonly sources: Set<MemorySource>;

  // 热缓存
  private sessionWarm = new Set<string>();

  // 文件监控
  private watcher: FSWatcher | null;

  // 向量搜索
  searchVector(params: {
    query: string;
    limit?: number;
    minScore?: number;
  });

  // 关键词搜索
  searchKeyword(params: {
    query: string;
    limit?: number;
  });

  // 混合搜索
  searchHybrid(params: {
    query: string;
    limit?: number;
    vectorWeight?: number;
    textWeight?: number;
  });
}
```

---

## 4. 核心功能调用链

### 4.1 CLI 命令执行流程

以 `openclaw agent --message "hello"` 为例：

```mermaid
sequenceDiagram
    participant User
    participant Entry as openclaw.mjs
    participant Main as entry.js
    participant CliRun as run-main.js
    participant Builder as build-program.ts
    participant AgentCmd as agent.ts
    participant Deps as deps.ts
    participant PiAgent as Pi Agent
    participant AI as AI Provider

    User->>Entry: openclaw agent --message "hello"
    Entry->>Main: import("./dist/entry.js")

    Note over Main: 解析 CLI profile<br/>应用环境变量

    Main->>CliRun: import("./cli/run-main.js")
    CliRun->>Builder: buildProgram()

    Note over Builder: 创建 ProgramContext<br/>注册所有命令

    Builder->>AgentCmd: 注册 agent 命令
    AgentCmd-->>Builder: 命令已注册

    Builder-->>CliRun: program
    CliRun->>AgentCmd: program.parseAsync(argv)

    Note over AgentCmd: Commander 解析参数<br/>调用 action()

    AgentCmd->>Deps: createDefaultDeps()
    Deps-->>AgentCmd: deps 对象

    AgentCmd->>AgentCmd: runCommandWithRuntime()

    Note over AgentCmd: 准备代理参数<br/>加载配置<br/>解析会话

    AgentCmd->>PiAgent: runEmbeddedPiAgent()

    Note over PiAgent: 轮询认证配置<br/>准备会话管理器<br/>订阅消息流

    PiAgent->>AI: 发送请求
    AI-->>PiAgent: 流式响应

    Note over PiAgent: 处理工具调用<br/>格式化输出

    PiAgent-->>AgentCmd: 结果
    AgentCmd-->>User: 输出结果
```

### 4.2 消息接收和处理流程

以 Telegram 消息为例：

```mermaid
sequenceDiagram
    participant TG as Telegram
    participant Monitor as telegram/monitor
    participant Handler as message handler
    participant Router as routing/resolve-route
    participant Session as session manager
    participant Agent as Pi Agent
    participant AI as AI Provider
    participant Outbound as outbound/send

    TG->>Monitor: 接收消息 (Webhook/轮询)
    Monitor->>Monitor: 归一化消息

    Note over Monitor: 提取元数据<br/>channel, accountId, peer

    Monitor->>Handler: 处理消息
    Handler->>Router: resolveAgentRoute()

    Note over Router: 按优先级匹配<br/>Peer → Guild → Account<br/>→ Channel → Default

    Router-->>Handler: agentId + source
    Handler->>Session: 获取/创建会话

    Note over Session: 加载会话历史<br/>构建上下文

    Session-->>Handler: session context
    Handler->>Agent: 执行代理

    Note over Agent: 准备消息<br/>添加媒体理解<br/>准备工具

    Agent->>AI: 发送请求

    loop 流式响应
        AI-->>Agent: 内容块
        Agent->>Outbound: 发送部分响应
    end

    Note over Agent: 处理工具调用<br/>执行命令

    Agent-->>Handler: 最终响应
    Handler->>Outbound: 发送消息

    Note over Outbound: 应用渠道适配器<br/>分块、格式化

    Outbound->>TG: 发送到 Telegram
```

### 4.3 媒体处理流程

以用户发送图片为例：

```mermaid
sequenceDiagram
    participant Channel as Telegram
    participant Handler as message handler
    participant Media as media-understanding
    participant Cache as MediaAttachmentCache
    participant Provider as AI Provider
    participant Agent as Pi Agent

    Channel->>Handler: 接收图片消息
    Handler->>Handler: normalizeMediaAttachments()

    Note over Handler: 提取 URL、本地路径<br/>检测 MIME 类型

    Handler->>Media: applyMediaUnderstanding()

    Media->>Cache: createMediaAttachmentCache()

    alt 本地文件
        Cache->>Cache: 读取文件
    else 远程 URL
        Cache->>Cache: fetchRemoteMedia()
    end

    Cache-->>Media: 文件缓冲区

    Media->>Media: extractFileBlocks()

    Note over Media: 按能力分组<br/>image/audio/video

    Media->>Provider: describeImage(buffer)

    Note over Provider: AI 分析图像<br/>生成描述

    Provider-->>Media: 描述文本

    Media->>Media: formatMediaUnderstandingBody()

    Note over Media: 构建格式化输出<br/>[Image]<br/>Description: ...

    Media-->>Handler: 处理结果

    Handler->>Agent: 发送消息（包含图像描述）

    Note over Agent: 使用描述理解上下文<br/>生成响应

    Agent-->>Channel: 发送响应
```

### 4.4 网关 WebSocket 通信流程

```mermaid
sequenceDiagram
    participant Client as Gateway Client
    participant Server as Gateway HTTP Server
    participant Methods as server-methods
    participant Agent as Pi Agent
    participant Channel as Channel Handler

    Client->>Server: WebSocket 连接

    Note over Client: 发送认证帧

    Client->>Server: {id: "1", method: "connect",<br/>device: {id, role, token}}

    Server->>Server: 验证设备

    alt 认证成功
        Server-->>Client: {id: "1", result: {connected: true}}
    else 认证失败
        Server-->>Client: {id: "1", error: {code: "UNAUTHORIZED"}}
    end

    Note over Client: 发送代理请求

    Client->>Server: {id: "2", method: "agent",<br/>params: {message, agentId, ...}}

    Server->>Methods: agentHandlers.agent()

    Note over Methods: 验证权限<br/>解析路由<br/>准备会话

    Methods->>Agent: runEmbeddedPiAgent()

    loop AI 处理
        Agent-->>Methods: 内容块
        Methods->>Server: {event: "agent.delta", data: {...}}
        Server-->>Client: 推送增量
    end

    Agent-->>Methods: 最终结果
    Methods-->>Server: {id: "2", result: {...}}
    Server-->>Client: 返回结果

    Note over Client: 发送消息请求

    Client->>Server: {id: "3", method: "send",<br/>params: {channel, to, text}}

    Server->>Channel: deliverOutbound()

    Note over Channel: 加载渠道适配器<br/>格式化消息

    Channel->>Channel: 发送到外部平台

    Channel-->>Server: 发送结果
    Server-->>Client: {id: "3", result: {sent: true}}
```

### 4.5 配置加载和热重载流程

```mermaid
sequenceDiagram
    participant App
    participant Config as config.ts
    participant Cache as ConfigCache
    participant FS as 文件系统
    participant Validator as validator

    App->>Config: loadConfig()

    Config->>Cache: 检查缓存

    alt 缓存有效且文件未修改
        Cache-->>Config: 返回缓存配置
        Config-->>App: 配置对象
    else 缓存无效或文件已修改
        Config->>FS: 读取 openclaw.json

        Note over FS: 使用文件锁<br/>确保并发安全

        FS-->>Config: 文件内容

        Config->>Config: JSON5.parse()

        Config->>Config: resolveConfigIncludes()

        Note over Config: 递归加载 $include<br/>引用的文件

        Config->>Config: resolveConfigEnvVars()

        Note over Config: 替换 ${VAR:default}<br/>环境变量

        Config->>Config: applyConfigDefaults()

        Config->>Validator: validateConfigWithPlugins()

        Note over Validator: Zod Schema 验证<br/>插件配置验证

        Validator-->>Config: 验证结果

        Config->>Cache: 更新缓存
        Config-->>App: 配置对象
    end

    Note over App: 使用配置

    Note over Config: 启动文件监控<br/>(chokidar)

    FS->>Config: 文件变更事件

    Config->>Config: 防抖延迟

    Config->>Config: 重新加载配置

    Note over Config: 比较新旧配置<br/>决定重载策略

    alt hot 模式
        Config->>Config: 热重载<br/>更新内存配置
    else hybrid 模式
        Config->>Config: 选择性重载<br/>部分重启
    else restart 模式
        Config->>App: 触发重启
    end
```

---

## 5. 关键设计模式

### 5.1 依赖注入模式

**位置**: `src/cli/deps.ts`

**目的**: 解耦命令实现和具体渠道发送函数

```typescript
// 定义依赖接口
type CliDeps = {
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  // ...
};

// 工厂函数创建依赖
function createDefaultDeps(): CliDeps {
  return {
    sendMessageTelegram,
    sendMessageDiscord,
    // ...
  };
}

// 命令接收依赖
async function agentCommand(opts: AgentOpts, runtime: RuntimeEnv, deps: CliDeps) {
  // 使用 deps.sendMessageTelegram 等
}
```

**优点**:
- 易于测试（可注入 mock）
- 支持多种实现
- 降低耦合度

### 5.2 插件模式

**位置**: `src/channels/plugins/`, `src/plugins/`

**目的**: 动态扩展渠道和功能

```typescript
// 插件接口
type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  config: ChannelConfigAdapter<ResolvedAccount>;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount>;
  // ...
};

// 插件注册表
const registry: ChannelPlugin[] = [
  telegramPlugin,
  discordPlugin,
  whatsappPlugin,
  // ...
];

// 动态加载
function loadChannelPlugin(id: ChannelId): ChannelPlugin {
  return registry.find(p => p.id === id);
}
```

**优点**:
- 开放封闭原则
- 易于添加新渠道
- 运行时扩展

### 5.3 策略模式

**位置**: `src/routing/resolve-route.ts`, `src/agents/auth-profiles/`

**目的**: 根据不同情况选择不同策略

```typescript
// 路由策略
type RouteBindingMatch = {
  peer?: RoutePeer;
  guildId?: string;
  accountId?: string;
  channelId?: string;
};

function resolveAgentRoute(params: ResolveInput): ResolvedRoute {
  // 按优先级匹配不同的策略
  if (peerMatch) return peerMatch.agentId;
  if (guildMatch) return guildMatch.agentId;
  if (accountMatch) return accountMatch.agentId;
  // ...
}

// 认证配置选择策略
function selectAuthProfile(
  profiles: AuthProfile[],
  strategy: "round-robin" | "failure-count"
): AuthProfile {
  if (strategy === "round-robin") {
    return rotateProfiles(profiles);
  } else {
    return selectByFailureCount(profiles);
  }
}
```

### 5.4 工厂模式

**位置**: `src/cli/deps.ts`, `src/channels/plugins/outbound/load.ts`

**目的**: 创建复杂的对象

```typescript
// 依赖工厂
function createDefaultDeps(): CliDeps {
  return {
    sendMessageWhatsApp,
    sendMessageTelegram,
    sendMessageDiscord,
    // ...
  };
}

// 渠道处理器工厂
async function createChannelHandler(params: {
  cfg: OpenClawConfig;
  channel: ChannelId;
  to: string;
}): Promise<ChannelHandler> {
  const outbound = await loadChannelOutboundAdapter(params.channel);
  return new ChannelHandler(outbound);
}
```

### 5.5 观察者模式

**位置**: `src/gateway/server-chat.ts`, `src/telegram/monitor/`

**目的**: 事件驱动的消息处理

```typescript
// 事件监听器
type MessageListener = (message: NormalizedMessage) => void;

class EventEmitter {
  private listeners: MessageListener[] = [];

  on(listener: MessageListener) {
    this.listeners.push(listener);
  }

  emit(message: NormalizedMessage) {
    this.listeners.forEach(l => l(message));
  }
}

// 在 Telegram 监控中使用
const emitter = new EventEmitter();
emitter.on(async (msg) => {
  await handleMessage(msg);
});
```

### 5.6 单例模式

**位置**: `src/config/`, `src/memory/manager.ts`

**目的**: 确保全局唯一实例

```typescript
// 配置缓存单例
let configCache: OpenClawConfig | null = null;
let cacheTimestamp = 0;

function loadConfig(): OpenClawConfig {
  const now = Date.now();
  if (configCache && (now - cacheTimestamp) < CACHE_TTL) {
    return configCache;
  }

  configCache = loadConfigImpl();
  cacheTimestamp = now;
  return configCache;
}

// 记忆索引管理器单例
let memoryIndexManager: MemoryIndexManager | null = null;

function getMemoryIndexManager(): MemoryIndexManager {
  if (!memoryIndexManager) {
    memoryIndexManager = new MemoryIndexManager();
  }
  return memoryIndexManager;
}
```

### 5.7 装饰器模式

**位置**: `src/cli/runtime.ts`

**目的**: 动态添加功能

```typescript
// 运行时装饰器
function runCommandWithRuntime(
  runtime: RuntimeEnv,
  action: () => Promise<void>
): Promise<void> {
  try {
    // 清除进度行
    clearActiveProgressLine();
    // 执行实际操作
    await action();
  } catch (err) {
    // 错误处理装饰
    runtime.error(String(err));
    runtime.exit(1);
  }
}
```

### 5.8 责任链模式

**位置**: `src/routing/resolve-route.ts`

**目的**: 路由匹配链

```typescript
function resolveAgentRoute(params: ResolveInput): ResolvedRoute {
  const bindings = params.cfg.bindings ?? [];

  // 链式匹配
  for (const binding of bindings) {
    if (matchesPeer(binding.match, params.peer)) {
      return {
        agentId: binding.agentId,
        source: "binding.peer"
      };
    }
  }

  // 继续链...
  if (matchesGuild(...)) { /* ... */ }
  if (matchesAccount(...)) { /* ... */ }

  // 默认处理器
  return defaultRoute;
}
```

---

## 6. 扩展点说明

### 6.1 添加新渠道

#### 步骤

1. **创建插件目录**
   ```
   extensions/mychannel/
   ├── package.json
   ├── src/
   │   └── channel.ts
   └── tsconfig.json
   ```

2. **实现渠道插件**
   ```typescript
   // src/channel.ts
   import type { ChannelPlugin } from "openclaw/plugin-sdk";

   export const mychannelPlugin: ChannelPlugin<ResolvedMyChannelAccount> = {
     id: "mychannel",
     meta: {
       label: "My Channel",
       selectionLabel: "My Channel (Custom)",
       detailLabel: "My Channel Bot",
       docsPath: "/channels/mychannel",
       blurb: "自定义渠道集成",
       systemImage: "cloud",
     },
     capabilities: {
       chatTypes: ["direct", "group"],
       reactions: true,
       media: true,
     },
     config: {
       listAccountIds: (cfg) => Object.keys(cfg.channels?.mychannel?.accounts ?? {}),
       resolveAccount: (cfg, accountId) => {
         const account = cfg.channels?.mychannel?.accounts?.[accountId];
         return {
           accountId,
           token: account?.token,
           // ...
         };
       },
       isConfigured: (account) => Boolean(account?.token),
     },
     outbound: {
       deliveryMode: "direct",
       sendText: async ({ to, text, accountId }) => {
         // 实现发送逻辑
       },
       sendMedia: async ({ to, media, accountId }) => {
         // 实现媒体发送
       },
     },
     status: {
       probeAccount: async ({ account, timeoutMs }) => {
         // 实现探测逻辑
       },
     },
     gateway: {
       startMonitor: async ({ cfg, accountId, messageHandler }) => {
         // 启动监控服务
       },
       stopMonitor: async ({ accountId }) => {
         // 停止监控服务
       },
     },
   };
   ```

3. **注册插件**
   ```typescript
   // 在 src/plugins/registry.ts 或插件 package.json 中
   export default {
     id: "mychannel",
     type: "channel",
     register: (registry) => {
       registry.registerChannel(mychannelPlugin);
     }
   };
   ```

### 6.2 添加新命令

#### 步骤

1. **创建命令文件**
   ```typescript
   // src/commands/mycommand/index.ts
   import { Command } from "commander";

   export function registerMyCommand(program: Command) {
     program
       .command("mycommand")
       .description("My custom command")
       .option("-v, --verbose", "Verbose output")
       .action(async (opts) => {
         console.log("Running my command", opts);
       });
   }
   ```

2. **注册到命令表**
   ```typescript
   // src/cli/program/command-registry.ts
   export const commandRegistry: CommandRegistration[] = [
     // ...
     {
       id: "mycommand",
       register: ({ program }) => registerMyCommand(program),
     },
   ];
   ```

### 6.3 添加 AI 提供者

#### 步骤

1. **实现提供者接口**
   ```typescript
   // src/providers/myprovider/index.ts
   import type { AIProvider } from "../types";

   export const myProvider: AIProvider = {
     id: "myprovider",
     name: "My Provider",

     // 流式聊天
     chatStream: async function* (params) {
       const response = await fetch("https://api.myprovider.com/v1/chat", {
         method: "POST",
         headers: {
           "Authorization": `Bearer ${params.apiKey}`,
           "Content-Type": "application/json",
         },
         body: JSON.stringify({
           model: params.model,
           messages: params.messages,
         }),
       });

       yield* parseStream(response);
     },

     // 嵌入生成
     embed: async (params) => {
       // 实现嵌入逻辑
     },
   };
   ```

2. **注册提供者**
   ```typescript
   // src/providers/registry.ts
   export const PROVIDERS = [
     openaiProvider,
     anthropicProvider,
     googleProvider,
     myProvider,  // 添加新的提供者
   ];
   ```

### 6.4 添加媒体理解提供者

#### 步骤

1. **实现媒体提供者接口**
   ```typescript
   // src/media-understanding/providers/myprovider.ts
   import type { MediaUnderstandingProvider } from "./types";

   export const myMediaProvider: MediaUnderstandingProvider = {
     id: "myprovider",
     capabilities: ["image", "audio", "video"],

     describeImage: async (req) => {
       const response = await fetch("https://api.myprovider.com/v1/vision", {
         method: "POST",
         body: JSON.stringify({
           image: req.buffer,
           prompt: "Describe this image in detail",
         }),
       });

       const data = await response.json();
       return {
         text: data.description,
         model: req.model,
         provider: "myprovider",
       };
     },

     transcribeAudio: async (req) => {
       // 实现音频转录
     },

     describeVideo: async (req) => {
       // 实现视频描述
     },
   };
   ```

2. **注册到注册表**
   ```typescript
   // src/media-understanding/providers/index.ts
   export function buildMediaUnderstandingRegistry(overrides?: Record<string, MediaUnderstandingProvider>) {
     const registry = new Map();

     // 添加内置提供者
     for (const provider of [...BUILT_IN_PROVIDERS]) {
       registry.set(provider.id, provider);
     }

     // 应用自定义覆盖
     if (overrides) {
       for (const [id, provider] of Object.entries(overrides)) {
         registry.set(id, provider);
       }
     }

     return registry;
   }
   ```

### 6.5 添加 Hook

#### 步骤

1. **配置 Hook**
   ```json
   // ~/.openclaw/openclaw.json
   {
     "hooks": {
       "enabled": true,
       "mappings": [
         {
           "match": {
             "path": "/webhook/github"
           },
           "action": "wake",
           "channel": "discord",
           "to": "123456789",
           "agentId": "main"
         }
       ]
     }
   }
   ```

2. **实现 Hook 处理器**
   ```typescript
   // src/hooks/handler.ts
   export async function handleWebhook(params: WebhookParams) {
     const hook = findMatchingHook(params);

     if (hook?.action === "wake") {
       await sendMessage({
         channel: hook.channel,
         to: hook.to,
         text: params.body,
       });
     }
   }
   ```

---

## 7. 数据流图

### 7.1 端到端消息流

```mermaid
flowchart TD
    Start([用户发送消息]) --> Channel[渠道接收<br/>Telegram/Discord/...]

    Channel --> Normalize[消息归一化<br/>提取元数据]
    Normalize --> Route[路由解析<br/>resolveAgentRoute]

    Route --> Session{会话存在?}
    Session -->|否| CreateSession[创建会话]
    Session -->|是| LoadSession[加载会话]

    CreateSession --> MediaCheck
    LoadSession --> MediaCheck

    MediaCheck{有媒体?}
    MediaCheck -->|是| MediaProcess[媒体处理<br/>AI 分析]
    MediaCheck -->|否| PrepareAgent
    MediaProcess --> PrepareAgent

    PrepareAgent[准备代理上下文<br/>添加工具/历史]
    PrepareAgent --> AuthSelect[选择认证配置<br/>轮询/失败计数]

    AuthSelect --> AIRequest[发送 AI 请求<br/>Pi Agent]

    AIRequest --> StreamLoop{流式响应?}

    StreamLoop -->|是| Delta[处理内容块]
    Delta --> SendDelta[发送增量<br/>可选]
    SendDelta --> StreamLoop

    StreamLoop -->|否| Complete[接收完整响应]

    Complete --> ToolCalls{有工具调用?}

    ToolCalls -->|是| ExecTools[执行工具<br/>bash-tools 等]
    ExecTools --> AIRequest

    ToolCalls -->|否| FormatResponse[格式化响应]

    FormatResponse --> Outbound[出站发送<br/>Channel Handler]

    Outbound --> End([用户接收响应])

    style Start fill:#e1f5ff
    style End fill:#e1ffe1
    style AIRequest fill:#fff4e1
    style ToolCalls fill:#ffe1f5
```

### 7.2 配置流

```mermaid
flowchart LR
    subgraph Input[配置来源]
        File[openclaw.json]
        Env[环境变量]
        Includes[$include 文件]
    end

    subgraph Process[处理流程]
        Parse[JSON5 解析]
        Resolve[解析 Includes]
        Substitute[环境变量替换]
        Validate[Zod 验证]
        Defaults[应用默认值]
    end

    subgraph Output[配置对象]
        GatewayCfg[gateway 配置]
        ChannelsCfg[channels 配置]
        AgentCfg[agent 配置]
        ToolsCfg[tools 配置]
        SessionsCfg[sessions 配置]
    end

    File --> Parse
    Env --> Substitute
    Includes --> Resolve

    Parse --> Resolve
    Resolve --> Substitute
    Substitute --> Validate
    Validate --> Defaults

    Defaults --> GatewayCfg
    Defaults --> ChannelsCfg
    Defaults --> AgentCfg
    Defaults --> ToolsCfg
    Defaults --> SessionsCfg

    style Input fill:#e1f5ff
    style Process fill:#fff4e1
    style Output fill:#e1ffe1
```

### 7.3 插件系统流

```mermaid
flowchart TD
    Start([启动时]) --> LoadCore[加载核心渠道<br/>内置插件]

    LoadCore --> ScanExtensions[扫描 extensions/ 目录]

    ScanExtensions --> LoadExt{发现扩展?}

    LoadExt -->|是| LoadExtPlugins[加载扩展插件<br/>jiti 动态导入]
    LoadExt -->|否| BuildRegistry

    LoadExtPlugins --> Validate[验证插件配置<br/>Schema 检查]
    Validate --> BuildRegistry

    BuildRegistry[构建插件注册表<br/>ChannelPlugin[]]

    BuildRegistry --> Register[注册到系统]

    Register --> Runtime{运行时请求}

    Runtime --> GetPlugin[获取插件<br/>loadChannelPlugin]
    GetPlugin --> CheckCache{缓存?}

    CheckCache -->|是| ReturnPlugin[返回插件]
    CheckCache -->|否| LoadAdapter[加载适配器]
    LoadAdapter --> Cache[更新缓存]
    Cache --> ReturnPlugin

    ReturnPlugin --> Use[使用插件功能<br/>config/outbound/gateway]

    Use --> End([完成])

    style Start fill:#e1f5ff
    style BuildRegistry fill:#fff4e1
    style Use fill:#ffe1f5
    style End fill:#e1ffe1
```

---

## 附录

### A. 术语表

| 术语 | 说明 |
|------|------|
| Agent | AI 代理，处理用户消息并生成响应 |
| Channel | 消息渠道，如 Telegram、Discord 等 |
| Peer | 消息发送者/接收者，可以是用户、群组、频道 |
| Guild | 服务器/团队（Discord/Slack 术语） |
| Account | 渠道账户配置 |
| Binding | 代理绑定规则 |
| Session | 会话，维护对话上下文 |
| Plugin | 插件，扩展系统功能 |
| Hook | 钩子，事件触发机制 |
| Outbound | 出站消息发送 |
| Monitor | 监控服务，接收消息 |
| Provider | AI 服务提供者 |
| Pi Agent | 内嵌的 AI 代理核心 |

### B. 配置文件路径

| 平台 | 配置路径 |
|------|----------|
| macOS/Linux | `~/.openclaw/openclaw.json` |
| Windows | `%USERPROFILE%\.openclaw\openclaw.json` |
| 会话存储 | `~/.openclaw/sessions/store.json` |
| 转录日志 | `~/.openclaw/sessions/transcripts/` |
| 凭据存储 | `~/.openclaw/credentials/` |
| 日志 | `~/.openclaw/logs/` |

### C. 环境变量

| 变量 | 说明 |
|------|------|
| `OPENCLAW_CONFIG_PATH` | 自定义配置文件路径 |
| `OPENCLAW_DATA_DIR` | 自定义数据目录 |
| `OPENCLAW_PROFILE` | CLI profile 名称 |
| `OPENCLAW_SKIP_CHANNELS` | 跳过渠道启动 |
| `CLAWDBOT_*` | 遗留环境变量支持 |

### D. 相关文档链接

- [OpenClaw 文档首页](https://docs.openclaw.ai)
- [配置参考](https://docs.openclaw.ai/configuration)
- [渠道文档](https://docs.openclaw.ai/channels)
- [Agent 文档](https://docs.openclaw.ai/agents)
- [API 参考](https://docs.openclaw.ai/reference/api)

---

**文档版本**: 1.0
**最后更新**: 2025-02-01
**适用于**: OpenClaw main 分支
