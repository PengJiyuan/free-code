# Genius-CLI (Free-Code) 项目分析报告

> 生成日期: 2026-04-08

---

## 1. 项目概述

**genius-cli** (对外名称 "free-code") 是 Anthropic **Claude Code** CLI 的重构修改版 - 一个终端原生 AI 编程代理。原始源码于 2026 年 3 月 31 日通过 npm 分发包中的 source map 暴露后公开。

此分支对上游快照进行了三类修改：

1. **移除遥测** - 所有出站遥测端点（OpenTelemetry/gRPC、GrowthBook 分析、Sentry 错误报告）已删除或存根化
2. **移除安全提示护栏** - Anthropic 注入的硬编码拒绝模式和"网络风险"指令块已被剥离
3. **解锁实验性功能** - 88 个功能标志中有 54 个默认启用

项目提供终端式 AI 编程助手，支持：

- 执行 bash 命令
- 读取、编辑、写入文件
- 使用 glob/grep 搜索代码库
- 与 MCP (Model Context Protocol) 服务器交互
- 运行多代理工作流
- 执行网络搜索和获取网络内容

---

## 2. 技术栈

| 类别           | 技术                                                                          |
| -------------- | ----------------------------------------------------------------------------- |
| **运行时**     | Bun (>= 1.3.11)                                                               |
| **语言**       | TypeScript (ESNext 目标)                                                      |
| **终端 UI**    | React 19 + Ink 6 (终端渲染)                                                   |
| **CLI 解析**   | Commander.js (@commander-js/extra-typings)                                    |
| **模式验证**   | Zod v4                                                                        |
| **代码搜索**   | ripgrep (打包)                                                                |
| **协议**       | MCP (Model Context Protocol), LSP (Language Server Protocol)                  |
| **API 提供商** | Anthropic SDK, OpenAI Codex, AWS Bedrock, Google Vertex AI, Anthropic Foundry |
| **功能标志**   | GrowthBook (仅本地评估)                                                       |
| **可观测性**   | OpenTelemetry (外部构建中为存根)                                              |

---

## 3. 架构设计

代码库采用分层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI 入口点                               │
│                  (src/entrypoints/cli.tsx)                   │
├─────────────────────────────────────────────────────────────┤
│                     主应用程序                               │
│                      (src/main.tsx)                          │
│  - Commander.js 参数解析                                     │
│  - 配置加载                                                  │
│  - 认证初始化                                                │
├─────────────────────────────────────────────────────────────┤
│                      REPL UI 层                              │
│                   (src/screens/REPL.tsx)                     │
│  - React/Ink 终端渲染                                        │
│  - 用户输入处理                                              │
│  - 消息显示                                                  │
├─────────────────────────────────────────────────────────────┤
│                    查询引擎                                  │
│                  (src/QueryEngine.ts)                        │
│  - LLM 对话管理                                              │
│  - 逐轮执行                                                  │
│  - 消息累积                                                  │
├─────────────────────────────────────────────────────────────┤
│                       工具层                                 │
│                    (src/tools/)                              │
│  - BashTool, FileReadTool, FileEditTool                      │
│  - GlobTool, GrepTool, WebSearchTool                         │
│  - AgentTool (多代理), SkillTool                             │
├─────────────────────────────────────────────────────────────┤
│                      服务层                                  │
│                   (src/services/)                            │
│  - API 客户端 (Anthropic, OpenAI, Bedrock, Vertex)          │
│  - MCP 客户端/服务器                                         │
│  - OAuth 流程                                                │
│  - 分析 (存根化)                                             │
├─────────────────────────────────────────────────────────────┤
│                     状态管理                                 │
│                   (src/state/)                               │
│  - AppState (React context 存储)                             │
│  - 权限上下文                                                │
│  - 会话持久化                                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 关键文件与目录

### 入口点

| 文件                      | 描述                                                                                                  |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| `src/entrypoints/cli.tsx` | 启动入口；处理 `--version`、`--dump-system-prompt`、守护进程模式、桥接模式的快速路径，委托给 main.tsx |
| `src/main.tsx`            | 主应用；Commander.js CLI 解析、初始化、REPL 启动                                                      |

### 核心引擎

| 文件                 | 描述                               |
| -------------------- | ---------------------------------- |
| `src/QueryEngine.ts` | LLM 查询生命周期管理；逐轮对话处理 |
| `src/query.ts`       | 底层查询执行；流式 API 处理        |

### UI 层

| 目录/文件              | 描述                                  |
| ---------------------- | ------------------------------------- |
| `src/screens/REPL.tsx` | 主交互终端 UI (React/Ink)             |
| `src/components/`      | 70+ UI 组件用于渲染消息、对话框、输入 |

### 工具系统

| 文件           | 描述                                 |
| -------------- | ------------------------------------ |
| `src/tools.ts` | 工具注册表；根据功能标志组装可用工具 |
| `src/Tool.ts`  | 工具类型定义和接口                   |
| `src/tools/`   | 各工具实现目录                       |

**工具实现目录结构：**

```
src/tools/
├── BashTool/           # Shell 命令执行
├── FileReadTool/       # 文件读取
├── FileEditTool/       # 文件编辑
├── FileWriteTool/      # 文件写入
├── GlobTool/           # 文件模式匹配搜索
├── GrepTool/           # 内容正则搜索
├── AgentTool/          # 多代理编排
├── WebSearchTool/      # 网络搜索
├── WebFetchTool/       # 网络内容获取
├── MCPTool/            # Model Context Protocol 集成
└── ...
```

### 命令系统

| 文件              | 描述                                            |
| ----------------- | ----------------------------------------------- |
| `src/commands.ts` | 斜杠命令注册表；加载 skills、plugins、workflows |
| `src/commands/`   | 各斜杠命令实现                                  |

### 服务层

| 目录                      | 描述                                          |
| ------------------------- | --------------------------------------------- |
| `src/services/api/`       | Anthropic、OpenAI、Bedrock、Vertex API 客户端 |
| `src/services/mcp/`       | MCP 客户端实现                                |
| `src/services/oauth/`     | Anthropic 和 OpenAI OAuth 流程                |
| `src/services/analytics/` | 分析服务 (外部构建中存根化)                   |

### 状态管理

| 文件                         | 描述                   |
| ---------------------------- | ---------------------- |
| `src/state/AppState.tsx`     | React context 状态存储 |
| `src/state/AppStateStore.ts` | 状态类型定义           |

### 构建系统

| 文件               | 描述                         |
| ------------------ | ---------------------------- |
| `scripts/build.ts` | Bun 构建脚本，含功能标志系统 |
| `package.json`     | 依赖和构建脚本               |

---

## 5. 入口点模式

应用程序根据模式有多种入口点：

### 1. 标准 CLI (`./cli` 或 `bun run dev`)

- 入口：`src/entrypoints/cli.tsx`
- 特殊标志快速路径 (`--version`, `--dump-system-prompt`)
- 委托给 `src/main.tsx` 完整初始化
- 通过 `launchRepl()` 启动 REPL

### 2. 桥接/远程控制模式 (`claude remote-control`)

- 需要 `BRIDGE_MODE` 功能标志
- 将本地机器作为远程桥接环境服务

### 3. 守护进程模式 (`claude daemon`)

- 长时间运行的管理进程
- 需要 `DAEMON` 功能标志

### 4. SDK/无头模式

- 直接使用 `QueryEngine` 进行非交互式使用
- 编程式 API 访问入口点

---

## 6. 核心功能

| 功能             | 描述                                                                             |
| ---------------- | -------------------------------------------------------------------------------- |
| **多模型支持**   | Anthropic Claude、OpenAI Codex、AWS Bedrock、Google Vertex AI、Anthropic Foundry |
| **斜杠命令**     | 50+ 内置命令 (`/login`、`/mcp`、`/compact`、`/cost`、`/model` 等)                |
| **代理工具**     | 30+ 工具用于文件操作、Shell 执行、网络访问、MCP 集成                             |
| **多代理工作流** | 通过 AgentTool 生成子代理；分布式工作的协调器模式                                |
| **MCP 集成**     | 连接 Model Context Protocol 服务器扩展能力                                       |
| **上下文管理**   | 自动压缩、微压缩、Token 预算管理                                                 |
| **权限系统**     | 细粒度工具权限，支持允许/拒绝规则                                                |
| **会话持久化**   | 恢复对话、会话历史、转录存储                                                     |
| **技能系统**     | 从 `.claude/skills/` 加载基于提示的技能                                          |
| **插件系统**     | 通过插件扩展                                                                     |
| **语音模式**     | 按需说话语音输入 (需要 `VOICE_MODE` 标志)                                        |
| **远程控制**     | 从移动端/Web 客户端控制 CLI (需要 `BRIDGE_MODE` 标志)                            |

---

## 7. 构建与开发

### 环境要求

- **Bun >= 1.3.11**
- **macOS 或 Linux** (Windows 通过 WSL)

### 构建命令

```bash
# 生产构建 (仅 VOICE_MODE)
bun run build          # 输出: ./cli

# 带版本标记的开发构建
bun run build:dev      # 输出: ./cli-dev

# 完整实验性功能构建 (54 个标志)
bun run build:dev:full # 输出: ./cli-dev

# 替代输出路径
bun run compile        # 输出: ./dist/cli
```

### 开发运行

```bash
# 从源码运行 (启动较慢)
bun run dev

# 自定义功能标志
bun run ./scripts/build.ts --feature=ULTRAPLAN --feature=ULTRATHINK
```

---

## 8. 配置

### 环境变量

| 变量                      | 用途                           |
| ------------------------- | ------------------------------ |
| `ANTHROPIC_API_KEY`       | Anthropic API 密钥             |
| `ANTHROPIC_MODEL`         | 覆盖默认模型                   |
| `ANTHROPIC_BASE_URL`      | 自定义 API 端点                |
| `CLAUDE_CODE_USE_OPENAI`  | 启用 OpenAI Codex 提供商       |
| `CLAUDE_CODE_USE_BEDROCK` | 启用 AWS Bedrock 提供商        |
| `CLAUDE_CODE_USE_VERTEX`  | 启用 Google Vertex AI 提供商   |
| `CLAUDE_CODE_USE_FOUNDRY` | 启用 Anthropic Foundry 提供商  |
| `AWS_REGION`              | AWS 区域 (用于 Bedrock)        |
| `CLAUDE_CODE_SIMPLE`      | 简单模式 (仅 Bash、Read、Edit) |

### 配置文件

| 文件                      | 用途           |
| ------------------------- | -------------- |
| `~/.claude/settings.json` | 用户设置       |
| `~/.claude/mcp.json`      | MCP 服务器配置 |
| `.claude/CLAUDE.md`       | 项目特定上下文 |
| `.claude/skills/`         | 自定义技能目录 |

### 构建时定义

构建脚本注入以下定义：

```javascript
process.env.USER_TYPE = "external";
process.env.CLAUDE_CODE_FORCE_FULL_LOGO = "true";
process.env.CLAUDE_CODE_VERIFY_PLAN = "false";
process.env.CCR_FORCE_BUNDLE = "true";
MACRO.VERSION = "<version>";
MACRO.BUILD_TIME = "<timestamp>";
```

---

## 9. 测试

代码库包含有限的测试基础设施：

| 组件                    | 描述                      |
| ----------------------- | ------------------------- |
| `src/tools/testing/`    | 测试权限工具              |
| 测试环境                | 使用 `NODE_ENV=test` 运行 |
| `TestingPermissionTool` | 测试环境权限工具          |

**注意：** 这是重构的源码快照，而非生产开发仓库，因此可能不包含完整的测试套件。

---

## 10. 依赖分析

### 关键运行时依赖

| 包名                             | 用途                      |
| -------------------------------- | ------------------------- |
| `@anthropic-ai/sdk`              | Anthropic API 客户端      |
| `@anthropic-ai/claude-agent-sdk` | Agent SDK 集成            |
| `@modelcontextprotocol/sdk`      | MCP 协议实现              |
| `ink`                            | 终端 UI 渲染 (基于 React) |
| `react`                          | UI 框架                   |
| `@commander-js/extra-typings`    | CLI 参数解析              |
| `zod`                            | 模式验证                  |
| `chalk`                          | 终端颜色                  |
| `execa`                          | 进程执行                  |
| `chokidar`                       | 文件监视                  |
| `axios`                          | HTTP 客户端               |
| `marked`                         | Markdown 解析             |
| `sharp`                          | 图像处理                  |
| `yaml`                           | YAML 解析                 |
| `lodash-es`                      | 工具函数                  |

### 提供商特定依赖

| 包名                        | 用途                   |
| --------------------------- | ---------------------- |
| `@anthropic-ai/bedrock-sdk` | AWS Bedrock 集成       |
| `@anthropic-ai/vertex-sdk`  | Google Vertex AI 集成  |
| `@anthropic-ai/foundry-sdk` | Anthropic Foundry 集成 |
| `@aws-sdk/client-bedrock`   | AWS SDK for Bedrock    |
| `@azure/identity`           | Azure 认证             |
| `google-auth-library`       | Google Cloud 认证      |

### 开发依赖

| 包名         | 用途              |
| ------------ | ----------------- |
| `@types/bun` | Bun 类型定义      |
| `typescript` | TypeScript 编译器 |

---

## 11. 目录结构总览

```
genius-cli/
├── src/
│   ├── entrypoints/          # 入口点
│   │   └── cli.tsx
│   ├── screens/              # 屏幕/页面
│   │   └── REPL.tsx
│   ├── components/           # UI 组件 (70+)
│   ├── tools/                # 工具实现
│   ├── commands/             # 斜杠命令
│   ├── services/             # 服务层
│   │   ├── api/              # API 客户端
│   │   ├── mcp/              # MCP 实现
│   │   ├── oauth/            # OAuth 流程
│   │   └── analytics/        # 分析服务
│   ├── state/                # 状态管理
│   ├── hooks/                # React hooks
│   ├── skills/               # 技能系统
│   ├── plugins/              # 插件系统
│   ├── bridge/               # IDE 桥接
│   ├── voice/                # 语音输入
│   ├── tasks/                # 后台任务
│   ├── main.tsx              # 主应用
│   ├── QueryEngine.ts        # 查询引擎
│   ├── query.ts              # 底层查询
│   ├── tools.ts              # 工具注册
│   ├── commands.ts           # 命令注册
│   └── Tool.ts               # 工具类型
├── scripts/
│   └── build.ts              # 构建脚本
├── package.json
├── tsconfig.json
└── README.md
```

---

## 12. 总结

genius-cli 是一个使用 TypeScript 和 Bun 构建的复杂终端 AI 编程代理，采用 React/Ink 进行终端 UI 渲染。它提供跨主要 LLM 提供商的多模型支持，广泛的代码操作工具集，以及灵活的插件/技能系统。

与上游 Claude Code 发行版相比，此分支移除了遥测并解锁了实验性功能，使其成为更开放、功能更丰富的 AI 编程助手。

---

_报告生成完毕_
